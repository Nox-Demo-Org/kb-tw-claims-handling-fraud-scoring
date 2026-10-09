---
mission: NOX-1
title: 'Route high-risk claims to counter-fraud review on internal signals alone'
role: engineering
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Engineering design: Internal signal fraud routing

## Applications changing

- **`fraud-scoring`**: Owned by the **Claims / Handling squad** (on-call: `#tw-claims`). This service evaluates incoming claim risk and controls fraud flagging and event publishing. It must change to introduce standalone internal heuristic threshold evaluation in `app/main.py` and resilient vendor timeout handling in `app/vendor.py`.
- **Applications that must not change**:
  - **`claims-intake`**: Owned by the Claims / Intake squad. Publishes the `claims.claim.reported` event. Intake submission pipelines and schemas remain untouched.
  - **`claims-management`**: Synchronously invokes `POST /v1/scores` and manages handler allocation (`claims.handler.assigned`). Its integration contract and downstream triage workflows remain compatible.
  - **Counter-fraud investigation tools**: Consumes the `fraud.score.flagged` Pub/Sub topic. The event payload schema remains untouched.

## Approach

Currently, `app/main.py` evaluates fraud risk strictly against a composite score threshold:
```python
total = round(0.6 * ours + 0.4 * theirs, 3)
if total > FLAG_THRESHOLD:  # FLAG_THRESHOLD = 0.8
    publish_flag(req.claim_id, total)
```
Because internal heuristics in `app/signals.py` are capped at a maximum raw score of `0.8` (`0.4` for tenure `< 30` days, `0.3` for `claims_last_3_years >= 2`, and `0.1` for high-risk perils `theft` or `accidental_damage`), internal signals alone contribute at most `0.6 * 0.8 = 0.48` to the composite score [[kb:fraud-scoring/decisions/flag-threshold-and-counter-fraud-routing]]. Consequently, claims with severe internal warning signs can never trigger fraud review if the external vendor score is clean (`0.0`) or unavailable.

The design decision is:
1. **Define a standalone internal flag threshold**: Add `INTERNAL_FLAG_THRESHOLD = 0.8` in `app/main.py` alongside the existing `FLAG_THRESHOLD = 0.8`.
2. **Dual-condition flagging criteria**: Flag a claim for counter-fraud investigation if *either* the composite score exceeds `FLAG_THRESHOLD` *or* the internal score meets `INTERNAL_FLAG_THRESHOLD`:
   ```python
   is_flagged = (total > FLAG_THRESHOLD) or (ours >= INTERNAL_FLAG_THRESHOLD)
   ```
   Setting `INTERNAL_FLAG_THRESHOLD = 0.8` requires all three internal warning signals (policy age under 30 days, 2+ prior claims in 3 years, and theft/accidental damage peril) to be present simultaneously to trigger standalone flagging. Claims with partial internal signals (e.g. 0.4 for early tenure alone, 0.7 for tenure and repeat claims without high-risk peril, or 0.5 for tenure and peril without prior claims) will not trigger standalone flagging and continue to require corroborating vendor data.
3. **Resilient vendor client execution**: In `app/vendor.py`, implement the pending decision in [[kb:fraud-scoring/decisions/vendor-resilience-and-timeouts]]:
   - Configure a strict `2.0` second timeout on `requests.post(VENDOR_URL, ...)`.
   - Wrap the vendor HTTP call in a `requests.RequestException` block that logs the degradation and returns a fallback score of `0.0`.
   - This ensures that if the vendor times out or errors, evaluation proceeds using `ours`, flagging high internal risk claims immediately without stalling intake pipelines.
4. **Publishing and response**: If `is_flagged` is true, invoke `publish_flag(req.claim_id, total)` to emit `fraud.score.flagged` and return `{"claim_id": req.claim_id, "score": total, "flagged": is_flagged}`.

## Contracts affected

