---
mission: NOX-8
title: 'Visible claim handler on customer claim tracker'
role: business
status: draft
version: 1
author: Maya Shah
ai_drafted: false
---

# Business requirement: Visible claim handler on customer claim tracker

## The request
"Customers keep calling to ask where their claim is. Can they just see who's handling it?"

## Problem
When policyholders report a claim online, their account shows only a generic "Submitted" label. Even after one of our claims handlers picks up the file and begins working on it, nothing on the screen changes for the customer. 

Because customers cannot tell whether anyone has looked at their file, they assume it is sitting forgotten in a queue. Anxious about repairs, rental cars, or replacement costs, they call our telephone support line simply to ask if anyone has started on their claim and who they are dealing with. 

These calls create two costly problems:
1. They clog our support phone queues with repetitive status checks, driving up wait times for people who have urgent new emergencies.
2. They pull support agents and claims handlers away from processing payouts to answer routine questions that should be obvious from the customer's account screen.

## Who is affected
- **Policyholders (customers)**: Roughly 3,500 customers each month who submit property, contents, or motor claims and log in to check progress while waiting for a decision.
- **Customer support agents**: Frontline phone representatives who spend up to a third of their daily call volume looking up claim references just to tell callers the name of their assigned adjuster.
- **Claims handlers**: Adjusters whose work is repeatedly interrupted by phone transfers and follow-up messages asking for simple confirmation that the claim is being handled.

## What should change
- When a customer logs into their online account and views an active claim:
  - Once a claim is assigned to a handler, the tracker should immediately show that an adjuster is actively reviewing the claim instead of lingering on "Submitted".
  - The customer should see the first name and team of the handler assigned to their claim (for example, "Sarah M. – Property Claims Team"), along with the best way to contact them or submit further documents.
  - If a claim has just been submitted and is still in initial triage, the screen should clearly state that we are reviewing the details and show the expected timeframe for an adjuster to be assigned (for example, "We are reviewing your claim. An adjuster will be assigned within 1 business day").
- When a claim is reassigned to another handler or specialist team, the tracker must update immediately so the customer always sees current information.

## What "done" looks like
- The weekly contact center report shows an immediate drop of at least 40% in inbound calls tagged under the category "Claim Status / Where is my claim?".
- An operations manager logging into the customer claims portal using a test claim can see the assigned handler's name and status as soon as the file is allocated to a team member.
- Average customer satisfaction scores on the post-claim survey for "Visibility during the claims process" improve from our current rating of 3.2 out of 5 to at least 4.2 out of 5 within 60 days.

## Examples
### Example 1: Home emergency claim (water damage)
On Monday, November 2, 2026, Arthur Pendelton submits a claim for a burst pipe in his kitchen (£2,400 estimated damage). 
- **Before**: Arthur logs in on Tuesday afternoon. The tracker still reads "Submitted". Nervous that no one has seen his photos, he calls our customer phone queue, waits on hold for 22 minutes, and an agent has to look up his policy to tell him that handler Marcus Vance took the file at 9:00 AM.
- **After**: Arthur logs in on Tuesday afternoon. The tracker displays "In Review — Assigned to Marcus V. (Home Claims Team)" alongside Marcus's direct extension and secure upload link. Arthur sees his file is in active hands, uploads his plumber's invoice, and never needs to call.

### Example 2: Stolen bicycle claim
On Friday, November 13, 2026, Maya Lin reports the theft of her road bike (£1,150). The claim is routed to counter-fraud review on Friday afternoon and assigned to a specialist handler on Monday morning.
- **Before**: Maya checks over the weekend and on Monday; the claim still appears stuck on "Submitted". She calls Monday lunchtime frustrated, asking whether her claim has been lost.
- **After**: Over the weekend, Maya's dashboard explains that her claim was received and is undergoing standard review. On Monday at 10:15 AM, when specialist handler Priya Patel picks up the case, the tracker updates to show Priya's name and contact guidance. Maya checks from her phone, sees progress, and leaves our phone lines free for new claims.

## Verification checklist
- [ ] Log in as a customer with a newly submitted claim and confirm the status clearly explains that the claim was received and is awaiting handler assignment.
- [ ] Log in as a customer whose claim has been assigned to an adjuster and confirm the adjuster's name and team appear clearly on the claim tracker.
- [ ] Reassign a claim internally to a different handler and confirm the customer tracker reflects the updated handler details without delay.
- [ ] Review the customer support weekly call report four weeks after release to confirm a measurable decrease in calls categorised as "Claim Status / Handler Query".
- [ ] "Customers keep calling to ask where their claim is. Can they just see who's handling it?" — customers can now see their assigned handler directly on their online claim tracker, and repetitive status calls have dropped.
