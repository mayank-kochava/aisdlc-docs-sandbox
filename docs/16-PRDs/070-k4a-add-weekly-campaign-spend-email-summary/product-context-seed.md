# Context Seed — K4A: add weekly campaign spend email summary

- **Work item:** wi_de40697f4d6d9c822afd
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): requirements unclear, needs discovery; more than 2 identified services (5). (Email content and channel list are vague ('metrics, for their marketing channel'); multi-service backend integration (5 services) requires design work.))
  - Requirement clarity: unclear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (5 weeks), multi-service blast radius, and customer-facing reporting pipeline with compliance/billing data exposure.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Send Meridian Travel's marketing team a weekly email summary of campaign spend by channel, eliminating manual Monday reporting.

## Behavior & Rules

- Weekly email delivery containing campaign spend metrics grouped by marketing channel
- Metrics to include: installs, grouped by week
- Applies to Meridian Travel only (standard plan customer)

## Success Criteria

- Weekly email is automatically sent to the marketing team
- Email displays spend data by channel with installs metric and weekly grouping
- Manual data pulling process is replaced

## Constraints & Deadline

- Must be live before summer campaign launch in approximately five weeks
- Customer is on standard plan

## Scope

- Product: K4A
- UI/Frontend impact: unclear — requires confirmation whether a UI component is needed to configure or manage this email summary feature

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- How should the marketing team receive and access historical email summaries — will they need to retrieve past weeks' reports, and if so, through what mechanism (email archive, dashboard view, or downloadable exports)?
- Which specific marketing channels should be included in the spend breakdown, and how should channel names/categories be defined — are these predetermined or configurable by the customer?
- What is the definition of "installs" in this context, and should the email include other conversion metrics (signups, bookings, revenue) beyond installs and spend?
- Who should be on the recipient list, how many team members, and what process should Meridian Travel use to add/remove recipients as their team changes?
- What timezone and day/time should the email be sent each week, and how should the weekly window be defined (calendar week, rolling 7 days, Monday-Sunday)?
- Are there data accuracy or latency requirements — should the email include spend data through end-of-day Friday, or is there an acceptable delay (e.g., data available by Tuesday morning)?

## Intake Q&A

- **Q:** What does the weekly email need to show—which channels, which metrics, and in what order or grouping?

  **A:** metrics, for their marketing channel. installs and grouped by week _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query campaign spend and channel metrics for weekly summary
- reporting-dispatcher (backend): Schedule and dispatch weekly email job to marketing team
- k4a-mcp-server (backend): K4A MCP endpoint orchestrating the weekly report workflow
- sam-api (backend): Provide campaign channel and spend data for Meridian account
- notifications-api (backend): Send formatted weekly email to marketing team addresses

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query campaign spend and channel metrics for weekly summary; kind: backend; confidence: high
- reporting-dispatcher — role: Schedule and dispatch weekly email job to marketing team; kind: backend; confidence: high
- k4a-mcp-server — role: K4A MCP endpoint orchestrating the weekly report workflow; kind: backend; confidence: high
- sam-api — role: Provide campaign channel and spend data for Meridian account; kind: backend; confidence: medium
- notifications-api — role: Send formatted weekly email to marketing team addresses; kind: backend; confidence: medium
