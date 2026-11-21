# Context Seed — K4A: add weekly attribution report PDF download

- **Work item:** wi_97572e5ee0629908ff5f
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.95 — Type A trigger(s): requirements unclear, needs discovery; significant new UX/UI design; more than 2 identified services (6). (Report content and structure are unspecified; affects 6 services across backend, frontend, and data layers, requiring architectural alignment.))
  - Requirement clarity: unclear
  - UX/design need: significant
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard year-end deadline (5 weeks), multi-service blast radius, and unconfirmed UX scope create significant delivery risk.)
- **Constraints/Deadline:** about 5 weeks, before their year-end close
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

---

## Objective

Provide Brightway Retail's finance team with a weekly attribution report as a downloadable PDF.

## Behavior & Rules

- Report is generated and made available weekly
- Report is downloadable as PDF format
- Report covers the last quarter only
- Applies to Brightway Retail (standard plan customer)

## Success Criteria

- Finance team can download the weekly attribution report as PDF
- Report is available before year-end close

## Constraints & Deadline

- Deadline: approximately 5 weeks, before Brightway Retail's year-end close
- Customer is on the standard plan

## Scope

**Product:** K4A
**UI/Frontend change needed:** Yes (download functionality for PDF report)

---

## Engineering Analysis

PDF layout and generation approach are left to engineering.

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- What specific attribution metrics or data points must the weekly report contain, and does Brightway Retail have a preferred format or template they're expecting?
- How should the system handle weeks where attribution data is incomplete or still processing—should the report still generate, or should it block/delay until data is finalized?
- Will the weekly PDF report be manually triggered by the finance team each week, automatically emailed to them, or continuously available in a dashboard for download?
- Does "last quarter only" mean the report recalculates and includes only the most recent completed quarter each week, or does it always show a fixed quarter (e.g., Q4) regardless of when the report is generated?
- What happens to historical weekly reports—should they remain downloadable indefinitely, or should older reports be archived or removed after a retention period?
- Are there access controls or permissions needed, or can any user on the Brightway Retail account download these reports?

## Intake Q&A

- **Q:** What does the weekly attribution report include today?

  **A:** sorry, i mean the K4A thing is right. they want it every week but they really need it before year end close _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Generates and delivers weekly attribution reports
- reporting-dispatcher (backend): Schedules and routes weekly report delivery jobs
- frontend-mos (frontend): K4A UI: PDF download button, report management interface
- k4a-mcp-server (backend): K4A backend service handling report requests
- reporting-subscriptions-worker (backend): Manages scheduled weekly report delivery subscriptions
- pubsuite-attribution (backend): Provides attribution data consumed by weekly reports

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Generates and delivers weekly attribution reports; kind: backend; confidence: high
- reporting-dispatcher — role: Schedules and routes weekly report delivery jobs; kind: backend; confidence: high
- frontend-mos — role: K4A UI: PDF download button, report management interface; kind: frontend; confidence: high
- k4a-mcp-server — role: K4A backend service handling report requests; kind: backend; confidence: high
- reporting-subscriptions-worker — role: Manages scheduled weekly report delivery subscriptions; kind: backend; confidence: medium
- pubsuite-attribution — role: Provides attribution data consumed by weekly reports; kind: backend; confidence: medium
