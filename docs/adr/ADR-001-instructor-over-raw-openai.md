# ADR-001: Instructor over Raw OpenAI API

**Date:** 2026-02-09
**Status:** Accepted
**Project:** P1: Synthetic Data, Home DIY Repair
**Category:** Tool Choice

## Context

I needed to generate structured JSON from GPT-4o-mini conforming to a 7-field Pydantic schema (`DIYRepairRecord`). The raw OpenAI API returns strings, so getting typed output means writing manual parsing, validation, and retry logic. Since this was the first portfolio project, whatever pattern I chose for LLM-to-structured-output would carry forward through P2 to P9.

## Decision

Use **Instructor** (`instructor.from_openai(OpenAI())`) as the interface between my code and the OpenAI API. Instructor wraps the client and provides three capabilities I'd otherwise build by hand:

1. **Schema injection**: calls `model_json_schema()` on the Pydantic model and appends it to the system prompt automatically.
2. **Response parsing**: calls `model_validate_json()` on the LLM response, returning a typed Pydantic object.
3. **Auto-retry with error feedback**: catches `ValidationError`, formats it as a follow-up message, and retries up to `max_retries=3`. The LLM sees its own mistake and self-corrects.

```python
record = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=DIYRepairRecord,
    messages=messages,
    max_retries=3,
)
```

The same pattern powers the evaluator (`response_model=JudgeResult`).

## Alternatives Considered

**Raw OpenAI + `json_object`**: No dependency, full control. But it's roughly 60 lines of boilerplate per call site: schema serialization, JSON parsing, validation, and a retry loop. With two call sites (generator + evaluator), that's ~120 lines of error-prone plumbing. Not worth it for 9 projects.

**LangChain `StructuredOutputParser`**: Part of the LangChain ecosystem. Heavy dependency for P1 and weaker Pydantic integration. I introduced LangChain later in P5 where it made sense.

**OpenAI function calling**: Built into the API, but requires manual validation, has no auto-retry, and the API surface is being superseded. Missing the self-healing loop that makes Instructor useful.

## Quantified Validation

- 30/30 records generated successfully (100%), zero manual retries needed.
- ~120 lines of boilerplate avoided across two call sites (generator + evaluator), covering schema injection, JSON parsing, validation, retry loops, and error formatting.
- Retry handling is the real win: Instructor feeds `ValidationError` details back to the LLM as a follow-up message. Building that manually means managing conversation history across retries, formatting Pydantic errors into natural language, and handling retry exhaustion. That's a state machine I didn't want to own.
- Same `response_model=` pattern used in both `generator.py` and `evaluator.py` with zero code duplication.

## Consequences

Adding new Pydantic models just works: pass `response_model=NewModel` and Instructor handles schema injection, parsing, and retry. That's why generation hit 100% success with no manual intervention.

Debugging failed retries is harder. Intermediate LLM responses aren't visible by default; you need Instructor's logging hooks. And there's a dependency on Instructor's API stability, though it's actively maintained and widely adopted.

The pattern carried forward to P2 (QA pair generation), P4 (resume/job generation, 550 records at 100% success), and P5 through P9. ADR-002's flat schema decision works partly because Instructor handles schema injection regardless of nesting depth. ADR-003's GPT-4o judge and ADR-004's correction loop both use the same `response_model=` pattern.
