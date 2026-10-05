---
type: Component
title: Vendor Client
description: The vendor client (app/vendor.py) is responsible for communicating with an external third-party fraud risk API to retrieve external fraud indicators for insurance claims.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/entities/vendor-client.md
tags:
- fraud-scoring
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/app/vendor.py
- resource: https://github.com/Nox-Demo-Org/fraud-scoring/blob/HEAD/app/main.py
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

<!-- anchor: app/vendor.py:L1-L16 -->
<!-- anchor: app/main.py:L1-L33 -->

# Vendor Client

The vendor client (`app/vendor.py`) is responsible for communicating with an external third-party fraud risk API to retrieve external fraud indicators for insurance claims. The external risk score is combined with internal heuristic signals during composite score calculations in [[concepts/scoring-formula]].

## Responsibilities

- **External Risk Querying**: Dispatches synchronous HTTP `POST` requests to the external fraud vendor service.
- **Payload Translation**: Maps the internal [[entities/score-request|ScoreRequest]] fields (`customer_id` and `peril`) to the vendor's expected payload format.
- **Score Extraction**: Parses the vendor JSON response to extract the float `risk` score, defaulting to `0.0` if the field is missing.

## Configuration

The vendor integration is configured via an environment variable:

| Environment Variable | Default Value | Description |
| :--- | :--- | :--- |
| `FRAUD_VENDOR_URL` | `https://vendor.example/score` | The base URL endpoint for querying external fraud scores. Assigned to `VENDOR_URL` in `app/vendor.py`. |

## Interface and Payload Structure

The primary function exported by `app/vendor.py` is `vendor_score(customer_id: str, peril: str) -> float`.

### Outbound Request

The client makes an HTTP `POST` request to `VENDOR_URL` using Python `requests`:

```python
POST {FRAUD_VENDOR_URL}
Content-Type: application/json

{
  "subject": "<customer_id>",
  "peril": "<peril>"
}
```

- **`subject`** (`str`): The unique customer ID (`customer_id` from [[entities/score-request|ScoreRequest]]).
- **`peril`** (`str`): The peril associated with the claim (e.g., `theft`, `accidental_damage`).

### Inbound Response

The vendor returns a JSON object containing a `risk` score:

```json
{
  "risk": 0.35
}
```

The client extracts `res.json().get("risk", 0.0)` and converts it to a `float`.

## Usage in Scoring Pipeline

The vendor client is invoked within the `POST /v1/scores` handler in `app/main.py`:

```python
ours = internal_score(req)
theirs = vendor_score(req.customer_id, req.peril)
total = round(0.6 * ours + 0.4 * theirs, 3)
```

The vendor score accounts for 40% of the overall composite fraud score. For complete calculation details, see [[concepts/scoring-formula]] and [[summaries/api-spec]].

## Operational Considerations and Limitations

- **Missing Timeout**: As implemented in `app/vendor.py`, `requests.post()` is called without a `timeout` parameter. 
- **Historical Incident**: During a vendor slowdown on 2026-08-27, requests blocked synchronously waiting for vendor responses, resulting in a 47-minute stall in claim registration. Architectural considerations and mitigation plans are detailed in [[decisions/vendor-resilience-and-timeouts]].

## Dependencies

- **`requests`**: Used to execute HTTP `POST` calls against `VENDOR_URL`.
- **`os`**: Reads the `FRAUD_VENDOR_URL` environment variable at module load time.
- **Called by**: `score` endpoint handler in `app/main.py` using data from [[entities/score-request|ScoreRequest]].
