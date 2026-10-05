---
type: Component
title: Fraud Signals
description: app/signals.py implements the internal heuristic scoring engine for fraud-scoring.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/entities/fraud-signals.md
tags:
- fraud-scoring
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/app/signals.py
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/tests/test_signals.py
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/app/main.py
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

<!-- anchor: app/signals.py:L1-L9 -->
<!-- anchor: tests/test_signals.py:L1-L8 -->
<!-- anchor: app/main.py:L1-L33 -->

# Fraud Signals

`app/signals.py` implements the internal heuristic scoring engine for `fraud-scoring`. It evaluates claim attributes from a [[entities/score-request|ScoreRequest]] to calculate an internal baseline fraud score ranging from `0.0` to `1.0`.

## Responsibilities

- **Tenure Evaluation**: Adds risk score if the policy is newly active (`days_since_policy_start < 30`).
- **Prior Claim Frequency**: Evaluates historical claim counts (`claims_last_3_years >= 2`) to identify repeat claimant patterns.
- **Peril Classification**: Checks whether the reported `peril` belongs to known higher-risk categories (`theft` or `accidental_damage`).
- **Score Normalization**: Caps the aggregated heuristic score at `1.0` using `min(score, 1.0)`.

## Heuristic Signals and Scoring Rules

The function `internal_score(req)` initializes a baseline score of `0.0` and applies cumulative increments based on the following rules:

| Signal / Attribute | Condition | Score Increment | Description |
| :--- | :--- | :--- | :--- |
| **Policy Age** (`days_since_policy_start`) | `< 30` | `+0.4` | Flags claims submitted within the first 30 days of policy inception. |
| **Claim History** (`claims_last_3_years`) | `>= 2` | `+0.3` | Flags frequent claims activity (2 or more claims within the past 3 years). |
| **Peril Type** (`peril`) | `in {"theft", "accidental_damage"}` | `+0.1` | Adds risk weight for theft and accidental damage perils. |

### Score Bounds
The sum of all applicable signal increments is bounded at `1.0`:
```python
score = min(score, 1.0)
```
If none of the conditions are met, `internal_score` returns `0.0`. If all conditions are met, the score sums to `0.8` (which is within the `<= 1.0` cap).

## Dependencies

- **Callers**:
  - `app/main.py`: Invoked by the `score` endpoint handler (`POST /v1/scores`, see [[summaries/api-spec]]) where the internal score is weighted at 60% in the composite formula (`0.6 * ours + 0.4 * theirs`). See [[concepts/scoring-formula]].
- **Input Types**:
  - Accepts a [[entities/score-request|ScoreRequest]] or any object with attributes:
    - `days_since_policy_start` (`int`)
    - `claims_last_3_years` (`int`)
    - `peril` (`str`)
- **Tests**:
  - Verified in `tests/test_signals.py` (e.g., `test_new_policy_raises_score`).
