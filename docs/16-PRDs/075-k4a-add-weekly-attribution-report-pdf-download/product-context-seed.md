# Context Seed — K4A: add weekly attribution report PDF download

- **Work item:** wi_7be162d9becce019d57f
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): requirements unclear, needs discovery; significant new UX/UI design; more than 2 identified services (4). (Delivery mechanism (pull vs. push) remains unconfirmed; affects 4 services (backend + frontend), requiring data-flow and UI/UX design alignment.))
  - Requirement clarity: unclear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (year-end close, ~5 weeks), multi-service blast radius, and unresolved delivery model create high course-correction risk.)
- **Constraints/Deadline:** about 5 weeks, before their year-end close
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Provide Brightway Retail's finance team with a weekly attribution report as a PDF that they can download from K4A, containing last quarter data only.

## Behavior & Rules

- Report generation: weekly
- Format: PDF
- Access method: download from K4A (self-service)
- Audience: finance team at Brightway Retail
- Plan type: standard plan
- Data scope: last quarter only (finance team requirement)

## Success Criteria

- Finance team can download the weekly attribution report as a PDF from K4A
- Report is available on a weekly cadence
- Report contains last quarter data only

## Constraints & Deadline

- Delivery required before year-end close (approximately 5 weeks from request date)
- Brightway Retail is on the standard plan

## Scope

**Product:** K4A
**UI/Frontend change needed:** Yes (download capability for weekly attribution report PDF)

## Open for engineering

- PDF appearance and generation method (engineering-owned)
- Does Brightway Retail want the PDF sent to them on a schedule, or do they want to download it themselves from K4A when they need it?

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- How should the report handle week-to-week changes in "last quarter" data as the current quarter progresses—should it always show the same rolling 90 days, or reset to a fixed quarter at specific dates?
- What specific attribution metrics and dimensions must the PDF include, and in what order or grouping (by channel, campaign, product, geography, etc.)?
- If the finance team downloads the report multiple times in one week, should they receive identical data, or should each download reflect the most current available data?
- Does Brightway Retail need any filtering, date range customization, or drill-down capability within the PDF, or is a static pre-formatted report sufficient?
- Who should have permission to download this report—all finance team members, specific roles only, or a designated finance manager—and how should access control be enforced in K4A?
- Are there any compliance, audit, or data retention requirements for these weekly PDFs (e.g., must they be archived, signed, or timestamped)?

## Intake Q&A

- **Q:** Does Brightway Retail want the PDF sent to them on a schedule, or do they want to download it themselves from K4A when they need it?

  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Attribution report generation and delivery
- reporting-dispatcher (backend): Scheduled distribution of weekly attribution reports
- frontend-mos (frontend): PDF download interface for weekly attribution reports
- reporting-subscriptions-worker (backend): CONSUMES subscription/scheduling data to trigger weekly report generation

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Attribution report generation and delivery; kind: backend; confidence: high
- reporting-dispatcher — role: Scheduled distribution of weekly attribution reports; kind: backend; confidence: high
- frontend-mos — role: PDF download interface for weekly attribution reports; kind: frontend; confidence: medium
- reporting-subscriptions-worker — role: CONSUMES subscription/scheduling data to trigger weekly report generation; kind: backend; confidence: medium