| Contract | Owner App | Consumers | Status | Details |
| :--- | :--- | :--- | :--- | :--- |
| `POST /v1/scores` | `fraud-scoring` | `claims-management`, `claims-intake` | **Unchanged** | Request body (`ScoreRequest`) and response body (`claim_id`, `score`, `flagged`) schemas remain identical. `flagged` can now return `true` when internal score meets `0.8` even if `score <= 0.8`. |
| `fraud.score.flagged` (Pub/Sub) | `fraud-scoring` | Counter-fraud triage, downstream claims services | **Unchanged** | Event message schema `{claim_id: str, score: float, reasons: list}` remains identical [[kb:fraud-scoring/summaries/api-spec]]. Outbound event volume will increase for high internal risk claims. |
| `claims.claim.reported` (Pub/Sub) | `claims-intake` | `fraud-scoring` | **Unchanged** | Subscribed inbound event schema consumed by `fraud-scoring` remains identical [[kb:fraud-scoring/concepts/claim-intake-integration]]. |
| Vendor Scoring API (`FRAUD_VENDOR_URL`) | External Vendor | `fraud-scoring` | **Unchanged** | Outbound HTTP POST payload remains `{"subject": customer_id, "peril": peril}`. Added client-side timeout and exception handling. |

## Must not break

- **Composite flagging path**: Claims on established policies that receive high external vendor scores exceeding `FLAG_THRESHOLD` (`total > 0.8`) must continue to flag and route to counter-fraud review.
- **Standard handling of low-risk claims**: Claims with low internal risk and low vendor scores must continue to return `flagged: false` and proceed to standard claims handler allocation without publishing `fraud.score.flagged`.
- **Partial internal signal protection**: Claims exhibiting only one or two internal signals (raw score `< 0.8`) must not flag on internal signals alone without sufficient vendor risk corroboration.
- **Boundary conditions**:
  - `days_since_policy_start == 30`: Must not receive the `+0.4` early tenure increment [[kb:fraud-scoring/entities/fraud-signals]].
  - `claims_last_3_years == 1`: Must not receive the `+0.3` repeat claim increment [[kb:fraud-scoring/entities/fraud-signals]].
  - Perils other than `theft` and `accidental_damage`: Must not receive the `+0.1` peril increment [[kb:fraud-scoring/entities/fraud-signals]].
- **Input validation**: Requests missing required fields or containing invalid data types must continue to be rejected by Pydantic validation on `ScoreRequest` with HTTP `422 Unprocessable Entity` [[kb:fraud-scoring/entities/score-request]].

## Architecture, guardrails and standards

- **ADR [[kb:fraud-scoring/decisions/flag-threshold-and-counter-fraud-routing]]**: Updates routing logic to a dual-evaluation pattern (`total > FLAG_THRESHOLD or ours >= INTERNAL_FLAG_THRESHOLD`), removing the hard vendor dependency for severe internal fraud cases while preserving automated dispatch to counter-fraud specialists.
- **ADR [[kb:fraud-scoring/decisions/vendor-resilience-and-timeouts]]**: Implements the 2.0-second timeout on external requests in `app/vendor.py` and exception fallback to protect intake throughput and prevent thread starvation (resolving TWCLM-6).
- **Layering and modularity**:
  - `app/signals.py`: Remains a pure heuristic calculation function (`internal_score(req)`) with no side effects or network calls [[kb:fraud-scoring/entities/fraud-signals]].
  - `app/vendor.py`: Encapsulates vendor HTTP transport, timeouts, and error handling [[kb:fraud-scoring/entities/vendor-client]].
  - `app/main.py`: Houses API endpoints, threshold orchestration, event publication, and response serialization [[kb:fraud-scoring/summaries/api-spec]].
- **Coding standards**: Strict type hinting (`ScoreRequest`, `float`, `dict`), explicit module-level threshold constants, and standard logging of vendor fallback events.

## Test strategy

