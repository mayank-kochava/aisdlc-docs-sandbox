# Context Seed — SAM: migrate Search Ads Maven to Apple Ads Platform API v1

- **Work item:** wi_6d4ed0b23e11db57fe33
- **Requester:** Mayank Ukey
- **Product/channel:** SAM
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (10). (Campaign and reporting behavior are well-specified; missing-metric handling is explicit.))
  - Requirement clarity: clear
  - UX/design need: minor
  - Technical design clarity: unclear
  - Identified services: 10 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): unclear (Type A trigger if yes)
- **Risk tier:** HIGH (Affects 10 backend services, payment/spend data integrity, and financial decision-making; Dec 28 deadline leaves limited course-correct room.)
- **Constraints/Deadline:** Not a customer thing, this is for our own team. Target is Dec 28 2026.
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Requirement Summary

## Objective

Migrate Search Ads Maven from Apple Ads Campaign Management API v5 to Platform API v1. Update all API endpoints, headers, request/response handling, and data models to align with v1 for campaign, ad group, and keyword management, so that campaign creation, editing, pausing, bid and budget updates, and 4-hourly performance reporting continue to work identically to users.

## Behavior & Rules

**Campaign, Ad Group & Keyword Operations**

- Create, edit, and pause campaigns, ad groups, and keywords via v1 endpoints
- Update bids and budgets via v1 endpoints
- Operations must behave identically to v5 current behavior

**Metrics & Reporting**

- All seven metrics must be returned from v1 and displayed correctly: impressions, taps, installs, spend, ROAS, avg CPT, avg CPM
- If v1 stops sending any of the seven metrics: display blank or n/a, plus a flag indicating the metric is missing
- Do not show last known value or default to 0 for any missing metric
- Spend and ROAS: show louder/more prominent alerts when missing
- Other five metrics (impressions, taps, installs, avg CPT, avg CPM): show standard alert when missing

**Dashboards & Automations**

- Dashboards: display warning only when a metric shows blank or n/a; continue to show all other data
- Automated bid and budget adjustments: pause only the specific rule or campaign that depends on a blank metric; other rules and campaigns continue to run
- Do not pause entire dashboard or all automations

## Success Criteria

- Users see no difference in functionality or behavior post-migration
- All seven metrics sync and display correctly from v1 API
- Missing metrics trigger appropriate alerts (louder for spend and ROAS) without defaulting to 0
- Dashboards warn and continue; automations for affected campaigns/rules pause while others run
- Campaign management, optimization, and 4-hourly reporting continue without interruption

## Constraints & Deadline

- Target: December 28, 2026
- Internal team project (not a customer-facing change)

## Scope

**Product:** SAM (Search Ads Maven)
**UI/Frontend change:** Unclear — to be confirmed

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Next Round

- How will the product distinguish between "metric temporarily delayed from v1 API" versus "metric permanently unsupported in v1 API" when deciding whether to show a blank/n/a state or retry the fetch?
- What is the acceptable latency for the 4-hourly performance reporting pipeline to detect a missing metric and trigger the appropriate alert (standard vs. louder) before users see stale or incomplete dashboards?
- For automated bid and budget adjustment rules that depend on ROAS or spend: should a paused rule automatically resume once the metric reappears, or does it require manual user intervention to restart?
- Does the v1 API return metrics in a different format, structure, or timestamp cadence than v5, and if so, how should the frontend normalize or transform that data to maintain visual consistency?
- When a dashboard displays a warning for a missing metric, should users still be able to manually trigger a campaign edit or bid update, or should the UI block those actions until all metrics are available?
- Are there any v5 endpoints or data fields that v1 does not support, and if so, how should the product handle campaign or ad group configuration requests that rely on those unsupported fields?

## Intake Q&A

- **Q:** I found no existing spec for this migration. When Search Ads Maven stops working with v5, what do users see today?

  **A:** Nothing yet, v5 still works so users don't see any problem today. I'm not sure exactly when Apple turns v5 off or what would happen on screen if it did, probably failed calls and stale reports, but that's what we're trying to avoid. _(answered by Mayank Ukey)_

- **Q:** What should they see after the migration?

  **A:** Exactly what they see now. Creating, editing, pausing campaigns, ad groups and keywords works, bids and budgets update, and the performance numbers keep refreshing every 4 hours. They shouldn't be able to tell we switched APIs. _(answered by Mayank Ukey)_

- **Q:** For example, do campaigns fail to sync, do reports show errors, or does optimization pause?

  **A:** None of that is happening right now, it's about what could break once v5 goes away. Honestly any of those could happen, but the one I'd worry about most is reports looking fine while quietly showing 0 for a metric that stopped coming through. _(answered by Mayank Ukey)_

- **Q:** You said reports could quietly show 0 for a metric that stopped coming through. Which metrics or fields worry you most—the ones most likely to break in the switch, or the ones that would hurt most if they went silent?

  **A:** A, keep the first one. The second drops the spend/ROAS part and the full list, which I do want in there. _(answered by Mayank Ukey)_

