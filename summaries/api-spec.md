---
type: Interface Reference
title: API and Event Specification
description: The fraud-scoring service exposes synchronous REST endpoints for real-time claim scoring and communicates asynchronously via Google Cloud Pub/Sub events.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/summaries/api-spec.md
tags:
- fraud-scoring
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/app/main.py
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/README.md
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/requirements.txt
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

<!-- anchor: app/main.py:L1-L33 -->
<!-- anchor: README.md:L1-L11 -->
<!-- anchor: requirements.txt:L1-L3 -->

# API and Event Specification

The `fraud-scoring` service exposes synchronous REST endpoints for real-time claim scoring and communicates asynchronously via Google Cloud Pub/Sub events.

---

## Contract Overview

| Contract | Protocol / Type | Direction | Description | Counterpart |
| :--- | :--- | :--- | :--- | :--- |
| `POST /v1/scores` | REST (JSON) | Inbound / Provided | Computes risk score for a claim payload | `claims-management` / downstream consumers |
| [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] | Pub/Sub Event | Inbound / Subscribed | Triggers scoring upon new claim submission | [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] |
| `fraud.score.flagged` | Pub/Sub Event | Outbound / Published | Emitted when composite score exceeds `0.8` | Counter-fraud & handling squad |

---

## REST API Specification

### `POST /v1/scores`

Evaluates claim risk by computing an internal heuristic score and requesting an external vendor score, combining them via the [[concepts/scoring-formula|composite scoring formula]]. If the composite score exceeds `FLAG_THRESHOLD` (`0.8`), it triggers `publish_flag` to emit a `fraud.score.flagged` event.

* **Controller Implementation:** `app/main.py`
* **Schema Definition:** `ScoreRequest` (see [[entities/score-request]])

#### Request Body (`application/json`)

| Field | Type | Description |
| :--- | :--- | :--- |
| `claim_id` | `string` | Unique identifier of the claim |
| `policy_id` | `string` | Unique identifier of the policy |
| `customer_id` | `string` | Customer identifier evaluated by the vendor client |
| `peril` | `string` | Claim peril type (e.g., `theft`, `accidental_damage`) |
| `days_since_policy_start` | `integer` | Age of policy in days at the time of claim |
| `claims_last_3_years` | `integer` | Count of prior claims within the last 3 years |

```json
{
  "claim_id": "CLM-10023",
  "policy_id": "POL-99212",
  "customer_id": "CUST-4412",
  "peril": "theft",
  "days_since_policy_start": 14,
  "claims_last_3_years": 2
}
```

#### Response Body (`200 OK`)

| Field | Type | Description |
| :--- | :--- | :--- |
| `claim_id` | `string` | Identifier matching the request `claim_id` |
| `score` | `number` (float) | Composite fraud score rounded to 3 decimal places (range `0.0` - `1.0`) |
| `flagged` | `boolean` | `true` if `score > 0.8` (`FLAG_THRESHOLD`), else `false` |

```json
{
  "claim_id": "CLM-10023",
  "score": 0.82,
  "flagged": true
}
```

---

## Pub/Sub Events

### Inbound: [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]]

* **Direction:** Subscribed (Inbound)
* **Publisher:** [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]]
* **Workflow:** Initiates fraud scoring workflows when a new claim is reported in the system. For integration flow, see [[concepts/claim-intake-integration]].

---

### Outbound: `fraud.score.flagged`

* **Direction:** Published (Outbound)
* **Trigger:** Published when the calculated composite score exceeds `FLAG_THRESHOLD = 0.8` (see [[decisions/flag-threshold-and-counter-fraud-routing]]).
* **Method:** `publish_flag(claim_id: str, score: float)` in `app/main.py`.

#### Message Payload

| Field | Type | Description |
| :--- | :--- | :--- |
| `claim_id` | `string` | Unique identifier of the flagged claim |
| `score` | `number` (float) | The total calculated fraud score |
| `reasons` | `array` / `object` | Context and heuristic signals contributing to the score |

```json
{
  "claim_id": "CLM-10023",
  "score": 0.82,
  "reasons": []
}
```

---

## Dependencies & Infrastructure

* **Framework:** FastAPI (`fastapi==0.111.0`)
* **HTTP Client:** Requests (`requests==2.32.3`) for [[entities/vendor-client|vendor queries]]
* **Messaging:** Google Cloud Pub/Sub (`google-cloud-pubsub==2.21.5`)
* **Index Reference:** See [[index]] for high-level architecture.
