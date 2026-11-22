# Context Seed — K4A: add weekly attribution report PDF download

- **Work item:** wi_ed2c438d085ecaff4cc9
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): requirements unclear, needs discovery; significant new UX/UI design; more than 2 identified services (5). (Recurrence cadence is ambiguous (weekly schedule vs. one-time year-end need); 5 services involved (backend + frontend) requires multi-layer design work…)
  - Requirement clarity: unclear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (5 weeks, year-end close), multi-service blast radius, unconfirmed UX scope, and unclear recurring vs. one-time requirement.)
- **Constraints/Deadline:** about 5 weeks, before their year-end close
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Provide Brightway Retail's finance team with a weekly attribution report as a downloadable PDF.

## Behavior & Rules

- Report is generated and made available on a weekly schedule
- Finance team can download the report as PDF
- Report contains data from the last quarter only
- Applies to Brightway Retail (standard plan customer)

## Success Criteria

- Weekly attribution report covering the last quarter is available for download in PDF format
- Report is ready before Brightway Retail's year-end close

## Constraints & Deadline

- Delivery required before year-end close, approximately 5 weeks from request date
- Customer is on standard plan

## Scope

- **Product:** K4A
- **Report period:** Last quarter (Q4)
- **UI/Frontend change needed:** Yes (download functionality for PDF report)

## Engineering Analysis

- PDF appearance and generation method are engineering decisions

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- What specific attribution metrics or data points must the report contain, and how should they be organized or grouped for the finance team's year-end close process?
- How frequently does the finance team need to review or download the report during the week, and what time of day should it be generated to best support their workflow?
- Should the report include only Brightway Retail's own data, or does it need to compare or benchmark against other customers or industry standards?
- If the report fails to generate in a given week, what notification or fallback should alert the finance team, and who should be responsible for investigation?
- Are there regulatory or audit requirements that dictate the report's format, data retention, or how attribution is calculated and documented?
- Beyond PDF download, does the finance team need the ability to filter, customize columns, or regenerate the report for specific date ranges within the quarter?

## Intake Q&A

- **Q:** Does Brightway Retail's finance team need this report on a schedule each week, or is this a one-time download for year-end?

  **A:** sorry, i mean the K4A thing is right. they want it every week but they really need it before year end close _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Generate and serve weekly attribution reports
- reporting-dispatcher (backend): Schedule and dispatch recurring weekly report generation
- reporting-subscriptions-worker (backend): Manage scheduled report delivery and subscription workflows
- frontend-mos (frontend): K4A UI: add PDF download button/flow for attribution reports
- pubsuite-attribution (backend): Provide attribution data for K4A weekly report content

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Generate and serve weekly attribution reports; kind: backend; confidence: high
- reporting-dispatcher — role: Schedule and dispatch recurring weekly report generation; kind: backend; confidence: high
- reporting-subscriptions-worker — role: Manage scheduled report delivery and subscription workflows; kind: backend; confidence: medium
- frontend-mos — role: K4A UI: add PDF download button/flow for attribution reports; kind: frontend; confidence: high
- pubsuite-attribution — role: Provide attribution data for K4A weekly report content; kind: backend; confidence: medium
