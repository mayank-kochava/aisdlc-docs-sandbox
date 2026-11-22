# Context Seed — K4A: add weekly attribution report PDF download

- **Work item:** wi_e2d03a4b901b4834728a
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): significant new UX/UI design; more than 2 identified services (3). (Requirement is clear: weekly PDF report of last quarter, downloadable from K4A before year-end.))
  - Requirement clarity: clear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 3 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard 5-week deadline, multi-service blast radius, PDF generation/delivery scheduling, and finance/billing-adjacent reporting data.)
- **Constraints/Deadline:** about 5 weeks, before their year-end close
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Provide Brightway Retail's finance team with a weekly attribution report as a PDF covering the last quarter that they can download from K4A.

## Behavior & Rules

- Report generation: weekly schedule
- Report format: PDF
- Report scope: last quarter of data only
- Report access: downloadable by the finance team from K4A
- Timing: must be available before Brightway's year-end close (approximately 5 weeks from request date)

## Success Criteria

- Finance team can download the weekly attribution report (last quarter data) as a PDF from K4A
- Report is generated and available on a weekly basis
- Report delivery meets the year-end close deadline

## Constraints & Deadline

- Customer plan: standard plan
- Deadline: approximately 5 weeks, before year-end close

## Scope

- Product: K4A
- UI/Frontend change: download capability for the weekly attribution report (exact interface location to be confirmed during design)
- PDF generation and formatting: to be determined by engineering
- Data scope: last quarter only

## Engineering Analysis

Engineering to define:

- How the PDF is generated
- How the PDF is formatted
- How to filter report data to last quarter
- Interface location and implementation approach for the download capability in K4A

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- What specific metrics, dimensions, and data points must the attribution report contain for the finance team's year-end close process?
- How should the report handle the rolling definition of "last quarter"—does it always show the same three calendar months, or does it update weekly to reflect the most recent completed quarter?
- Should the weekly PDF report be automatically generated and stored for retrieval, or generated on-demand when the finance team clicks download?
- What are the finance team's preferences for report organization, branding, and visual presentation (charts, tables, summary sections) in the PDF?
- Are there any data freshness or latency requirements—how recently must the underlying attribution data be processed before the report generates each week?
- Does Brightway's standard plan include the technical capacity and resource allocation needed to deliver weekly PDF generation and storage within the 5-week timeline, or will this require plan upgrade or phased implementation?

## Intake Q&A

- **Q:** Does Brightway's finance team need this report to arrive on a schedule, or just download it themselves from K4A when they need it?

  **A:** sorry, i mean the K4A thing is right. they want it every week but they really need it before year end close _(answered by Mayank Ukey)_

- **Q:** Where will Brightway Retail see this change?

A. On screen in K4A
B. In a file they download
C. Nowhere
My recommendation: B, because your request mentions a report or a file. Pick a letter, or answer in your own words.
  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Provides attribution report data and PDF export API
- reporting-dispatcher (backend): Schedules and delivers weekly reports to customers
- frontend-mos (frontend): K4A UI: download link and report delivery interface

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Provides attribution report data and PDF export API; kind: backend; confidence: high
- reporting-dispatcher — role: Schedules and delivers weekly reports to customers; kind: backend; confidence: high
- frontend-mos — role: K4A UI: download link and report delivery interface; kind: frontend; confidence: medium
