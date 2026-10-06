---
mission: NOX-1
title: 'Route high-risk claims to counter-fraud review on internal signals alone'
role: business
status: draft
version: 1
author: dev
ai_drafted: false
---

# Business requirement: Route high-risk claims to counter-fraud review on internal signals alone

## The request
"Update our claim routing rules so that strong internal warning signs can flag a claim for fraud investigation on their own without requiring an external vendor score."

## Problem
When policyholders file a claim, our system evaluates risk to decide whether the claim needs specialist counter-fraud investigation or can proceed through standard claim handling. Today, that evaluation relies heavily on an external fraud rating vendor alongside our own internal checks (such as claims filed within days of buying a policy, repeat claims in recent years, or high-risk claim types like theft).

Even when a claim sets off every major internal alarm bell, our internal rules alone can never trigger an automatic fraud review on their own. Because external vendor ratings make up a large portion of the final score, a claim with severe internal red flags will slip straight into standard processing if the outside vendor returns a clean rating or fails to return a high score.

This creates a serious blind spot:
- High-risk claims bypass our counter-fraud team and risk being paid out without proper investigation.
- Our claims handlers spend unnecessary time processing suspicious files that should have been routed immediately to fraud specialists.
- The business absorbs preventable fraud losses.

## Who is affected
- **Counter-fraud investigators**: They miss suspicious claims during initial intake and often only discover issues after payments or repairs have already been approved.
- **Claims handlers**: They receive complex, high-risk claims in their regular queues instead of having them triaged straight to fraud specialists.
- **Business operations and finance**: The business experiences unflagged fraud leakage across dozens of suspicious claims each month.

## What should change
- Claims showing strong internal risk indicators (such as claims filed right after policy inception, policyholders with multiple recent claims, or theft/accidental damage) must be flagged for fraud investigation on the strength of those internal signals alone.
- An external vendor score should remain a helpful addition, but a low or missing vendor score must no longer prevent a claim with severe internal warning signs from routing to the counter-fraud queue.
- Claims handlers will see these high-risk files routed straight to the counter-fraud team upon submission, rather than landing in standard claims queues.

## What "done" looks like
- When a claim with multiple internal red flags is submitted, it immediately appears in the fraud review queue for specialist review.
- Claims operations reports confirm that claims with severe internal warning signs are routed to fraud investigation regardless of what the external vendor score was.
- Claims handlers no longer find unflagged, high-risk new-policy claims in their standard claim-handling queues.

## Examples

### Example 1: New policyholder with repeated claims
On October 12, 2026, David Miller submits a £4,200 theft claim on a policy purchased just 14 days earlier. His record shows two prior claims in the last three years.
- **Before**: The external vendor database returns a low risk score. Because the vendor score is low, the overall calculation falls short of the fraud threshold. The claim is routed to a standard claims handler for immediate payout processing.
- **After**: The combination of an early claim, repeat claims history, and theft triggers an automatic fraud flag based on internal warning signs. The claim is routed directly to the counter-fraud team's review queue before any payment is authorized.

### Example 2: First-party damage shortly after inception
On November 3, 2026, Sarah Jenkins files a £3,500 accidental damage claim 8 days after starting her policy, with a history of two previous claims.
- **Before**: The external vendor check is neutral, so the claim bypasses specialist review and sits in the general motor claims queue.
- **After**: The severe internal indicators alone meet the routing criteria for investigation, placing the file straight into the counter-fraud work list for document verification.

## Verification checklist
- [ ] Claims submitted with high internal risk factors (such as early policy inception and multiple past claims) appear immediately in the counter-fraud review queue.
- [ ] Claims with high internal risk factors are flagged for fraud investigation even when the external vendor returns a clean or low score.
- [ ] Claims with low internal risk and low vendor risk continue to route to standard claims handling without disruption.
- [ ] Claims operations reports show that suspicious claims are routed to fraud investigation based on internal warning signs.
- [ ] "Update our claim routing rules so that strong internal warning signs can flag a claim for fraud investigation on their own without requiring an external vendor score."
