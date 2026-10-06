---
mission: NOX-7
title: 'Originating desk visibility and escalation SLA for flagged orders'
role: business
status: draft
version: 1
author: dev
ai_drafted: false
---

# Business requirement: Originating desk visibility and escalation SLA for flagged orders

## The request
"Compliance should see which desk placed each flagged order. Any fraudelent activity needs to be escalated within an SLA."

## Problem
When a transaction is flagged for potential fraud, compliance officers cannot readily see which desk placed the order. Staff must spend time cross-referencing multiple internal systems and contacting desk managers to discover where the order originated. 

This tracking delay leaves the business exposed to financial loss and regulatory penalties. Without a clear time commitment for escalating confirmed issues, urgent cases sit in review queues for unpredictable amounts of time instead of reaching desk supervisors immediately.

## Who is affected
- **Compliance and counter-fraud officers** (around 15 staff members): They review dozens of flagged orders each day and need instant context to make fast decisions.
- **Desk supervisors and team leads** (across regional and trading desks): They need prompt notification whenever suspicious activity is tied to their desk.
- **Operations and compliance managers**: They are responsible for audit reporting, regulatory compliance, and meeting internal response standards.

## What should change
- Every flagged order in the compliance queue will clearly show the specific desk that placed it.
- Compliance staff will be able to filter, sort, and group flagged orders by originating desk.
- Each flagged order will track an agreed response timeline, making it obvious how much time remains before an escalation deadline is missed.
- Orders nearing their target deadline will be visually highlighted so investigators can prioritize them.
- When an order requires escalation, compliance officers will be able to send the alert directly to the responsible desk lead with all background details attached.

## What "done" looks like
- A compliance officer opens the flagged order queue and immediately sees the originating desk next to each flagged item without running secondary searches.
- Every flagged item displays its time remaining against the escalation deadline.
- Compliance managers can view summary reports showing flagged activity and escalation turnaround times by desk.

## Examples

### Example 1: Immediate desk identification and rapid escalation
On October 12, 2026, a $75,000 order is flagged for high risk. Compliance officer Sarah opens the alert and immediately sees that the order originated from the "London Commodities Desk". The screen shows a 2-hour escalation target. Sarah completes her initial review and escalates the case to the London desk manager in 25 minutes, well within the agreed window.

### Example 2: Managing urgent items across multiple desks
On October 14, 2026, the compliance team handles several incoming flags during peak market hours. The queue highlights two orders from the "New York Equities Desk" that have only 20 minutes left on their escalation timer. The team prioritizes these items first, completes the required checks, and escalates them before the deadline expires.

## Verification checklist
- [ ] Compliance officers can see the originating desk name directly on every flagged order in the review list.
- [ ] Compliance officers can filter and search flagged orders by desk.
- [ ] Each flagged order displays an escalation deadline timer indicating the time remaining to take action.
- [ ] Routine compliance reports show the number of flagged orders and escalation completion times broken down by desk.
- [ ] "Compliance should see which desk placed each flagged order. Any fraudelent activity needs to be escalated within an SLA."
