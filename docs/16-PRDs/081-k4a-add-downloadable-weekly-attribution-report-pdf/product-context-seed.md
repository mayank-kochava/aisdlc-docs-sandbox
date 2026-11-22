# Context Seed — K4A: add downloadable weekly attribution report PDF

- **Work item:** wi_88844a425eb289518ebf
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (4). (Desired behavior (weekly PDF, last-quarter data, on-demand download) is confirmed.))
  - Requirement clarity: clear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): no (Type A trigger if yes)
- **Risk tier:** HIGH (Hard deadline (5 weeks, year-end close), multi-service backend+frontend blast radius, and unresolved delivery architecture pose schedule and integration risk.)
- **Constraints/Deadline:** about 5 weeks, before their year-end close
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Provide Brightway Retail's finance team with a weekly attribution report as a PDF they can download from K4A, covering the last quarter only.

## Behavior & Rules

- Report is generated weekly
- Report covers the last quarter only
- Report is delivered as a downloadable PDF file
- Finance team accesses the report on-demand from K4A

## Success Criteria

- Finance team can download the weekly attribution report in PDF format, with last-quarter data only
- Report is available before year-end close (approximately 5 weeks from request date)

## Constraints & Deadline

- Brightway Retail is on the standard plan
- Deadline: approximately 5 weeks, before their year-end close
- Weekly delivery required

## Assumptions

- The PDF delivery mechanism (whether automatically generated on a fixed weekly schedule or generated on-demand when downloaded) will be confirmed during engineering review

## Scope

- Product: K4A
- UI/Frontend change needed: No

## Engineering Analysis

The PDF's appearance and generation method are left to engineering to determine during design.

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- How should the report handle weeks where attribution data is incomplete or still processing — should it include a data freshness timestamp, exclude incomplete weeks, or display a data quality warning?
- What specific attribution metrics and dimensions does the finance team need in the report (e.g., revenue by channel, conversion by campaign, ROI by touchpoint), and in what order or format?
- Should the weekly PDF always contain the same rolling 13-week window, or should it reset/change based on calendar quarters, fiscal quarters, or some other period definition?
- If the report is generated on-demand rather than on a fixed schedule, how should users know when new data becomes available, and what's the acceptable latency between data collection and PDF generation?
- Does the finance team need historical access to previous weeks' PDF reports, or only the current week's report covering the last quarter?
- Are there specific branding, compliance, or audit requirements (logos, disclaimers, row-level detail vs. summaries) that must appear in the PDF for year-end close purposes?

## Intake Q&A

- **Q:** Does Brightway's finance team need the PDF to arrive on a fixed day each week, or just whenever they download it from K4A?

  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Attribution report generation and PDF export
- reporting-dispatcher (backend): Report scheduling and delivery workflow
- frontend-mos (frontend): K4A UI: download link, report preview, delivery status
- pubsuite-attribution (backend): Attribution data source for weekly reports

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Attribution report generation and PDF export; kind: backend; confidence: high
- reporting-dispatcher — role: Report scheduling and delivery workflow; kind: backend; confidence: high
- frontend-mos — role: K4A UI: download link, report preview, delivery status; kind: frontend; confidence: high
- pubsuite-attribution — role: Attribution data source for weekly reports; kind: backend; confidence: medium
