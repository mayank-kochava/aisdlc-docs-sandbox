# Context Seed — K4A: add weekly campaign spend by channel email

- **Work item:** wi_f12db137adb1ddc28bd8
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): requirements unclear, needs discovery; more than 2 identified services (5). (Business rules are clear (weekly email, spend by channel), but email trigger logic, recipient list, report format, and data accuracy requirements need discovery. Multi-service backend…)
  - Requirement clarity: unclear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 5 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard 5-week deadline before live campaign, affects 5 backend services, and data accuracy is business-critical for spend reporting.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Requirement Summary

## Objective

Send Meridian Travel's marketing team a weekly email summary of campaign spend by channel, so they no longer need to manually pull the report every Monday.

## Behavior & Rules

- Weekly email delivery (day/time to be confirmed)
- Summary groups campaign spend by channel
- Automated generation to replace manual Monday report pull

## Success Criteria

- Marketing team receives email weekly without manual intervention
- Report accuracy is consistent (eliminates the need for secondary verification)
- Deployment completed before summer campaign launch

## Constraints & Deadline

- Standard plan customer
- Delivery required in approximately 5 weeks, before summer campaign launch
- Current manual process: ~2 hours per week for one analyst, with occasional data errors requiring re-verification

## Scope

- **Product:** K4A
- **UI/Frontend change needed:** Unclear — to be confirmed (request mentions email delivery; no frontend interaction specified)

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- What specific channels should be included in the spend summary, and do any channels need special calculation logic or data transformations before display?
- How should the email handle missing, delayed, or incomplete spend data from upstream sources — should it still send with partial data, delay delivery, or alert the team?
- Does the marketing team need any interactive elements in the email (like links to drill down into channel details), or is a static summary table sufficient for their workflow?
- What format and level of detail should the spend numbers take — totals only, totals with comparisons to prior week/budget, or breakdowns by campaign within each channel?
- Should this automation integrate with an existing data warehouse or reporting tool, or does it need to query campaign spend data directly from multiple sources?
- Are there any approval or review steps required before the email sends, or should it be fully automated without human intervention once deployed?

## Intake Q&A

- **Q:** What does pulling the report by hand cost them each week—time spent, or something else?

  **A:** About two hours every Monday for one analyst, and sometimes the numbers come out wrong, so the team has to check them again. _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query campaign spend by channel data
- reporting-dispatcher (backend): Schedule and dispatch weekly email summary reports
- k4a-mcp-server (backend): Expose scheduled report feature to K4A MCP/client
- sam-api (backend): Provide campaign channel-level spend data for Meridian Travel
- sam-report-scheduler (backend): Schedule and manage recurring K4A report delivery jobs

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query campaign spend by channel data; kind: backend; confidence: high
- reporting-dispatcher — role: Schedule and dispatch weekly email summary reports; kind: backend; confidence: high
- k4a-mcp-server — role: Expose scheduled report feature to K4A MCP/client; kind: backend; confidence: high
- sam-api — role: Provide campaign channel-level spend data for Meridian Travel; kind: backend; confidence: high
- sam-report-scheduler — role: Schedule and manage recurring K4A report delivery jobs; kind: backend; confidence: medium
