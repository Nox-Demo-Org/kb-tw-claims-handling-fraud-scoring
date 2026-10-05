# Architecture decisions

One ADR per architecture decision the code or documents make evident.

## Pages

- [Decision: Flag Threshold and Counter-Fraud Routing](/decisions/flag-threshold-and-counter-fraud-routing.md) — The service produces a composite risk score between 0.0 and 1.0 by aggregating internal heuristic rules (fraud-signals) and external fraud intelligence (vendor-client) via the weighted model defined in scoring-formula: $$\text{total} = \…
- [Vendor Resilience and Timeouts](/decisions/vendor-resilience-and-timeouts.md) — The vendor's agreed service level targets are 99.5% availability and a median response latency of 400 ms.
