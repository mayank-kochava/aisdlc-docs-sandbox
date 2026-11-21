# Context Seed — K4A: add weekly attribution report PDF download

- **Work item:** wi_fd32e0f751562211517d
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): significant new UX/UI design; more than 2 identified services (6). (Requirement is clear: weekly PDF pubsuite-attribution report for finance team. Design is unclear due to 6-service blast radius and unconfirmed UI/UX specifics.))
  - Requirement clarity: clear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard 5-week deadline before year-end close, 6-service backend+frontend blast radius, and financial reporting data sensitivity.)
- **Constraints/Deadline:** about 5 weeks, before their year-end close
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Deliver a weekly attribution report as a downloadable PDF to Brightway Retail's finance team.

## Behavior & Rules

- Report generates and becomes available on a weekly schedule
- Finance team can download the report as a PDF file
- Report contains attribution data for the last quarter only (scope and specific metrics assumed standard for K4A attribution reports)
- PDF generation approach and styling are engineering decisions made during design

## Success Criteria

- Finance team receives the weekly attribution report in PDF format before their year-end close
- Report is downloadable by the finance team on a recurring weekly basis

## Constraints & Deadline

- Brightway Retail is on the standard plan
- Delivery required within approximately 5 weeks, before year-end close

## Scope

**Product:** K4A
**UI/Frontend change needed:** Yes (download functionality for weekly PDF report requires UI component)

## Engineering Analysis

KG-identified services: (pending — to be confirmed during design review)

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- What specific attribution metrics must the PDF report contain, and does Brightway Retail need them in a particular format or layout (e.g., summary table, charts, breakdowns by channel)?
- Who exactly in the finance team needs download access, and are there any permission or role-based restrictions on who can view or download this report?
- Should the report automatically email the PDF to the finance team each week, or is a self-service download portal sufficient to meet their year-end close deadline?
- How far back should historical reports be retained and available for download (e.g., current week only, full quarter, rolling 12 months)?
- What happens if the weekly report fails to generate—does the system need to alert someone, retry automatically, or simply skip that week's delivery?
- Are there any compliance, audit, or data retention requirements Brightway Retail has for attribution reports that might affect how the PDF is generated, stored, or archived?

## Intake Q&A

- **Q:** Does Brightway Retail's finance team need this report on a schedule (for example, every Monday), or is a one-time download enough?

  **A:** sorry, i mean the K4A thing is right. they want it every week but they really need it before year end close _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Attribution report generation and delivery API
- reporting-dispatcher (backend): Scheduled weekly report dispatch and PDF export
- reporting-subscriptions-worker (backend): Manages scheduled report subscriptions and delivery
- frontend-mos (frontend): K4A UI: report download, PDF preview, download interface
- pubsuite-attribution (backend): Attribution data computation for weekly reports
- sam-report-scheduler (backend): Schedules and triggers weekly SAM/K4A report generation

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Attribution report generation and delivery API; kind: backend; confidence: high
- reporting-dispatcher — role: Scheduled weekly report dispatch and PDF export; kind: backend; confidence: high
- reporting-subscriptions-worker — role: Manages scheduled report subscriptions and delivery; kind: backend; confidence: medium
- frontend-mos — role: K4A UI: report download, PDF preview, download interface; kind: frontend; confidence: high
- pubsuite-attribution — role: Attribution data computation for weekly reports; kind: backend; confidence: medium
- sam-report-scheduler — role: Schedules and triggers weekly SAM/K4A report generation; kind: backend; confidence: medium