| Level | What it proves | AC or Contract Covered |
| :--- | :--- | :--- |
| **Unit** | `internal_score` boundary evaluation: verifies raw score is `0.8` when tenure `< 30`, claims `>= 2`, and peril is `theft`/`accidental_damage`; verifies raw score is `< 0.8` at boundary tenure (day 30), boundary claim count (1 claim), and standard perils. | Edge cases: tenure boundary, prior claims boundary, peril types [[kb:fraud-scoring/entities/fraud-signals]] |
| **Unit** | `vendor_score` error handling: verifies that `requests.RequestException` or HTTP timeout invokes client fallback, returning `0.0` instead of raising an unhandled exception. | AC-2, [[kb:fraud-scoring/decisions/vendor-resilience-and-timeouts]] |
| **Integration** | `POST /v1/scores` returns `flagged: true` and calls `publish_flag` when all three internal signals are present and vendor returns `0.0`. | AC-1, AC-5 |
| **Integration** | `POST /v1/scores` returns `flagged: true` and calls `publish_flag` when internal signals equal `0.8` and vendor times out or raises an error. | AC-2, AC-5 |
| **Integration** | `POST /v1/scores` returns `flagged: true` and calls `publish_flag` when internal risk is low (`0.0`) but vendor score exceeds `0.8`. | AC-3, AC-5 |
| **Integration** | `POST /v1/scores` returns `flagged: false` and does not call `publish_flag` when both internal risk and vendor risk are low. | AC-4, AC-5 |
| **Integration** | `POST /v1/scores` returns `flagged: false` for single or partial internal signals (e.g., tenure `< 30` with 0 prior claims and standard peril) when vendor score is low. | Edge cases: single internal signal, first-time policyholders |
| **Integration** | `POST /v1/scores` rejects malformed or missing payload attributes with HTTP `422`. | Edge case: input validation |
| **Contract** | Verifies `POST /v1/scores` response matches consumer schema expectations (`claim_id`, `score`, `flagged`). | Contract: `POST /v1/scores` |
| **Contract** | Verifies `fraud.score.flagged` event matches expected Pub/Sub schema (`claim_id`, `score`, `reasons`). | Contract: `fraud.score.flagged` |
| **End-to-End** | Ingesting a high internal risk claim via `claims.claim.reported` produces a `fraud.score.flagged` event and diverts the claim to the counter-fraud queue rather than standard handler assignment. | AC-1, AC-2, [[kb:fraud-scoring/concepts/claim-intake-integration]] |

## Rollout and rollback

- **Rollout order**:
  - `fraud-scoring` is an independently deployable microservice. Because its inbound and outbound contracts are unchanged, it can be deployed with zero cross-service deployment dependencies.
  - An environment variable toggle `INTERNAL_FLAG_ROUTING_ENABLED` (default: `true`) will be introduced to control standalone internal threshold checks.
- **Data migration**: None. The service is stateless and maintains no persistent database tables.
- **Rollback plan**:
  - If unexpected queue volume impacts the counter-fraud squad, set `INTERNAL_FLAG_ROUTING_ENABLED=false` via service environment configuration to immediately revert to composite-only threshold evaluation without redeployment.
  - Alternatively, revert the service container image to the previous build artifact.

## Risks

| Risk | Likelihood | Mitigation |
| :--- | :--- | :--- |
| **False-positive surge in counter-fraud queue** | Low | Standalone threshold `INTERNAL_FLAG_THRESHOLD = 0.8` requires all three internal signals to be satisfied simultaneously. Claims with 1 or 2 signals will not flag without vendor corroboration. Pre-release alignment with counter-fraud investigators conducted. |
| **Intake latency stall during vendor outages** | Low | Strict `timeout=2.0` on external vendor HTTP requests with `requests.RequestException` fallback in `app/vendor.py` guarantees request completion within acceptable latency bounds [[kb:fraud-scoring/decisions/vendor-resilience-and-timeouts]]. |
| **Duplicate scoring requests** | Low | Endpoint evaluation is purely deterministic and idempotent for identical claim attributes. |

## Verification checklist

- [ ] Scope matches the design: only `fraud-scoring` changes; no modifications made to `claims-intake`, `claims-management`, or counter-fraud downstream tools.
- [ ] Each contract marked unchanged is untouched: `POST /v1/scores`, `fraud.score.flagged`, and `claims.claim.reported` wire contracts remain identical.
- [ ] The architecture and standards named were followed: `INTERNAL_FLAG_THRESHOLD = 0.8` implemented alongside `FLAG_THRESHOLD = 0.8`; ADR [[kb:fraud-scoring/decisions/vendor-resilience-and-timeouts]] 2.0-second timeout and exception handling applied in `app/vendor.py`.
- [ ] Test strategy delivered at every level: unit tests for signal boundaries and vendor fallback, integration tests for AC-1 through AC-5, contract tests for REST and Pub/Sub interfaces, and end-to-end intake routing verification.
- [ ] Rollout and rollback mechanism verified: `INTERNAL_FLAG_ROUTING_ENABLED` feature toggle and container rollback proven ready in staging.
