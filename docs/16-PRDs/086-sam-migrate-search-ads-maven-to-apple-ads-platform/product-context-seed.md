# Context Seed — SAM: Migrate Search Ads Maven to Apple Ads Platform API v1

- **Work item:** wi_dc4a014391ed4c6b5196
- **Requester:** Mayank Ukey
- **Product/channel:** SAM
- **Classification:** Type A (confidence 0.7 — Type A trigger(s): more than 2 identified services (6). (Behavior and success criteria are fully specified; requester confirmed no visible UI/workflow change.))
  - Requirement clarity: clear
  - UX/design need: none
  - Technical design clarity: unclear
  - Identified services: 6 (Type A trigger if >2)
  - UI/frontend impact (PM-confirmed): no (Type A trigger if yes)
- **Risk tier:** HIGH (Hard Dec 28 deadline, multi-service blast radius (6 services), external API contract change.)
- **Constraints/Deadline:** by Dec 28
- **Gate outcome:** approved
- **Next-round owner:** (unassigned)

## Requirement Summary

## Objective

Migrate Search Ads Maven from Apple Ads Campaign Management API v5 to Platform API v1, updating API endpoints, headers, request/response handling, and data models to align with v1 for campaign, ad group, and keyword management, so that all existing functionality continues to operate without visible change to customers.

## Behavior & Rules

**Campaign Management:**

- Campaign creation, editing, deletion, pausing, and resuming must work exactly as today.

**Bulk Operations:**

- Bulk keyword upload must continue to work. When v1 rejects a keyword, skip only that keyword and allow the rest to proceed; do not fail the entire upload.
- Rejected keywords must be shown to customers with a list of which keywords failed and the reason Apple provided.
- Results must display both a count of successful uploads and a list of skipped keywords with reasons.

**Bid & Budget Adjustments:**

- Manual and automated bid and budget adjustments must continue to work with the same handling as today.
- Timeout and rate limit errors must be handled the same way as today; no change in error handling behavior.

**Performance Reporting:**

- Reports must refresh every 4 hours and pull the same metrics as today: impressions, taps (clicks), installs, spend, ROAS, avgCPT, and avgCPM.
- Reports must display data at campaign, ad group, and keyword level, with the same fields and breakdowns as today.
- When v1 renames a metric field, map it back to the v5 equivalent so the value displays correctly.
- When v1 removes a metric with no equivalent, do not default to 0; instead show nothing or flag it as unavailable. Exact presentation is a design decision for engineering to propose.

## Success Criteria

- All campaign management workflows (create, edit, delete, pause, resume) function identically to v5 behavior from the customer perspective.
- Bulk keyword uploads complete with partial success: rejected keywords are listed with reasons; successful uploads are counted and shown.
- Performance reporting continues to refresh every 4 hours with the same metrics, breakdowns, and fields visible to customers.
- Metric mapping handles v1 field name changes transparently; no false zeros are displayed for missing or renamed fields.
- Bid and budget adjustments, timeouts, and rate limit handling remain unchanged from today.

## Constraints & Deadline

Must be completed by Dec 28.

## Scope

**Product:** SAM
**UI/Frontend impact:** No; the migration is internal; customer-facing screens for campaign management and reporting must remain visually and functionally identical.

## Related / Existing PRDs

- (none found)

## Open Questions for Next Round

## Open Questions for Discovery/Design

- How does the v1 API structure campaign, ad group, and keyword hierarchies, and what—if any—nesting or relationship changes require adjustment in our data model or state management?
- Which specific metrics does v1 rename or remove compared to v5, and what is Apple's recommended mapping or fallback behavior for metrics we can no longer retrieve?
- Does v1 impose different rate limits, pagination patterns, or batch size constraints than v5, and how should we adjust our bulk upload logic and reporting refresh cadence to stay within v1 quotas?
- What error codes and error response formats does v1 use for rejected keywords in bulk operations, and do we need to parse or transform them before displaying reasons to customers?
- How should we handle the transition period if any customer campaigns or keywords exist only in v5 (or only partially synced to v1), and do we need a one-time sync or reconciliation before full cutover?
- What is the exact timeline and rollout plan—do we migrate all customers at once on Dec 28, run a phased pilot, or support both APIs in parallel, and how does each option affect testing, monitoring, and rollback strategy?

## Intake Q&A

- **Q:** Search Ads Maven customers use campaign management, optimization, and reporting today via the v5 API. When you migrate to v1, what should they see change in the product, if anything?

  **A:** Ideally nothing. People shouldn't notice the switch. They're not touching the API themselves, SAM talks to Apple for them, so campaigns, ad groups, keywords, bids and budgets and the reports should look and behave exactly like now. _(answered by Mayank Ukey)_

- **Q:** Where will people see this change?

A. In a tool or API result
B. On screen in SAM
C. In a file they download
D. Nowhere
E. In an email or message
Pick a letter, or answer in your own words.
  **A:** Mostly D, nowhere, since it's meant to be invisible. If you want the closest one, it's B, the SAM screens where they manage campaigns and look at reports, those need to keep looking the same. _(answered by Mayank Ukey)_

- **Q:** For example, should campaign creation work the same way, or will the UI or workflow change?

  **A:** Same way. Creating, editing, deleting, pausing and resuming campaigns should work exactly as it does now, no UI or workflow change from my side. _(answered by Mayank Ukey)_

