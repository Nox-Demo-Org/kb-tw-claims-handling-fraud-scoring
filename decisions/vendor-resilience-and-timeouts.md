---
type: Architecture Decision
title: Vendor Resilience and Timeouts
description: The vendor's agreed service level targets are 99.5% availability and a median response latency of 400 ms.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-fraud-scoring/blob/main/decisions/vendor-resilience-and-timeouts.md
tags:
- fraud-scoring
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:44Z'
---

# Vendor Resilience and Timeouts

## Status
Proposed / Pending Implementation (tracked in Jira issue `TWCLM-6` following incident review on 2026-08-27)

## Context
The `fraud-scoring` service evaluates fraud risk by combining internal heuristic signals (60% weight) with an external risk score from a third-party fraud data vendor (40% weight) as detailed in [[concepts/scoring-formula]]. External lookups are performed in `app/vendor.py` via `requests.post` to `FRAUD_VENDOR_URL` (default: `https://vendor.example/score`) as documented in [[entities/vendor-client]].

The vendor's agreed service level targets are 99.5% availability and a median response latency of 400 ms. However, the client implementation in `app/vendor.py` does not specify an HTTP timeout on `requests.post`. 

During a major vendor service degradation on 27 August 2026:
- The third-party fraud vendor response times exceeded 30 seconds for a continuous duration of 47 minutes.
- Without a client-side timeout, scoring execution threads blocked while waiting on vendor responses.
- Upstream claim intake and registration stalled, causing a queue backlog of 1,140 claims waiting on evaluation via [[concepts/claim-intake-integration]].

## Decision
1. **Set a 2-Second HTTP Timeout**: Configure `requests.post` in `app/vendor.py` with a strict `2.0` second timeout on requests to `FRAUD_VENDOR_URL`.
2. **Internal Signals Fallback**: If the vendor call times out or fails, fall back to evaluating the claim using internal signals from `app/signals.py` ([[entities/fraud-signals]]) alone.
3. **Review Flagging**: Because vendor data represents 40% of the overall calculation, claims scored during a vendor failure must be explicitly marked for review to account for missing vendor inputs.

## Consequences
### Positive
- Prevents upstream claim ingestion and registration pipelines from stalling during third-party service latency spikes or outages.
- Caps request execution time for `POST /v1/scores` and [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] consumers.
- Keeps claim processing throughput stable even when the external vendor breaches its 400 ms median SLA.

### Negative & Trade-offs
- Internal heuristic signals only account for 60% of standard score weighting. Bypassing the vendor alters score distribution relative to the standard `0.8` cutoff handled by [[decisions/flag-threshold-and-counter-fraud-routing]].
- Requires handling and marking fallback evaluations so counter-fraud teams know vendor data was omitted during scoring.
