# Context Seed — K4A: send weekly email summary of spend by channel

- **Work item:** wi_bc184d2d47ca4d6e58a6
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (4). (Desired behavior is unambiguous: weekly email with channel spend totals. However, 4 backend services are involved, requiring multi-service integration design.))
  - Requirement clarity: clear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (~5 weeks), multi-service blast radius, and email delivery dependencies create tight schedule risk.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Send Meridian Travel's marketing team a weekly email summary showing total spend per channel for the week, with one line per channel, to replace their current manual Monday pull.

## Behavior & Rules

- Email is generated and sent weekly
- Content: total spend per channel for the reporting week, one line per channel
- Recipient: marketing team at Meridian Travel
- Frequency: weekly (day/time to be confirmed during design)

## Success Criteria

- Marketing team receives the weekly email summary
- Email contains total spend broken down by channel
- Manual data pull process is no longer needed
- Meridian Travel stops pulling the data by hand each Monday

## Constraints & Deadline

- Meridian Travel is on the standard plan
- Must be live before their summer campaign launch in approximately five weeks

## Scope

- Product: K4A
- UI/Frontend change: unclear — to be confirmed during detailed design

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

• What specific channels does Meridian Travel currently track spend across, and do all of them have spend data available in K4A or do some require manual aggregation from external sources?

• Which day of the week and what time should the email be sent to best fit the marketing team's workflow, and who is the specific recipient email address or distribution list?

• Does the email need to show only total spend per channel, or should it also include comparison metrics like week-over-week changes, budget variance, or performance indicators?

• How should the email handle weeks with zero spend or missing data for certain channels — should those channels appear in the summary, and what should be displayed?

• Does Meridian Travel need to customize the channel names, order, or format of the email, and are there any branding or template requirements for how the summary should look?

• What happens if the weekly data pull fails or is delayed — should there be error handling, retry logic, or a notification to alert the team that the email wasn't sent?

## Intake Q&A

- **Q:** What does the weekly summary need to show?

  **A:** Just total spend per channel for the week is enough, one line per channel. That is what they copy into a spreadsheet today. _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query campaign spend data by channel
- reporting-dispatcher (backend): Execute scheduled weekly email job
- k4a-mcp-server (backend): K4A MCP server interface for email delivery
- sam-api (backend): Provide spend and channel data for K4A accounts

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query campaign spend data by channel; kind: backend; confidence: high
- reporting-dispatcher — role: Execute scheduled weekly email job; kind: backend; confidence: high
- k4a-mcp-server — role: K4A MCP server interface for email delivery; kind: backend; confidence: medium
- sam-api — role: Provide spend and channel data for K4A accounts; kind: backend; confidence: medium
