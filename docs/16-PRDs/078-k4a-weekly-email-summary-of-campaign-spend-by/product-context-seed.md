# Context Seed — K4A: weekly email summary of campaign spend by channel

- **Work item:** wi_f4bdc3e54dc471748d78
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): more than 2 identified services (4); confirmed UI/frontend change required. (Desired behavior is fully specified; however, email delivery & scheduling across 4 backend services requires architecture/integration design work.))
  - Requirement clarity: clear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): yes (Type A trigger if yes)
- **Risk tier:** HIGH (Hard 5-week deadline, multi-service blast radius, and scheduled email delivery adds complexity and integration risk.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Send Meridian Travel's marketing team a weekly email summary of total campaign spend by channel, so they no longer need to manually compile and copy this data every Monday.

## Behavior & Rules

- Email is sent weekly (frequency: once per week)
- Email contains total spend per channel only (one line per channel)
- Email is triggered/sent on Monday (or at a regular weekly cadence aligned to their manual Monday process)
- Data includes all campaigns for Meridian Travel
- Recipient: Meridian Travel's marketing team (specific recipients to be confirmed during design)

## Success Criteria

- Marketing team receives the weekly email without manual intervention
- Email format is clear and ready to forward or copy to spreadsheet
- Meridian Travel can stop their weekly manual pull and spreadsheet compilation process

## Constraints & Deadline

- Target delivery: before summer campaign launch, approximately 5 weeks from request date
- Meridian Travel is on the standard plan

## Scope

**Product:** K4A
**UI/Frontend change:** No (email only; no on-screen K4A interface changes required)

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- What specific channels does Meridian Travel currently track, and do they expect the email to include *all* channels from K4A or only a curated subset they define?
- Who exactly are the recipients of this email (names/roles), and do they all need to receive the same data, or should there be different summaries by team/geography?
- What time on Monday should the email send, and in what timezone—does Meridian Travel have a preferred delivery window to align with their workflow?
- Should the email include any context beyond total spend per channel (e.g., period-over-period comparison, campaign count, or performance notes) to make it immediately actionable?
- How should the email handle weeks with missing or delayed data from certain channels—should it send on schedule with partial data, wait for all channels to report, or flag incomplete data?
- Does Meridian Travel need the ability to customize which channels appear, adjust the email schedule, or modify recipients themselves, or is this a one-time static setup?

## Intake Q&A

- **Q:** What does Meridian's marketing team do with that spend-by-channel summary today?

  **A:** They copy the totals into a spreadsheet every Monday and send it up to their VP. Just total spend per channel is enough, one line per channel. _(answered by Mayank Ukey)_

- **Q:** Where will Meridian Travel see this change?

A. On screen in K4A
B. In a file they download
C. Nowhere
Pick a letter, or answer in your own words.
  **A:** It is an email that lands in their inbox, nothing on screen in K4A _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query campaign spend data by channel
- reporting-dispatcher (backend): Schedule & dispatch weekly email summary job
- k4a-mcp-server (backend): K4A platform backend: integrate scheduled reporting
- sam-api (backend): SAM campaign data source for spend-by-channel

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query campaign spend data by channel; kind: backend; confidence: high
- reporting-dispatcher — role: Schedule & dispatch weekly email summary job; kind: backend; confidence: high
- k4a-mcp-server — role: K4A platform backend: integrate scheduled reporting; kind: backend; confidence: high
- sam-api — role: SAM campaign data source for spend-by-channel; kind: backend; confidence: medium
