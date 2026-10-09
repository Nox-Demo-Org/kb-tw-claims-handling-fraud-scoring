---
mission: NOX-1
title: 'Route high-risk claims to counter-fraud review on internal signals alone'
role: product
status: draft
version: 3
author: dev
ai_drafted: false
---

# Product spec: Route high-risk claims to counter-fraud review on internal signals alone

| Column | Column |
| --- | --- |
|  |  |

## Goal
Ensure claims exhibiting strong internal fraud warning signs—such as new policies, frequent prior claims, and high-risk perils—are automatically flagged and routed directly to the counter-fraud investigation queue upon intake, without being blocked or downgraded by a low or missing external vendor score.

## User stories
- As a **counter-fraud investigator**, I want claims with severe internal risk indicators routed straight to my team's review queue upon submission, so that I can investigate high-risk files before any settlement, repair, or payout is approved.
- As a **claims handler**, I want suspicious, high-risk claims diverted away from standard claims processing queues automatically, so that I do not spend time triaging complex fraud cases that require specialist investigation.
- As an **automated claims routing service**, I want fraud evaluation to flag claims based on internal warning thresholds independently of third-party vendor responses, so that downstream routing to specialist queues remains resilient and accurate.

## Acceptance criteria
- **AC-1: Flagging on combined high internal risk factors with clean vendor score**  
  **Given** a claim submitted within 30 days of policy start, on an account with 2 or more claims in the past 3 years, for a theft or accidental damage peril,  
  **When** the claim is evaluated for fraud risk and the external vendor returns a clean or zero risk score,  
  **Then** the claim is marked as flagged for fraud review and routed to the counter-fraud investigation queue with the fraud flagged event published.

- **AC-2: Flagging on high internal risk signals with missing or unreachable vendor**  
  **Given** a claim submitted with maximum internal risk factors (policy age under 30 days, 2+ past claims in 3 years, high-risk peril),  
  **When** the external fraud vendor service times out or returns an error,  
  **Then** the claim is marked as flagged for fraud review and routed to the counter-fraud investigation queue.

- **AC-3: Flagging on strong external vendor score with low internal risk**  
  **Given** an established policy (30 or more days old) with fewer than 2 prior claims and a standard peril,  
  **When** the external fraud vendor returns a high fraud risk score that crosses the overall flagging threshold,  
  **Then** the claim is marked as flagged for fraud review and routed to the counter-fraud investigation queue.

- **AC-4: Standard routing for low internal and low vendor risk claims**  
  **Given** a claim with no internal risk indicators (established policy, 0 or 1 prior claim, standard peril) and a low external vendor risk score,  
  **When** the claim is evaluated for fraud risk,  
  **Then** the claim is marked as not flagged and proceeds directly to standard claims handler assignment queues.

- **AC-5: Scoring API response payload reflects flag status**  
  **Given** any claim evaluation request sent to the fraud scoring service,  
  **When** the evaluation completes,  
  **Then** the response indicates the overall risk score and explicitly sets the flagged indicator to true when internal or composite thresholds are met, and false otherwise.

## Edge cases

- **Boundary policy tenure (exactly 30 days)** *(from the map)*: A claim filed on day 30 or later must not receive the early policy risk increment [[kb:fraud-scoring/entities/fraud-signals]], while a claim filed on day 29 or earlier receives the full early tenure risk weighting.
- **Boundary prior claim count (exactly 1 vs. 2 claims)** *(from the map)*: A claimant with exactly 1 claim in the past 3 years must not trigger the repeat claim history increment, whereas 2 or more claims must trigger the risk increment [[kb:fraud-scoring/entities/fraud-signals]].
- **Unrecognized or low-risk perils** *(from the map)*: Claims for perils other than theft or accidental damage (such as windscreen damage or standard water leak) must not receive the high-risk peril increment [[kb:fraud-scoring/entities/fraud-signals]].
- **Vendor slowdown or outage** *(from the map)*: When the external vendor is slow or unavailable [[kb:fraud-scoring/entities/vendor-client]], claims meeting internal high-risk criteria must still flag immediately and not block standard claims intake.
- **Single internal signal present**: A claim with only one internal flag (for instance, an early claim on a clean record with a standard peril) must not flag on internal signals alone unless corroborated by a high external vendor score.
- **First-time policyholders (zero prior claims)**: A new policyholder filing their first claim (zero prior claims in the past 3 years) on a high-risk peril receives only partial internal risk points and must not be flagged on internal signals alone without corroborating vendor data.
- **Duplicate submission of flagged claim**: If an intake score request for the same claim ID is re-submitted, the service must produce consistent scoring and flagging outcomes.
- **Input validation and missing attributes**: If incoming claim data is missing required attributes or contains invalid negative values for policy age or claims count, the service rejects the request with standard schema validation errors before scoring.
- **Role permissions and queue access segregation**: Claims routed to the counter-fraud investigation queue must be accessible only to counter-fraud investigators; standard claims handlers cannot view or authorize payments on files while under fraud review.

