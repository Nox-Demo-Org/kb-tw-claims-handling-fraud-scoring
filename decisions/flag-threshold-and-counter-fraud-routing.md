---
type: Architecture Decision
title: 'Decision: Flag Threshold and Counter-Fraud Routing'
description: 'The service produces a composite risk score between 0.0 and 1.0 by aggregating internal heuristic rules (fraud-signals) and external fraud intelligence (vendor-client) via the weighted model defined in scoring-formula: $$\text{total} = \…'
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/decisions/flag-threshold-and-counter-fraud-routing.md
tags:
- fraud-scoring
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

# Decision: Flag Threshold and Counter-Fraud Routing

## Status
Accepted

## Context
When claims are submitted and processed through [[concepts/claim-intake-integration]] (triggered by [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] from [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] or direct calls to `POST /v1/scores`), `fraud-scoring` evaluates the risk of fraudulent activity.

The service produces a composite risk score between `0.0` and `1.0` by aggregating internal heuristic rules ([[entities/fraud-signals]]) and external fraud intelligence ([[entities/vendor-client]]) via the weighted model defined in [[concepts/scoring-formula]]:
$$\text{total} = \text{round}(0.6 \times \text{internal} + 0.4 \times \text{vendor}, 3)$$

A clear threshold is required to classify claims as high risk and trigger automated downstream escalation to the counter-fraud team without overwhelming them with false positives.

## Decision
We define a constant `FLAG_THRESHOLD = 0.8` in `app/main.py`.

When evaluating an incoming [[entities/score-request]]:
1. If the calculated `total` composite score is strictly greater than `0.8` (`total > FLAG_THRESHOLD`):
   - The response field `flagged` is set to `true` (see [[summaries/api-spec]]).
   - `app/main.py` invokes `publish_flag(claim_id, total)`, which publishes an event to the `fraud.score.flagged` Pub/Sub topic with payload `{claim_id, score, reasons}`.
2. The `fraud.score.flagged` event routes the claim to the counter-fraud team for specialized investigation and alerts downstream claims processing services.
3. If `total <= FLAG_THRESHOLD`, `flagged` is set to `false` and no `fraud.score.flagged` event is published.

## Consequences

### Positive
- **Automated Triage**: Claims with severe fraud indicators are immediately dispatched to counter-fraud specialists without requiring manual triage by standard claims handlers.
- **High Specificity**: Requiring a composite score $> 0.8$ ensures that internal heuristics alone (which can reach at most $0.8 \times 0.6 = 0.48$) cannot cross the threshold without significant corroborating risk signals from the external vendor ($0.4 \times \text{vendor} > 0.32$, meaning vendor score must exceed $0.8$).

### Trade-offs & Limitations
- **Vendor Reliance**: Because internal heuristics are weighted at 60% and capped at 0.8 raw signal points (giving a maximum contribution of 0.48), a claim cannot be flagged unless the third-party vendor returns a high fraud score (> 0.8). If the vendor is unreachable or degrades, flagging behavior is impacted (see [[decisions/vendor-resilience-and-timeouts]]).
- **Static Configuration**: The threshold is currently hard-coded as `FLAG_THRESHOLD = 0.8` in `app/main.py` rather than being dynamically configurable via environment variables or feature flags.
