---
okf_version: '0.2'
title: fraud-scoring
description: 'fraud-scoring is a Python/FastAPI microservice owned by the Claims / Handling squad (on-call: #tw-claims).'
generated:
  at: '2026-10-05T12:46:44Z'
---

# fraud-scoring

`fraud-scoring` is a Python/FastAPI microservice owned by the **Claims / Handling squad** (on-call: `#tw-claims`). It scores every new insurance claim for fraud risk on a scale from 0.0 to 1.0. It evaluates claims by combining internal heuristics (60% weighting) with external risk data fetched from a third-party fraud vendor (40% weighting).

### Core Responsibilities
- **Score Ingestion and Calculation**: Evaluates claim attributes (e.g., peril, policy age, prior claims history) alongside external vendor data.
- **Flagging High-Risk Claims**: When the weighted risk score exceeds `0.8` (`FLAG_THRESHOLD`), the service publishes a `fraud.score.flagged` event so the counter-fraud team and downstream claims services can take action.
- **Claim Event Consumption**: Subscribes to [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] to trigger scoring workflows upon new claim submission.

### Data Flow & Architecture
1. Incoming requests arrive via `POST /v1/scores` (called by `claims-management` or downstream consumers) or through the [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] Pub/Sub event subscriber.
2. **Internal Scoring** (`app/signals.py`): Scores risk based on policy tenure (<30 days adds 0.4), claim frequency (>=2 claims in the last 3 years adds 0.3), and specific peril types (`theft` and `accidental_damage` add 0.1), capped at 1.0.
3. **Vendor Integration** (`app/vendor.py`): Queries an external fraud vendor via HTTP POST using `FRAUD_VENDOR_URL`.
4. **Weighted Aggregation**: Computes composite score `0.6 * internal + 0.4 * vendor`. If greater than 0.8, triggers `publish_flag` (`fraud.score.flagged`).

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 2 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 2 pages. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 3 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 1 page. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