- **Q:** You said the metric that worries you most could go silent—showing 0 instead of the real number. Which specific metrics from Search Ads Maven reports are most critical to get right in the v1 switch?

  **A:** Same as I just said: spend and ROAS are the critical ones, but all seven (impressions, taps, installs, spend, ROAS, avg CPT, avg CPM) need to come through correctly. _(answered by Mayank Ukey)_

- **Q:** What would it cost the team or our customers if one of them stayed at 0 for a week?

  **A:** Hard to put a number on it. Spend at 0 for a week and people could overspend or pause the wrong campaigns without realising, and ROAS at 0 would make good campaigns look dead. It'd erode trust in the numbers fast, and cleaning up after would take real time. No exact dollar figure from me. _(answered by Mayank Ukey)_

- **Q:** You said all seven metrics need to come through correctly. When the v1 API returns data, what should happen if one of those seven is missing from the response—should the report show the last known value, show blank, show 0, or fail the whole sync?

  **A:** A, those are the same thing really, keep the short one. _(answered by Mayank Ukey)_

- **Q:** You said spend or ROAS at 0 for a week would hurt trust and take time to clean up. Which of those seven metrics should the report show as blank or the last known value if v1 stops sending it, versus which ones should fail the whole sync to alert the team right away?

  **A:** All seven metrics: show blank or n/a plus a flag when v1 stops sending them. No last known value for any of them. Spend and ROAS get a louder alert. Failing the whole sync is left to engineering. _(answered by Mayank Ukey)_

- **Q:** You said all seven metrics should show blank or n/a plus a flag when v1 stops sending them, with spend and ROAS getting a louder alert. When a metric shows blank or n/a in a report, what should happen to any dashboards, alerts, or automations that depend on that metric—should they pause, show a warning, or keep running with the blank value?

  **A:** Keep the first (A). _(answered by Mayank Ukey)_

- **Q:** You said dashboards and automations should pause when a metric shows blank or n/a. Does that mean pause the whole dashboard, pause only the chart or card that uses that metric, or pause only the automation rule that depends on it?

  **A:** Not quite what I said. Dashboards only show a warning, nothing gets paused there. Only the automated bid and budget adjustments pause, and only the rule or campaign that depends on the blank metric, not everything. _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- k4a-sam-backend (backend): Implements v5→v1 API migration, endpoint/header/request-response updates
- sam-api (backend): Search Ads Maven API, exposed contract contracts for campaign/ad group/keyword management
- sam-campaign-data-sync (backend): Syncs campaign data from Apple API; must adapt to v1 response schema and field shapes
- sam-performance-reporter (backend): Reports seven critical metrics (spend, ROAS, impressions, taps, installs, avg CPT/CPM); consumes v1 response fields
- sam-agent (backend): Reads Apple-shaped fields (avgCPT, avgCPM, spend, ROAS, etc.) from SAM API; depends on adapter maintaining correct v1 field contract
- sam-util (backend): Shared utilities for campaign/ad group/keyword data models; likely owns v5 model definitions requiring v1 updates
- sam-mmp-ingestion (backend): Ingests performance data that feeds reporting; consumes v1 metrics
- sam-report-scheduler (backend): Schedules and triggers report generation; consumes performance data synced from v1
- sam-bulk-actions (backend): Bulk campaign/ad group/keyword operations; must call updated v1 endpoints
- sam-keyword-search-optimization (backend): Keyword optimization logic; depends on v1 keyword response structure

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- k4a-sam-backend — role: Implements v5→v1 API migration, endpoint/header/request-response updates; kind: backend; confidence: high
- sam-api — role: Search Ads Maven API, exposed contract contracts for campaign/ad group/keyword management; kind: backend; confidence: high
- sam-campaign-data-sync — role: Syncs campaign data from Apple API; must adapt to v1 response schema and field shapes; kind: backend; confidence: high
- sam-performance-reporter — role: Reports seven critical metrics (spend, ROAS, impressions, taps, installs, avg CPT/CPM); consumes v1 response fields; kind: backend; confidence: high
- sam-agent — role: Reads Apple-shaped fields (avgCPT, avgCPM, spend, ROAS, etc.) from SAM API; depends on adapter maintaining correct v1 field contract; kind: backend; confidence: medium
- sam-util — role: Shared utilities for campaign/ad group/keyword data models; likely owns v5 model definitions requiring v1 updates; kind: backend; confidence: high
- sam-mmp-ingestion — role: Ingests performance data that feeds reporting; consumes v1 metrics; kind: backend; confidence: medium
- sam-report-scheduler — role: Schedules and triggers report generation; consumes performance data synced from v1; kind: backend; confidence: medium
- sam-bulk-actions — role: Bulk campaign/ad group/keyword operations; must call updated v1 endpoints; kind: backend; confidence: medium
- sam-keyword-search-optimization — role: Keyword optimization logic; depends on v1 keyword response structure; kind: backend; confidence: low