## Out of scope
- Modifying the underlying list of internal heuristic signals (policy tenure threshold, 3-year claim count threshold, or peril classifications).
- Changing the downstream counter-fraud investigation queue user interface or investigative workflows.
- Replacing or changing the external third-party fraud vendor integration provider.
- Implementing automated payout blocking logic outside the fraud scoring and routing pipeline.

## Success metric
- **Baseline**: 0% of high-risk claims flagged on internal indicators alone when vendor scores are low or missing (currently requires vendor score above 0.8 to exceed threshold [[kb:fraud-scoring/decisions/flag-threshold-and-counter-fraud-routing]]).
- **Target**: 100% of claims meeting high internal risk criteria (early policy, repeat claimant, high-risk peril) routed to the counter-fraud queue regardless of vendor score *(suggestion)*.
- **Where the number comes from**: Claims operations reporting and counter-fraud intake audit logs comparing intake heuristic classifications against routing destinations.
- **When to read**: 14 days and 30 days post-deployment.

## Priority

**P1 (High)**: High-risk claims with severe internal warning signs are currently slipping past counter-fraud triage into standard claims queues whenever external vendor ratings are clean or unavailable. Doing nothing leaves the business vulnerable to preventable fraud leakage and burdens general claims handlers with manual triage.

### Stakeholder Communication
- **Counter-fraud investigators**: Must be notified prior to release about the expected intake volume increase of severe internal risk files triaged directly into their review queue.
- **Claims handling teams / operations**: Must be briefed on the shift in queue distribution, as high-risk early-tenure repeat claims will no longer appear in standard claims queues.
- **Claims support and intake handlers**: Must be informed of routing automation changes during intake.

## Verification checklist

- [ ] AC-1: Verify that a claim with all internal risk signals is flagged and routed to counter-fraud when the vendor returns a 0.0 or low risk score.
- [ ] AC-2: Verify that a claim with all internal risk signals is flagged and routed to counter-fraud when the vendor service is unavailable or times out.
- [ ] AC-3: Verify that a claim with low internal risk but a high vendor risk score continues to be flagged for fraud review.
- [ ] AC-4: Verify that a claim with low internal risk and low vendor risk routes to standard claims handling without being flagged.
- [ ] AC-5: Verify that the scoring API response correctly sets the flagged boolean field to true for high internal risk claims.
- [ ] Edge case (tenure boundary): Verify claim evaluation at day 29 vs. day 30 from policy inception.
- [ ] Edge case (prior claims boundary): Verify claim evaluation with 1 prior claim vs. 2 prior claims in the past 3 years.
- [ ] Edge case (peril type): Verify standard perils do not trigger the high-risk peril indicator.
- [ ] Edge case (vendor outage): Verify claim intake and routing continue operating during vendor service degradation.
- [ ] Edge case (single internal signal): Verify a single internal indicator alone does not trigger premature fraud routing without vendor corroboration.
- [ ] Edge case (first-time policyholder): Verify that a first-time claimant with no prior claims is not flagged on internal signals alone.
- [ ] Edge case (duplicate intake): Verify idempotency and consistent scoring on re-submitted claims.
- [ ] Edge case (input validation): Verify missing or invalid claim attributes trigger validation rejection before scoring.
- [ ] Edge case (queue permissions): Verify access restriction prevents standard claims handlers from settling claims in the counter-fraud queue.
- [ ] Success metric: Review claims operations reporting at 14 and 30 days to confirm 100% routing of high internal risk claims to counter-fraud review.
