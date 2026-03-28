# ADR-003: Judge Prompt Calibration (0% → 20%)

**Date:** 2026-02-09
**Status:** Accepted
**Project:** P1: Synthetic Data, Home DIY Repair
**Category:** Evaluation

## Context

The LLM-as-Judge uses GPT-4o to evaluate GPT-4o-mini records across 6 binary failure modes. My first prompt produced a 0% failure rate: the judge passed every record. Manual inspection showed genuine issues (electrical repairs without "turn off circuit breaker" as step 1, beginner tasks requiring specialty tools). GPT-4o's RLHF training biases it toward positive assessments unless you explicitly prime for strictness.

## Decision

Rewrite the judge system prompt with explicit strictness calibration using three levers:

1. **Identity priming**: "You are a STRICT quality evaluator" + "Your job is to find deficiencies"
2. **Concrete criteria**: Replace vague definitions with checkable conditions, numeric thresholds, and named examples
3. **Distribution anchoring**: "Most guides are 3-4" and "most have at least 1-2 issues" to set expectations that finding failures is normal

The calibration moves that mattered most:

| Element | Lenient (0%) | Strict (20%) |
|---------|-------------|-------------|
| Identity | "quality evaluator" | "**STRICT** quality evaluator" |
| Expectation | *(none)* | "Most guides have 1-2 issues" |
| `safety_violations` | "Missing safety info" | Explicit checklist: power-off, PPE, professional referral, hazard callouts |
| `unrealistic_tools` | "Tools a homeowner wouldn't have" | Named examples: multimeter, torque wrench + "fail if list suspiciously short" |
| `overcomplicated` | "Too complex" | Hard limits: beginner ≤8 steps, intermediate ≤12 |
| Anti-rationalization | *(none)* | "Only give 0 when genuinely strong" |

## Alternatives Considered

**Lenient prompt (original)**: Simple, few failures. But 0% failure rate is useless; it defeats the entire purpose of having an evaluation step. No diagnostic value.

**Multi-turn judge (ask, then challenge)**: Could catch edge cases by having the judge argue with itself. But it doubles API cost, adds orchestration complexity, and offers diminishing returns on 30 records.

**Temperature tuning (lower = stricter?)**: No prompt rewrite needed. But temperature controls randomness, not strictness. Lower temperature makes the judge more deterministic, not more critical.

## Quantified Validation

- Failure rate went from 0% to 20% (0 to 36 failures across 180 evaluations).
- Failures concentrated in `incomplete_answer` (50%) and `poor_quality_tips` (43.3%), which gave me actionable signal for ADR-004's template improvements.
- Manual vs LLM agreement: 81.7% raw agreement across 10 records × 6 modes (60 binary comparisons). Range: 60% (`poor_quality_tips`) to 100% (`overcomplicated_solution`, `missing_context`). Overall Cohen's κ=0.201; per-mode: `incomplete_answer` κ=0.545, `safety_violations` κ=0.412. Some modes had degenerate all-zero labels making kappa undefined. The positive κ on the two most frequent failure modes is what matters. See ADR-005 for the dual-labeling strategy behind these numbers.
- Without "most guides are 3-4", scores clustered at 4-5. With anchoring, realistic 3-4 distribution.

## Consequences

The 20% failure rate gives the correction loop (ADR-004) meaningful work. Failure reasons in `JudgeResult.labels[].reason` are specific enough to drive targeted template improvements (v2 prompts). The failure concentration in `incomplete_answer` and `poor_quality_tips` told me exactly where to focus.

The "right" failure rate is subjective. 20% works for DIY repair, but each new domain needs its own calibrated criteria; this prompt doesn't copy-paste to other domains without rethinking the thresholds.

The three-lever calibration pattern (identity + specificity + anchoring) carried forward to P2 evaluation judges and P4's quality scoring system. Same Instructor pattern from ADR-001 (`response_model=JudgeResult`) powers the judge here. The analogy to linter configuration is useful: a lenient prompt is ESLint with rules off, a strict prompt is a tuned config catching real issues.
