# ADR-005: Dual Labeling Strategy for Judge Validation

**Date:** 2026-02-11
**Status:** Accepted
**Project:** P1: Synthetic Data, Home DIY Repair
**Category:** Evaluation

## Context

The LLM-as-Judge (GPT-4o) labels all 30 records across 6 binary failure modes, producing 180 evaluations total. But an LLM judge is itself an unvalidated system. If it's systematically wrong, the entire correction pipeline (ADR-004) is acting on bad signal. Manually labeling all 180 pairs is feasible but slow. The question was how many manual labels establish enough ground truth to validate the judge.

## Decision

Manually label a 10-record stratified sample (covering all 5 categories and all difficulty levels) across all 6 failure modes, producing 60 binary ground-truth comparisons. Compute per-mode agreement rates and Cohen's Kappa between manual and LLM labels. If kappa is positive on the dominant failure modes, treat the LLM judge as reliable enough to label the remaining 20 records.

The point is to use human labels to calibrate the evaluator, not to replace it. The 10 manually labeled records are a calibration set for the LLM judge, analogous to a held-out test set validating a classifier before deploying it.

```python
# evaluator.py — compute_agreement()
from sklearn.metrics import cohen_kappa_score

def compute_agreement(
    manual: list[ManualLabel], llm_csv: list[dict]
) -> dict:
    per_mode_kappa = {}
    for mode in FAILURE_MODES:
        manual_bits = [getattr(m, mode) for m in matched]
        llm_bits    = [row[mode] for row in matched_llm]
        try:
            kappa = cohen_kappa_score(manual_bits, llm_bits)
        except ValueError:
            kappa = None  # degenerate: one label class absent
        per_mode_kappa[mode] = kappa
```

Stratified sampling ensures every category and difficulty appears in the calibration set.

## Alternatives Considered

**Label all 30 records manually**: Maximum ground truth. But 3x the effort, and diminishing returns given that the first 10 already showed 81.7% agreement.

**Raw agreement only (no kappa)**: Simpler to compute. But raw agreement is inflated when one class dominates (90% "pass" means 90% agreement by always predicting pass). Kappa corrects for chance agreement, which matters for imbalanced binary labels.

**Cross-validate with a second LLM (GPT-3.5 or Claude)**: No human effort needed. But LLM-vs-LLM agreement doesn't establish ground truth; both could be wrong in the same direction. Human labels are the only external validity anchor.

## Quantified Validation

- 10 records × 6 modes = 60 binary comparisons.
- Overall raw agreement: 81.7%.
- Overall Cohen's κ = 0.201 (fair agreement by Landis & Koch scale, positive = above chance).
- Per-mode breakdown:

| Failure Mode | Agreement | κ | Interpretation |
|---|---|---|---|
| `incomplete_answer` | 80.0% | 0.545 | Moderate, dominant mode, reliable signal |
| `safety_violations` | 80.0% | 0.412 | Moderate |
| `unrealistic_tools` | 70.0% | −0.154 | Slight negative, judge stricter than human |
| `overcomplicated_solution` | 100.0% | N/A | Degenerate: all labels 0, kappa undefined |
| `missing_context` | 100.0% | N/A | Degenerate: all labels 0, kappa undefined |
| `poor_quality_tips` | 60.0% | 0.000 | Chance level, highest disagreement mode |

- The two modes that drove the correction loop (`incomplete_answer` at 50% of failures, `poor_quality_tips` at 43.3%) have κ of 0.545 and 0.000 respectively. The `unrealistic_tools` negative kappa (−0.154) reflects the strict identity priming from ADR-003 making the judge stricter than a human labeler.
- Without score anchoring ("most guides are 3-4"), scores clustered at 4-5. With anchoring, realistic 3-4 distribution.

## Consequences

The κ=0.545 on `incomplete_answer` and κ=0.412 on `safety_violations` give the correction pipeline a measurable validation claim rather than an assumption. ADR-003's strict prompt calibration (0% to 20%) is what made the judge strict enough to have labels worth validating in the first place.

`poor_quality_tips` (κ=0.000) and `unrealistic_tools` (κ=−0.154) remain weakly validated. Corrections targeting those modes have lower confidence. A production system would expand the manual label set for those two modes specifically.

The pattern (sample-based calibration of an LLM evaluator against human ground truth) carried forward to P2 (QA pair quality) and P4 (resume/job match quality). It's the same discipline as computing precision/recall on a held-out set before deploying a classifier.