- **Q:** You said campaign creation, editing, deletion, pausing and resuming should work exactly as now. Does that cover everything customers do in SAM, or are there other workflows—like bulk operations, performance reporting, or syncing data to other tools—that also need to keep working the same way?

  **A:** Yes, those too. Bulk keyword upload, the manual and automated bid and budget adjustments, and performance reporting that refreshes every 4 hours all need to keep working the same. Syncing to other tools, not sure there's anything like that, I'd leave that to engineering to spot. And one thing, it's internal, there's no one customer behind this. _(answered by Mayank Ukey)_

- **Q:** You said bulk keyword upload, bid and budget adjustments, and performance reporting all need to keep working the same. When performance reporting refreshes every 4 hours, what metrics and data should it pull from Apple?

  **A:** Impressions, taps (clicks), installs, spend, ROAS, avgCPT and avgCPM, same as today, at campaign, ad group and keyword level. My worry is our metric object defaults everything to 0, so if v1 renames or drops one we'd show a fake 0 instead of an error. _(answered by Mayank Ukey)_

- **Q:** What should SAM show customers—the same fields and breakdowns as today?

  **A:** Yes, same fields as today. On breakdowns, I said campaign/ad group/keyword level a moment ago but honestly I'm not certain about every breakdown the reports have, so 'same as today' is the requirement, whatever that is. _(answered by Mayank Ukey)_

- **Q:** You said the metric object defaults everything to 0 if v1 renames or drops a field, and that's a worry. When v1 changes a metric name or removes one, what should SAM do—show an error to customers, show nothing, show 0, or handle it another way?

  **A:** Definitely not show 0. Ideally map the renamed field back to the v5 equivalent so the number is still right. If there's truly no equivalent, show nothing or flag it as unavailable rather than a made-up zero. Exactly how to surface that is a design call, I'd want engineering to propose it. _(answered by Mayank Ukey)_

- **Q:** You said bulk keyword upload, bid and budget adjustments, and performance reporting all need to keep working the same way. When those operations fail—a keyword upload times out, a bid adjustment hits a rate limit, or a performance report can't refresh—what should customers see today?

  **A:** Not sure what the exact error screens look like today, that's something I'd have to check or engineering would. What I do know I want: in a bulk keyword upload, if v1 rejects a keyword, skip only that keyword and let the rest go through, don't fail the whole upload. For timeouts or rate limits, same handling as today, I don't want it to change. _(answered by Mayank Ukey)_

- **Q:** You said if v1 rejects a keyword in bulk upload, skip that one and let the rest through. When a keyword is skipped, what should customers see—a list of which keywords failed?

  **A:** Yes, a list of the keywords that got skipped, and ideally the reason Apple gave if we have it. Whatever the upload result screen shows today for rejects, keep that. _(answered by Mayank Ukey)_

- **Q:** Why, or just a count of how many went through?

  **A:** Both would be nice: a count of how many went through plus the list of the skipped ones with the reason. A bare count alone isn't enough, they need to know which keywords to fix. _(answered by Mayank Ukey)_

## Appendix: Engineering Reference

_The sections below are derived automatically from the codebase knowledge graph, for the engineering team's reference during planning — not required reading for product/requirement review._

### Affected Services (agent-derived from the codebase KG)

- sam-api (backend): Implements v5→v1 Apple Ads API migration, endpoint and data model updates
- sam-agent (backend): Consumes Apple Ads metrics (avgCPT, avgCPM, impressions, taps, spend, ROAS) via SAM API; fixtures may need verification
- sam-performance-reporter (backend): Implements 4-hour performance report refresh using Apple Ads metrics; handles metric mapping and missing field logic
- sam-bulk-actions (backend): Implements bulk keyword upload with v1 API error handling, skip-and-continue logic, failure reason reporting
- sam-campaign-data-sync (backend): Implements campaign, ad group, keyword lifecycle (create/edit/delete/pause/resume) using v1 API
- frontend-mos (frontend): SAM UI: displays campaign management, reports, bulk upload results, error/unavailable metric messaging (no API calls; needs design/behavior verification)

### Technical Analysis

Technical analysis (agent-derived from the codebase KG):

- sam-api — role: Implements v5→v1 Apple Ads API migration, endpoint and data model updates; kind: backend; confidence: high
- sam-agent — role: Consumes Apple Ads metrics (avgCPT, avgCPM, impressions, taps, spend, ROAS) via SAM API; fixtures may need verification; kind: backend; confidence: medium
- sam-performance-reporter — role: Implements 4-hour performance report refresh using Apple Ads metrics; handles metric mapping and missing field logic; kind: backend; confidence: high
- sam-bulk-actions — role: Implements bulk keyword upload with v1 API error handling, skip-and-continue logic, failure reason reporting; kind: backend; confidence: high
- sam-campaign-data-sync — role: Implements campaign, ad group, keyword lifecycle (create/edit/delete/pause/resume) using v1 API; kind: backend; confidence: high
- frontend-mos — role: SAM UI: displays campaign management, reports, bulk upload results, error/unavailable metric messaging (no API calls; needs design/behavior verification); kind: frontend; confidence: medium
