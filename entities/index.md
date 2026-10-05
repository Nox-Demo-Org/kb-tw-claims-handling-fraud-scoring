# Components and data models

One page per significant component and core data model.

## Pages

- [Fraud Signals](/entities/fraud-signals.md) — app/signals.py implements the internal heuristic scoring engine for fraud-scoring.
- [ScoreRequest](/entities/score-request.md) — ScoreRequest is a Pydantic data model defined in app/main.py that represents the incoming payload required to evaluate fraud risk for an insurance claim.
- [Vendor Client](/entities/vendor-client.md) — The vendor client (app/vendor.py) is responsible for communicating with an external third-party fraud risk API to retrieve external fraud indicators for insurance claims.
