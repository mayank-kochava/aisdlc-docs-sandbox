# Context Seed — K4A dashboard: fix date filter yesterday option

- **Work item:** wi_435f3850232dc2de5c7e
- **Requester:** Mayank Ukey
- **Product/channel:** K4A
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (4). (Desired behavior is unambiguous: 'yesterday' filter must return yesterday's data, not last week's. However, implementation spans 4 services (frontend + backend), requiring cross-layer coordination and debugging of the date…)
  - Requirement clarity: clear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 4 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** MEDIUM (Hard end-of-month deadline, affects 4 services, and root cause of date miscalculation is unknown—could be client-side, API, or analytics-query layer.)
- **Constraints/Deadline:** end of this month would be fine
- **Gate outcome:** approved with deviation — deviation: actually please change it so the fix is only for Acme Media's EU users, not everyone (owner: Mayank Ukey)
- **Next-round owner:** <@U09N9QMKE31> (mukey@kochava.com)

## Requirement Summary

## Requirement Summary

## Objective

Fix the K4A dashboard date filter so that when a user selects "yesterday," the dashboard displays only yesterday's data instead of last week's data. The issue is reported by Acme Media.

## Behavior & Rules

- When "yesterday" is selected in the date filter, the dashboard should return and display only the previous calendar day's data.
- Current behavior returns last week's data instead.

## Success Criteria

- Acme Media selects "yesterday" in the date filter.
- The dashboard displays only yesterday's data (not last week's data).

## Constraints & Deadline

- End of this month.

---

## Scope

- Product: K4A
- UI/Frontend change: **To be confirmed by engineering** (the date filter itself may or may not require UI changes; the display/behavior impact is unclear)

## Open for engineering

- Will its Acme Media see this change on screen in K4A, in a file or report they download, or not at all because it's backend-only? I'll assume on screen in K4A.

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- How is the "yesterday" filter currently calculating the date range—what logic bug causes it to return last week's data instead (off-by-one error, timezone issue, week-start assumptions)?
- Does this date filter bug affect other relative date options (e.g., "today," "last 7 days," "last month"), or is "yesterday" the only broken filter?
- What data sources or backend APIs power the dashboard, and do they need changes, or is the bug purely in the frontend query logic?
- Should the fix apply to all K4A users or only Acme Media, and are there any customizations in Acme Media's account that might be causing this behavior?
- How will we verify the fix works correctly—do we need to test against historical data, or will a simple date comparison suffice?
- Are there any dependent reports, exports, or scheduled jobs that also use this "yesterday" filter and might be affected by the fix?

## Intake Q&A

- **Q:** Which customer is seeing this, and when they pick yesterday in the date filter, what date range do they expect to see instead?

  **A:** its Acme Media, they expect to see just yesterday's data _(answered by Mayank Ukey)_

- **Q:** Will its Acme Media see this change on screen in K4A, in a file or report they download, or not at all because it's backend-only? I'll assume on screen in K4A.

  **A:** Unknown, for engineering to confirm _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- frontend-mos (frontend): K4A dashboard UI: date filter and display logic
- reporting-api (backend): Query/data retrieval for date-filtered reporting data
- analytics-query (backend): Date range parsing and query logic for analytics
- dashproxy-api (backend): Dashboard data proxy layer, may handle date filter translation

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- frontend-mos — role: K4A dashboard UI: date filter and display logic; kind: frontend; confidence: high
- reporting-api — role: Query/data retrieval for date-filtered reporting data; kind: backend; confidence: high
- analytics-query — role: Date range parsing and query logic for analytics; kind: backend; confidence: medium; endpoints: DELETE /analytics/accounts/{accountID}/filter-sets/{filterID}, POST /analytics/accounts/{accountID}/filter-sets, PUT /analytics/accounts/{accountID}/apps/{appID}/filter-sets/{filterID}, PUT /analytics/accounts/{accountID}/filter-sets
- dashproxy-api — role: Dashboard data proxy layer, may handle date filter translation; kind: backend; confidence: medium
