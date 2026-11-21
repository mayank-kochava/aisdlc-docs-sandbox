# Context Seed — K4A: add weekly campaign spend by channel email

- **Work item:** wi_f0359a5a91deef7e1ff0
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): requirements unclear, needs discovery; significant new UX/UI design; more than 2 identified services (6); confirmed UI/frontend change required. (Recipient and email trigger are stated, but scheduling UI, delivery confirmation, template design, and…)
  - Requirement clarity: unclear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): yes (Type A trigger if yes)
- **Risk tier:** HIGH (Hard 5-week deadline, multi-service blast radius, customer-facing email reliability, and new UI/UX work required.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Send the marketing team at Meridian Travel a weekly email summary of campaign spend by channel.

## Behavior & Rules

- Email is sent weekly
- Content: campaign spend summarized by channel
- Recipient: marketing team at Meridian Travel
- Plan: standard plan

## Success Criteria

- Weekly email is delivered to Meridian Travel's marketing team
- Email contains campaign spend breakdown by channel
- Eliminates the need for manual weekly data pulling (currently done every Monday)

## Constraints & Deadline

- Deadline: about 5 weeks, before their summer campaign launch
- Customer is on standard plan

## Scope

Product: K4A
UI/Frontend change needed: Yes

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- What specific channels need to be included in the spend breakdown (e.g., paid search, social media, email, display, affiliate), and do these channels vary by campaign or stay consistent week-to-week?
- Who exactly on the marketing team should receive this email, and should there be role-based or permission-based controls over who gets access to the data?
- What format and level of detail does the spend data need (e.g., total spend only, spend + impressions + conversions, comparison to budget or prior week)?
- Are there any filtering or segmentation needs for the data (e.g., by campaign, region, product line, or time period within the week)?
- How should the system handle weeks with missing or incomplete data from certain channels, and what's the acceptable latency for channel spend data to appear in the email?
- Does the standard plan tier currently support automated recurring email reports, or does this require infrastructure/feature work to enable scheduled delivery?

## Intake Q&A

- **Q:** Who at Meridian Travel should receive this weekly email?

  **A:** The marketing team at Meridian Travel. _(answered by Mayank Ukey)_

- **Q:** Where will the customer see this change?

A. On screen in K4A
B. In a file they download
C. Nowhere
Pick a letter, or answer in your own words.
  **A:** On screen in K4A _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query and aggregates spend-by-channel data
- reporting-dispatcher (backend): Schedules and delivers weekly email summaries
- frontend-mos (frontend): Displays email summary on-screen in K4A console
- notifications-api (backend): Sends email notification to marketing team
- accounts-api (backend): Verifies customer plan entitlement (standard plan check)
- k4a-mcp-server (backend): K4A API backend layer needs integration point

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query and aggregates spend-by-channel data; kind: backend; confidence: high
- reporting-dispatcher — role: Schedules and delivers weekly email summaries; kind: backend; confidence: high
- frontend-mos — role: Displays email summary on-screen in K4A console; kind: frontend; confidence: high
- notifications-api — role: Sends email notification to marketing team; kind: backend; confidence: medium
- accounts-api — role: Verifies customer plan entitlement (standard plan check); kind: backend; confidence: medium
- k4a-mcp-server — role: K4A API backend layer needs integration point; kind: backend; confidence: low
