# ADR-002: Flat Schema over Nested Models

**Date:** 2026-02-09
**Status:** Accepted
**Project:** P1: Synthetic Data, Home DIY Repair
**Category:** Data Model

## Context

The `DIYRepairRecord` Pydantic model had two viable designs: a flat schema with 7 top-level fields and lists as `list[str]` matching the PRD exactly, or nested sub-models (`Tool(name, category, is_common)`, `RepairStep(order, instruction, estimated_time)`) for richer typing. Nested models give stronger type safety but produce deeper JSON schemas that the LLM has to get right in a single pass.

## Decision

Flat schema, exactly as specified in the PRD. `tools_required` is `list[str]` with per-item `min_length=2` via `Annotated`, `steps` is `list[str]` with per-item `min_length=10`. All fields use plain strings with length constraints.

```python
tools_required: list[Annotated[str, Field(min_length=2)]] = Field(
    min_length=1, description="List of tools needed for this repair",
)
steps: list[Annotated[str, Field(min_length=10)]] = Field(
    min_length=2, description="Ordered repair steps the homeowner should follow",
)
```

I gave up domain-rich typing in exchange for LLM generation reliability. `Annotated[str, Field(min_length=N)]` on list items gives per-element validation without full sub-models, which is the Pydantic v2 way of getting constraints on individual list elements.

## Alternatives Considered

**Nested models (`Tool`, `RepairStep`)**: Richer domain modeling, stronger validation per field. But deeper JSON schema means more `properties`, `required`, and `type` blocks the LLM has to produce correctly. Each nesting level compounds the error surface. And the data ultimately flows to CSV/DataFrame, so the nested structure would need flattening anyway.

**Hybrid (nest steps, flat tools)**: Partial richness, but the boundary is arbitrary. "Why nest steps but not tools?" is a question with no good answer. Inconsistency for no real gain.

## Quantified Validation

- 30/30 records generated successfully (100%) with zero structural parse errors. The LLM never produced malformed JSON.
- The flat schema achieved 0 retries across 30 records. Nested schemas would have increased the JSON structure complexity per call; even a conservative 5% retry rate would mean 3+ unnecessary API calls.
- Flat schema injects roughly 200 tokens of JSON schema into the prompt. Nested models with `Tool`, `RepairStep`, and `SafetyPrecaution` sub-models would inject 500+ tokens, a 2.5x increase in schema overhead per call.
- Flat records map directly to DataFrame rows with zero flattening logic, so `analysis.py` works with simple dicts.

## Consequences

Schema maps 1:1 to the PRD, so any reviewer can verify correctness by comparing the two. Downstream consumers (evaluator, analysis) work with simple dicts and DataFrames without flattening nested structures. The `groupby('failure_mode').count()` operations in ADR-004's failure pattern analysis would have required flattening logic with nested models.

If I later need structured tool metadata (e.g., "is this a specialty tool?" for `unrealistic_tools` detection), I'd need to parse strings or restructure. Currently the LLM judge handles that semantic analysis.

P4 (Resume Coach) adopted the same flat-first approach: `Job` and `Resume` schemas use `list[str]` for skills rather than nested `Skill(name, category, years)`, and hit 100% validation on 550 records. Instructor handles schema injection regardless of complexity (ADR-001), but flat schemas maximize the benefit since simpler schema means fewer tokens and higher success rate.
