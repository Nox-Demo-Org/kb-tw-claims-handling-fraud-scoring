---
type: Concept
title: Scoring Formula and Aggregation Model
description: The fraud-scoring service evaluates the fraud risk of every reported insurance claim on a continuous scale from 0.0 (lowest risk) to 1.0 (highest risk).
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/concepts/scoring-formula.md
tags:
- fraud-scoring
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

# Scoring Formula and Aggregation Model

The `fraud-scoring` service evaluates the fraud risk of every reported insurance claim on a continuous scale from `0.0` (lowest risk) to `1.0` (highest risk). The final risk score is computed by combining internal heuristic signals with third-party fraud risk data through a weighted aggregation model.

---

## The Composite Scoring Formula

The service implements a weighted linear combination defined in `app/main.py`:

$$\text{Score}_{\text{composite}} = \text{round}\Big(0.6 \times \text{Score}_{\text{internal}} + 0.4 \times \text{Score}_{\text{vendor}},\, 3\Big)$$

```python
ours = internal_score(req)
theirs = vendor_score(req.customer_id, req.peril)
total = round(0.6 * ours + 0.4 * theirs, 3)
```

The composite score is rounded to three decimal places.

---

## Component Scores

### 1. Internal Risk Score ($S_{\text{internal}}$ — 60% Weight)

The internal heuristic score evaluates attributes from the incoming [[entities/score-request|ScoreRequest]] payload. Computed in `app/signals.py` via `internal_score()`, the score starts at `0.0` and accumulates additions based on the following rules, capped at `1.0`:

| Signal / Condition | Additive Value | Evaluated Field |
| :--- | :--- | :--- |
| Early policy claim (`days_since_policy_start < 30`) | `+0.4` | `days_since_policy_start` |
| High claim frequency (`claims_last_3_years >= 2`) | `+0.3` | `claims_last_3_years` |
| High-risk peril (`peril in {"theft", "accidental_damage"}`) | `+0.1` | `peril` |

For deep-dive documentation on signal rules and capping behavior, see [[entities/fraud-signals]].

### 2. Vendor Risk Score ($S_{\text{vendor}}$ — 40% Weight)

The external score is retrieved via `app/vendor.py` through `vendor_score(customer_id, peril)`. It performs an HTTP `POST` request to `FRAUD_VENDOR_URL` sending `{"subject": customer_id, "peril": peril}` and extracts the float value from the `"risk"` response field (defaulting to `0.0` if missing).

For integration details, request payload specifications, and network behavior, see [[entities/vendor-client]] and [[decisions/vendor-resilience-and-timeouts]].

---

## Flagging Threshold & Action

The service defines a static risk threshold constant in `app/main.py`:

```python
FLAG_THRESHOLD = 0.8
```

### Evaluation Logic

When the composite score is calculated during execution of `POST /v1/scores`:

1. **Threshold Check**: The score is checked against the threshold: `total > FLAG_THRESHOLD` (strictly greater than `0.8`).
2. **Flag Event Publication**: If `total > 0.8`, `publish_flag(claim_id, total)` is invoked, publishing a `fraud.score.flagged` event containing the `claim_id`, `score`, and `reasons` payload to alert downstream counter-fraud investigators.
3. **Response Payload**: The endpoint responds to the caller with the calculated score and flagging outcome:
   ```json
   {
     "claim_id": "CLM-10023",
     "score": 0.84,
     "flagged": true
   }
   ```

For business reasoning and operational workflows surrounding this threshold, refer to [[decisions/flag-threshold-and-counter-fraud-routing]].

---

## Example Calculations

| Scenario | $S_{\text{internal}}$ Breakdown | $S_{\text{internal}}$ | $S_{\text{vendor}}$ | Formula Calculation | Composite Score | Flagged (`> 0.8`) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Low-risk claim** | Policy age: 120d (`0.0`), 0 prior claims (`0.0`), peril: `water` (`0.0`) | `0.0` | `0.10` | $0.6(0.0) + 0.4(0.10) = 0.04$ | `0.04` | `false` |
| **Moderate-risk claim** | Policy age: 15d (`+0.4`), 0 prior claims (`0.0`), peril: `fire` (`0.0`) | `0.4` | `0.60` | $0.6(0.4) + 0.4(0.60) = 0.48$ | `0.48` | `false` |
| **High internal / Moderate vendor** | Policy age: 10d (`+0.4`), 3 prior claims (`+0.3`), peril: `theft` (`+0.1`) | `0.8` | `0.70` | $0.6(0.8) + 0.4(0.70) = 0.76$ | `0.76` | `false` |
| **High risk (Flagged)** | Policy age: 5d (`+0.4`), 2 prior claims (`+0.3`), peril: `theft` (`+0.1`) | `0.8` | `0.90` | $0.6(0.8) + 0.4(0.90) = 0.84$ | `0.84` | `true` |
| **Maximum risk** | Policy age: 2d (`+0.4`), 4 prior claims (`+0.3`), peril: `accidental_damage` (`+0.1`) | `0.8` (capped at `1.0` if exceeded) | `1.0` | $0.6(0.8) + 0.4(1.0) = 0.88$ | `0.88` | `true` |

---

## Related Documentation

- [[summaries/api-spec]] — REST interface and Pub/Sub event schemas
- [[entities/score-request]] — Structure and fields of `ScoreRequest`
- [[entities/fraud-signals]] — Internal heuristic signal evaluation details
- [[entities/vendor-client]] — Integration client for the third-party fraud vendor
- [[decisions/flag-threshold-and-counter-fraud-routing]] — Selection of the `0.8` flag threshold
- [[decisions/vendor-resilience-and-timeouts]] — Fallback scoring and latency considerations during vendor outages
