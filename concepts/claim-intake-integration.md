---
type: Concept
title: Claim Intake Integration
description: fraud-scoring integrates with the claim intake lifecycle to assess fraud risk automatically whenever a new insurance claim is registered across Tidewell systems.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/concepts/claim-intake-integration.md
tags:
- fraud-scoring
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

# Claim Intake Integration

`fraud-scoring` integrates with the claim intake lifecycle to assess fraud risk automatically whenever a new insurance claim is registered across Tidewell systems.

## Claims Intake Lifecycle

When a policyholder reports a claim via the intake journey (managed by `claims-intake`), the claim enters the initial lifecycle stage:

| Lifecycle Status | Managed By | Description | Customer View |
|---|---|---|---|
| `submitted` | `claims-intake` | Reported, awaiting handler assignment | Submitted |
| `assigned` | `claims-management` | Assigned to a handler (publishes `claims.handler.assigned`) | Submitted |
| `in_review` | `claims-management` | Handler is actively assessing claim | In review |
| `settled` | `claims-management` | Payout amount agreed | Settled |
| `paid` | `claims-management` | Payout sent (`payments.payout.sent`) | Paid |
| `declined / withdrawn` | `claims-management` | Closed without payment | Closed |

During the `submitted` phase, `fraud-scoring` evaluates the risk profile of the claim before or in parallel with claims triage and handler assignment.

---

## Event Ingestion: [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]]

The primary asynchronous entry point for claim scoring is the Pub/Sub topic subscription on [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] (Jira TWCLM-11):

1. **Trigger**: When a claim is reported via `claims-intake`, the [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] event is published.
2. **Consumption**: `fraud-scoring` listens for [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] messages to initiate scoring.
3. **Synchronous Invocation Alternative**: Upstream services (such as `claims-management`) can also evaluate fraud scores synchronously by calling the REST endpoint `POST /v1/scores` (see [[summaries/api-spec]]).

---

## Intake Scoring Workflow

```
claims.claim.reported / POST /v1/scores
              │
              ▼
   ┌──────────────────────┐
   │ Parse ScoreRequest   │ (claim_id, customer_id, peril, tenure, prior claims)
   └──────────┬───────────┘
              ├──────────────────────────────┐
              ▼                              ▼
   ┌──────────────────────┐       ┌──────────────────────┐
   │ Internal Signals     │       │ External Vendor API  │
   │ (app/signals.py)     │       │ (app/vendor.py)      │
   └──────────┬───────────┘       └──────────┬───────────┘
              │ (60% weight)                 │ (40% weight)
              └───────────────┬──────────────┘
                              ▼
               ┌────────────────────────────┐
               │ Composite Score Calculate  │
               └──────────────┬─────────────┘
                              │
                    score > 0.8?
                   /             \
             Yes  /               \  No
                 ▼                 ▼
   ┌───────────────────────────┐  ┌───────────────────────┐
   │ Publish fraud.score.flagged│  │ Complete / Return JSON│
   │ (routes to Counter-Fraud) │  │ (score, flagged=False)│
   └───────────────────────────┘  └───────────────────────┘
```

1. **Payload Extraction**: Attributes from the intake submission are mapped to the [[entities/score-request|ScoreRequest]] schema:
   - `claim_id`
   - `policy_id`
   - `customer_id`
   - `peril`
   - `days_since_policy_start`
   - `claims_last_3_years`
2. **Signal Evaluation**:
   - Internal heuristics score is calculated in `app/signals.py` based on policy tenure, frequency, and peril (see [[entities/fraud-signals]]).
   - External fraud score is queried from the third-party fraud vendor via HTTP in `app/vendor.py` (see [[entities/vendor-client]]).
3. **Aggregation**: The final score is computed using the 60/40 weighted formula in [[concepts/scoring-formula]].
4. **Flagging and Routing**: If the composite score exceeds the threshold (`FLAG_THRESHOLD = 0.8`), `publish_flag()` triggers a `fraud.score.flagged` event containing `{claim_id, score, reasons}` to route the claim to the counter-fraud team (see [[decisions/flag-threshold-and-counter-fraud-routing]]).

---

## Intake Latency and Vendor Dependencies

Because fraud scoring is tied to new claim intake, vendor latency directly impacts claim throughput:
- During an incident (TWCLM-6), high response times from the external vendor (>30s) led to 1,140 claims queuing during intake.
- Remediation and fallback mechanisms for vendor unresponsiveness during intake are documented in [[decisions/vendor-resilience-and-timeouts]].
