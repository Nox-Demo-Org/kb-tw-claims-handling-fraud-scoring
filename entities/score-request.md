---
type: Component
title: ScoreRequest
description: ScoreRequest is a Pydantic data model defined in app/main.py that represents the incoming payload required to evaluate fraud risk for an insurance claim.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/entities/score-request.md
tags:
- fraud-scoring
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/app/main.py
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/app/signals.py
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

<!-- anchor: app/main.py:L1-L33 -->
<!-- anchor: app/signals.py:L1-L9 -->

# ScoreRequest

`ScoreRequest` is a Pydantic data model defined in `app/main.py` that represents the incoming payload required to evaluate fraud risk for an insurance claim. It is accepted directly by the `POST /v1/scores` REST endpoint (see [[summaries/api-spec]]) and supplies the parameters required by both internal heuristic scoring and external vendor evaluation.

## Schema Definition

`ScoreRequest` inherits from `pydantic.BaseModel` and specifies the following attributes:

| Field Name | Type | Description | Usage in System |
| :--- | :--- | :--- | :--- |
| `claim_id` | `str` | Unique identifier for the claim being evaluated. | Returned in the scoring response and passed to `publish_flag` when score exceeds `FLAG_THRESHOLD` (see [[decisions/flag-threshold-and-counter-fraud-routing]]). |
| `policy_id` | `str` | Unique identifier for the associated insurance policy. | Claim context and tracking. |
| `customer_id` | `str` | Unique identifier for the policyholder/claimant. | Forwarded to `vendor_score` in [[entities/vendor-client]]. |
| `peril` | `str` | Type of loss/peril (e.g., `"theft"`, `"accidental_damage"`). | Evaluated by internal heuristics in [[entities/fraud-signals]] and forwarded to [[entities/vendor-client]]. |
| `days_since_policy_start` | `int` | Number of days elapsed between policy inception and claim creation. | Evaluated by internal heuristics in [[entities/fraud-signals]] (< 30 days adds 0.4). |
| `claims_last_3_years` | `int` | Count of prior claims filed by the policyholder within the past three years. | Evaluated by internal heuristics in [[entities/fraud-signals]] (>= 2 claims adds 0.3). |

## Code Implementation

Defined in `app/main.py`:

```python
class ScoreRequest(BaseModel):
    claim_id: str
    policy_id: str
    customer_id: str
    peril: str
    days_since_policy_start: int
    claims_last_3_years: int
```

## Data Flow & Usage

1. **Payload Ingestion**: Received by the `score` route handler in `app/main.py` via `POST /v1/scores`.
2. **Internal Scoring**: The full `ScoreRequest` instance is passed to `internal_score(req)` in `app/signals.py` to evaluate policy age, prior claims count, and peril type (see [[entities/fraud-signals]]).
3. **Vendor Query**: The `customer_id` and `peril` fields are passed to `vendor_score(req.customer_id, req.peril)` in `app/vendor.py` (see [[entities/vendor-client]]).
4. **Aggregation & Flagging**: The combined composite score is calculated using [[concepts/scoring-formula]]. If the score exceeds `0.8`, `publish_flag(req.claim_id, total)` is called.

## Responsibilities

- **Type Validation and Coercion**: Enforces structural validity and field types (`str`, `int`) for claim scoring requests submitted via HTTP POST.
- **Internal Feature Provider**: Delivers peril, tenure (`days_since_policy_start`), and claim history (`claims_last_3_years`) to internal scoring heuristics.
- **Vendor Request Parameters**: Extracts claimant and peril identifiers (`customer_id`, `peril`) for external vendor risk assessments.
- **Traceability Identification**: Supplies `claim_id` and `policy_id` across scoring calculations and notification publishing.

## Dependencies

- **FastAPI / Pydantic**: Relies on `pydantic.BaseModel` for validation in `app/main.py`.
- **[[entities/fraud-signals]] (`app/signals.py`)**: Consumes `ScoreRequest` instances in `internal_score`.
- **[[entities/vendor-client]] (`app/vendor.py`)**: Consumes `customer_id` and `peril` extracted from `ScoreRequest` in `vendor_score`.
- **[[summaries/api-spec]] (`app/main.py`)**: Defined as the request body schema for the `POST /v1/scores` endpoint.
