# Context Seed — K4A: add weekly attribution report PDF download

- **Work item:** wi_8cb5da4783a748639b9c
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): requirements unclear, needs discovery; significant new UX/UI design; more than 2 identified services (6). (Report structure, content, scheduling cadence, and delivery mechanism (email vs. UI download) remain unspecified; affects 6 services across backend and…)
  - Requirement clarity: unclear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard year-end deadline (5 weeks), multi-service blast radius, and finance/billing data sensitivity elevate risk despite incomplete spec.)
- **Constraints/Deadline:** about 5 weeks
- **Gate outcome:** approved with deviation — deviation: yes confirm with note. (owner: Mayank Ukey)
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Provide Brightway Retail's finance team with a weekly attribution report as a downloadable PDF.

## Behavior & Rules

- Report generates on a weekly basis
- Report is available for download as PDF format
- Finance team can access and download the report

## Success Criteria

- Weekly attribution report is generated and available for download as PDF
- Report is delivered before year-end close (approximately 5 weeks from request date)

## Constraints & Deadline

- Brightway Retail is on the standard plan
- Delivery required: approximately 5 weeks

## Scope

- Product: K4A
- UI/Frontend change needed: Yes (download functionality for PDF reports requires confirmation)

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- What specific data fields or metrics should the weekly attribution report contain, and does this differ across different finance team members or departments?
- Who owns the responsibility for triggering the weekly report generation — is it automated on a schedule, manually requested by a finance team member, or both?
- Are there any existing attribution reports or formats the finance team currently uses that this PDF should match or build upon?
- How many weeks of historical attribution data need to be included in each week's report, or should each report contain only the most recent week's data?
- What are the finance team's requirements around report security, access controls, or audit trails for downloaded PDFs?
- Should the weekly report be sent automatically to the finance team's email inboxes, or only made available for on-demand download through the UI?

## Intake Q&A

- **Q:** This looks like a *K4A* request. Is that right?

Reply in this thread to keep going; start a new message for a new request.

Does Brightway Retail's finance team need this report on a fixed day each week, or just before year-end close?
  **A:** sorry, i mean the K4A thing is right. they want it every week but they really need it before year end close _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Attribution report generation and delivery API
- reporting-dispatcher (backend): Scheduled weekly report dispatch and delivery mechanism
- reporting-subscriptions-worker (backend): Weekly subscription scheduling and PDF report generation
- frontend-mos (frontend): K4A UI: download link, report preview, delivery status display
- k4a-mcp-server (backend): K4A MCP server coordination and plan entitlement check
- pubsuite-attribution (backend): CONSUMING attribution data K4A report will reference and display

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Attribution report generation and delivery API; kind: backend; confidence: high
- reporting-dispatcher — role: Scheduled weekly report dispatch and delivery mechanism; kind: backend; confidence: high
- reporting-subscriptions-worker — role: Weekly subscription scheduling and PDF report generation; kind: backend; confidence: high
- frontend-mos — role: K4A UI: download link, report preview, delivery status display; kind: frontend; confidence: medium
- k4a-mcp-server — role: K4A MCP server coordination and plan entitlement check; kind: backend; confidence: medium
- pubsuite-attribution — role: CONSUMING attribution data K4A report will reference and display; kind: backend; confidence: medium
