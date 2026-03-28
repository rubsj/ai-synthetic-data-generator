# ADR-004: Template Improvement as Correction Strategy

**Date:** 2026-02-09
**Status:** Accepted
**Project:** P1: Synthetic Data, Home DIY Repair
**Category:** Algorithm

## Context

After generating 30 v1 records, the LLM-as-Judge (ADR-003) found a 20% failure rate (36/180 evaluations). Two modes dominated: `incomplete_answer` (50% of records) and `poor_quality_tips` (43.3%), accounting for 28 of 36 total failures. The PRD specified two correction strategies: individual record correction and template improvement. I implemented both, measured them independently, then combined them.

## Decision

**Strategy A (Correction)**: Send failing records + judge feedback to GPT-4o-mini via Instructor for targeted fixes.

**Strategy B (Template v2)**: Analyze failure distributions with `groupby('failure_mode').count()` (ADR-002's flat schema made this trivial), then add explicit quality requirements to the generation prompt. V2 templates append `--- QUALITY REQUIREMENTS ---` with mode-specific instructions: `incomplete_answer` now requires troubleshooting advice and what-if scenarios; `poor_quality_tips` explicitly bans generic advice.

Templates outperformed correction because they prevent failures upstream. Individual correction asks the same model to fix its own output, and it often repeats the same mistake class. The dominant failures were systemic (GPT-4o-mini's default "good enough" behavior), and template instructions override that default.

## Alternatives Considered

**Template improvement only**: 77.8% reduction, prevents failures upstream. But it missed edge cases and fell short of the PRD's 80% target on its own.

**Individual correction only**: Simpler, no template changes needed. But only 66.7% reduction because it treats symptoms without fixing the systemic prompt issue.

**Few-shot examples in templates**: Potentially very effective at showing the model what "good" looks like. But significantly longer prompts and harder to maintain as the domain evolves.

## Quantified Validation

| Metric | V1 Original | Strategy A (Correction) | Strategy B (V2 Templates) | Combined (B then A) |
|--------|:-----------:|:-----------------------:|:-------------------------:|:-----------------:|
| Total failures | 36 | 12 | 8 | 0 |
| Failure rate | 20.0% | 6.7% | 4.4% | 0.0% |
| Reduction vs V1 | - | 66.7% | 77.8% | 100.0% |
| `incomplete_answer` | 15 | 5 | 5 | 0 |
| `poor_quality_tips` | 13 | 6 | 2 | 0 |
| `safety_violations` | 3 | 0 | 0 | 0 |
| `unrealistic_tools` | 5 | 0 | 0 | 0 |

The combined pipeline exceeded the PRD's >80% target, dropping failures from 36 to zero. V2 templates reduced the correction scope from 21/30 records to 6/30, so the correction pass only had to handle edge cases.

## Consequences

The failure analysis, template improvement, correction cycle is reusable across LLM generation pipelines. All strategies are cached, so re-running is free. ADR-003's calibrated judge provided the failure signal that made template improvements possible; without the 0% to 20% calibration, there was nothing to act on. Instructor (ADR-001) powered both v2 re-generation and the correction loop.

V2 templates are more prescriptive, which may reduce output diversity. The combined pipeline requires two LLM calls for failing records (correction + re-evaluation), though this was only 6 of 30.

P4 (Resume Coach) adopted this upstream prevention pattern as its primary lesson: dual-channel enforcement (schema validation + prompt requirements) hit 100% validation on 550 records. The shift-left idea is the same as adding stricter types and lint rules rather than catching bugs in QA.
