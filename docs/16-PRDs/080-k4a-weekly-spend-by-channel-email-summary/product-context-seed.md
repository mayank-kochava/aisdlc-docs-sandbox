# Context Seed — K4A: weekly spend-by-channel email summary

- **Work item:** wi_0040fac874c466d1e4b8
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (4). (Behavior is fully specified (weekly Monday email, spend per channel).))
  - Requirement clarity: clear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): no (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (5 weeks, summer campaign), multi-service blast radius, billing/spend data handling.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Create a weekly automated email summary for Meridian Travel's marketing team that sends every Monday morning, showing total campaign spend broken down by channel (one line per channel).

## Behavior & Rules

- Email delivery: every Monday morning
- Content: total spend per channel, one line per channel
- Format: matches what the marketing team currently copies into a spreadsheet manually
- Recipient: Meridian Travel marketing team

## Success Criteria

- Marketing team receives the email summary every Monday morning
- Summary contains total spend per channel in the format they currently use
- Eliminates the need for manual weekly data pulling

## Constraints & Deadline

- Customer is on the standard plan
- Delivery required before summer campaign launch, approximately 5 weeks from now

## Scope

- **Product:** K4A
- **UI/Frontend change:** No

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- Which specific channels should be included in the breakdown, and does this list change seasonally or by campaign type?
- What data source(s) will feed the spend totals—a single system or multiple platforms that need reconciliation—and how fresh does the data need to be (same-day, previous day, weekly aggregate)?
- Who exactly comprises "the marketing team," how many people need this email, and are there different permission levels for viewing spend data?
- What does the current manual spreadsheet format look like (column headers, number of decimal places, currency symbol, sorting order), and are there any calculations or derived metrics beyond raw channel spend?
- Does the spend data need any validation or anomaly flagging (e.g., alert if a channel's spend is 3x higher than usual), or should it be raw numbers only?
- Are there any integration or API limitations with K4A's standard plan that might prevent automated data export, and has this been validated with the technical team?

## Intake Q&A

- **Q:** Does Meridian Travel want this summary emailed to their marketing team every Monday, or delivered another way?

  **A:** Emailed every Monday morning, just total spend per channel, one line per channel. That is what they copy into a spreadsheet today. _(answered by Mayank Ukey)_

- **Q:** Where will Meridian Travel see this change?

A. On screen in K4A
B. In a file they download
C. Nowhere
D. In an email or message
Pick a letter, or answer in your own words.
  **A:** It is an email that lands in their inbox, nothing on screen in K4A _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-subscriptions-worker (backend): Implements scheduled email delivery of reports
- reporting-api (backend): Fetches campaign spend data by channel for report generation
- cost-v2 (backend): Provides spend aggregation and channel-level cost data
- accounts-api (backend): Manages customer subscription plan validation and entitlements

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-subscriptions-worker — role: Implements scheduled email delivery of reports; kind: backend; confidence: high
- reporting-api — role: Fetches campaign spend data by channel for report generation; kind: backend; confidence: high
- cost-v2 — role: Provides spend aggregation and channel-level cost data; kind: backend; confidence: high
- accounts-api — role: Manages customer subscription plan validation and entitlements; kind: backend; confidence: medium
