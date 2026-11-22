# Context Seed — K4A: weekly email summary of spend by channel

- **Work item:** wi_18dbfe7e1fbfdc485f70
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (4). (Requirement is well-defined: weekly email with one line per channel showing total spend. However, design is unclear—affects 4 backend services, requiring scheduler/email orchestration architecture decisions.))
  - Requirement clarity: clear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (5 weeks), multi-service blast radius, and email delivery system integration create moderate implementation risk.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Send Meridian Travel's marketing team a weekly email summary showing total spend by channel for the previous week, so they no longer need to manually pull this data every Monday.

## Behavior & Rules

- Email contains one line per channel with that channel's total spend for the week
- Email is sent weekly
- Covers the previous week's spend data

## Success Criteria

- Weekly email is delivered to Meridian Travel's marketing team
- Email displays total spend by channel in a format ready for spreadsheet entry
- Manual weekly data pull is no longer needed

## Constraints & Deadline

- Meridian Travel is on the standard plan
- Must be live before their summer campaign launch in approximately five weeks

## Scope

**Product:** K4A
**UI/Frontend change needed:** No

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- Which specific channels should be included in the spend breakdown (e.g., paid search, social, display, email, affiliate, direct), and will this list change seasonally or by campaign?
- What constitutes "the previous week" in Meridian Travel's business context—calendar week (Mon-Sun), rolling 7 days, or some other definition—and how should it handle weeks that cross billing cycles?
- Should the email be sent on a fixed day/time each week (e.g., every Monday at 9 AM), and what timezone should that use given Meridian Travel's location(s)?
- Beyond total spend per channel, what other metrics or context does the marketing team currently pull manually (e.g., spend vs. budget, week-over-week change, clicks or impressions) that might prevent them from still needing manual reports?
- How should the email handle data delays or incomplete reporting from ad platforms, and what's the acceptable lag time between week-end and delivery?
- Who exactly on the marketing team should receive this email, and should there be different recipient lists or permission levels for viewing spend data?

## Intake Q&A

- **Q:** What does the weekly email need to show—just total spend by channel, or do they also need breakdown by campaign, geography, date range, or other dimensions?

  **A:** Just total spend per channel for the week is enough, one line per channel. That is what they copy into a spreadsheet today. _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query and fetch campaign spend by channel data
- reporting-dispatcher (backend): Schedule and dispatch weekly email jobs
- k4a-mcp-server (backend): K4A platform orchestration for scheduled tasks
- sam-api (backend): Provide spend and channel dimension data

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query and fetch campaign spend by channel data; kind: backend; confidence: high
- reporting-dispatcher — role: Schedule and dispatch weekly email jobs; kind: backend; confidence: high
- k4a-mcp-server — role: K4A platform orchestration for scheduled tasks; kind: backend; confidence: high
- sam-api — role: Provide spend and channel dimension data; kind: backend; confidence: high
