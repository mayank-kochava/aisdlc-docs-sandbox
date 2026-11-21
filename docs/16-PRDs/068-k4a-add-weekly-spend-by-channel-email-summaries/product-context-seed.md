# Context Seed — K4A: add weekly spend by channel email summaries

- **Work item:** wi_ec6f08247880c39f334a
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.85 — Type A trigger(s): requirements unclear, needs discovery; more than 2 identified services (4). (Delivery schedule, recipient list, channel scope, and data period remain unconfirmed; affects 4 backend services, requiring coordination across reporting and dispatch layers.))
  - Requirement clarity: unclear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Hard external deadline (5 weeks) with multi-service blast radius and unresolved critical parameters leaves little room for course-correction.)
- **Constraints/Deadline:** about 5 weeks, before their summer campaign launch
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Requirement Summary

## Objective

Send Meridian Travel's marketing team a weekly email summary showing total spend per channel, so they stop manually pulling the report every Monday.

## Behavior & Rules

- Weekly email delivery (day/time not yet specified)
- Content: total spend per channel breakdown
- Recipient: marketing team (specific email addresses/distribution list not yet confirmed)

## Success Criteria

- Weekly email is automatically generated and sent
- Email displays total spend by channel
- Marketing team stops manual weekly report pulls

## Constraints & Deadline

- Meridian Travel is on the standard plan
- Must be live before summer campaign launch in approximately five weeks

## Scope

- **Product:** K4A
- **UI/Frontend change needed:** Unclear — to be confirmed (request mentions email delivery, impact on user interface not yet determined)

---

**Items to confirm before proceeding:**

- Which day and time should the weekly email send?
- What are the email addresses or distribution list for the marketing team?
- Which channels should be included in the spend breakdown?
- Should this report include data from the previous week only, or a rolling period?

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

• What specific channels does the marketing team currently track, and are all of them available in K4A's data model?

• Should the email include only raw spend totals, or do stakeholders also need trend comparisons, performance metrics, or alerts if spend deviates from expected ranges?

• Are there any data access or permissions restrictions that would prevent certain team members from receiving the full spend breakdown by channel?

• How should the email handle weeks with missing or incomplete data—should it still send, delay, or notify the team of data gaps?

• Does the marketing team need to take action on this summary (like approving spend or adjusting budgets), or is this purely informational?

• What format or visual style would make the spend breakdown most actionable for the marketing team—simple text table, charts, comparison to budget, or something else?

## Intake Q&A

- **Q:** What does the weekly email need to show—which metrics or breakdowns matter most to their marketing team?

  **A:** Total spend per channel is enough. _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- reporting-api (backend): Query campaign spend data by channel
- reporting-dispatcher (backend): Schedule and dispatch weekly email job
- k4a-mcp-server (backend): K4A API server: expose spend summary endpoint
- sam-api (backend): SAM: provide spend and channel data source

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- reporting-api — role: Query campaign spend data by channel; kind: backend; confidence: high
- reporting-dispatcher — role: Schedule and dispatch weekly email job; kind: backend; confidence: high
- k4a-mcp-server — role: K4A API server: expose spend summary endpoint; kind: backend; confidence: high
- sam-api — role: SAM: provide spend and channel data source; kind: backend; confidence: high
