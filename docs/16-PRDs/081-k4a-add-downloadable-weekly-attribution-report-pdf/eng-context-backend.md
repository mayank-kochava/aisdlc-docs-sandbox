# Engineering Notes: Weekly Last-Quarter Attribution PDF - Backend

**Engineer:** (unassigned)
**Date:** 2026-11-22
**Based on:** product-spec.md (+ live repo investigation across 9 services)

---

## Initial Thoughts

Good news first: this is way more "extend existing machinery" than "build new." The reports pipeline already exists end-to-end — `reporting-api` queues a request row → `reporting-dispatcher` polls `kochava_reports.requested` every ~10s and routes to a worker (local exec or Pub/Sub) → worker (`summary-report-cloud` / `aaas-reporting`) runs a BigQuery query, writes a file, uploads to S3/GCS, and callbacks status + a download link. The frontend (`frontend-mos`) already has the download button, the queued/running/completed/error chips, and 10s progress polling. So the on-demand "generate → poll → download" UX is essentially already shipped for CSV/JSON reports.

The real work concentrates in three gaps, all backend:
1. **PDF output + Brightway logo** — does not exist *anywhere*. No PDF lib, no renderer, no logo-embedding in any report service. `reporting-api` hard-rejects anything that isn't `csv`/`json` (`api.go:546`). This is the single biggest net-new piece and it lives in the **worker**, not reporting-api/dispatcher.
2. **"Last quarter = most recent completed calendar quarter"** — no calendar-quarter logic exists in any service. The dispatcher's `quarterly` frequency is a ~65-day *approximation* (`root.go:1996-2004`, even has a suspicious double-subtraction bug). Workers take `time_start`/`time_end` pre-computed in the request. So quarter-boundary math is net-new and I think it should live upstream (whoever builds the request), computed at generation time.
3. **Revenue data** — this is the scary one. See concerns.

Complexity feels **moderate** overall, but with one genuine unknown (revenue) that could blow up the estimate if the answer is bad.

Also worth stating up front: the spec has an **internal contradiction** between BR-1 (on-demand, generate-fresh-at-download) and FR-001 (weekly system-triggered cadence). The codebase supports *both* models and investigation confirmed both paths exist. We need a ruling before we can design cleanly — I'm leaning hard toward on-demand since BR-1/DEC-001/DEC-007 all resolved that way and it needs the least new infra.

---

## Concerns & Risks

- **Revenue column may not exist in the attribution data.** This is my #1 risk. `pubsuite-attribution` writes the `conversions` BigQuery table (`aaas_<SAN>.conversions`) and a repo-wide grep for `revenue` returned **zero matches** — no revenue column, no channel column. `aaas-reporting`'s summary report also has conversion counts by network but **no revenue**. `summary-report-cloud` *does* have `event_revenue`/`event_count` grouped by `network` — so revenue data exists for *that* report's data source but it's unclear it's the same attribution data the spec intends. If "attributed revenue" has no backing field for Brightway's dataset, this is a **schema change + a product definition of what revenue even means** — potentially blocking for the 5-week deadline.
- **"Channel" is undefined against real fields.** Candidates: `network`/`network_name` (summary-report-cloud, aaas-reporting), `partner`, `claim_traffic_type`. The spec says "one row per channel" but existing summary reports group more finely (network + campaign + conversion_value). Need a precise mapping.
- **Which service actually generates the report?** The spec/KG name three candidate generation paths (reporting-api, summary-report-cloud, aaas-reporting) but investigation shows reporting-api only *queues* — it doesn't generate. The real worker is a separate repo. `summary-report-cloud` (Pub/Sub summary worker) and `aaas-reporting` (on-demand attribution w/ revenue+conversions-by-network already!) are both plausible homes. **We need to pick the generation home before estimating PDF work.** `aaas-reporting` looks like the closest semantic fit (attribution, on-demand, by-channel) *except* it has no revenue column and Brightway isn't in its static hardcoded auth map.
- **Must preserve existing `/v1` API contract — no breaking changes to response fields.** Any PDF/format additions to `reporting-api` must be additive only (new fields/enum values), not changes or removals to existing response fields. This reinforces NFR-010 backward-compat and constrains the `delivery_format: "pdf"` plumbing and any status/progress response changes.
- **5-week hard deadline + multi-service blast radius.** If PDF rendering + quarter logic + revenue schema all land across 2-3 services, that's a lot of coordination for 5 weeks. NFR-010 backward-compat means we can't regress existing CSV/JSON consumers of these shared services.
- **403 vs reality.** Spec flow says "server-side 403." Every service I looked at does something *else*: `reporting-api` returns **HTTP 200 + `{status:"Error"}`**; `mos-iam-server` returns **401**. Nothing returns a clean 403. If the spec truly requires 403, that's a behavior change with existing-client risk.
- **"Standard plan" has no home.** No service models plan tiers. `reporting-api` gates on `APIReportingAccess` flag; `mos-iam-server` uses per-org feature flags (`org_feature_mapping`); dispatcher only checks `status='activated'`. BR-5 will most likely be a feature-flag/entitlement, not a tier check — needs entitlements/PM confirmation.
- **KG false positives.** `reporting-subscriptions-worker`, `audit-api`, and `mos-iam-server` were flagged but investigation rules out the first two as implementation homes (wrong data domain; subscriptions worker is run-once per report-id, not a scheduler). The hoped-for "reusable weekly-cadence worker" is really the **generic scheduled-report loop in `reporting-dispatcher`** (`addScheduledReportThatAreReady`, `root.go:1717`), not a subscription-specific service. Good to know, but only matters if we go scheduled instead of on-demand.
- **Data-quality warning for incomplete weeks (FR-006).** No existing equivalent. `summary-report-cloud`'s `_PARTIAL` means row-cap-exceeded, totally unrelated. If this is truly Phase-1, it's net-new logic (and net-new in a PDF, which doesn't exist yet). Honestly feels in tension with on-demand "last completed quarter" — if the quarter is complete, is any week really still processing?
- **Logo source.** The Brightway logo lives in `mos-iam-server` (GCS-backed, PNG/JPEG only, URL on the org record). The PDF renderer in whatever worker will need to pull it from there — cross-service dependency, and PNG/JPEG only (no SVG).

---

## Questions & Answers

### Q1: On-demand (BR-1) or weekly scheduled (FR-001)? The spec says both.
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks the whole design. On-demand = reuse the one-time queue path (reporting-api `/summary` → dispatcher picks up `queued` row in ~10s → worker), minimal new infra. Scheduled = reuse `reporting-dispatcher`'s generic scheduled loop (`kochava_reports.scheduled`). FR-004/FR-005 (idempotency, no-overlap) only make sense for the scheduled model. Can't finalize scope or estimate until resolved.

### Q2: Does "attributed revenue" exist as data for Brightway's attribution dataset, and if so where?
- **For:** PM / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Potentially blocking. `pubsuite-attribution`'s `conversions` table has no revenue column; `aaas-reporting`'s summary has none either; only `summary-report-cloud` has `event_revenue`. Need confirmation of (a) what "attributed revenue" means as a number, (b) which table/column holds it for Brightway. If it needs a new schema column in `pubsuite-attribution` (`auditlog.go:337`), that's a write-path change + backfill question and likely won't be populated for last quarter retroactively.

### Q3: What exactly is "channel"? Map it to a real field.
- **For:** PM / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Determines the GROUP BY. Candidates: `network`/`network_name`, `partner`, `claim_traffic_type`. "One row per channel" (FR-002/report contents) needs the exact dimension before we can write/adapt the query.

### Q4: Which service is the generation home — adapt `aaas-reporting`, extend `summary-report-cloud`, or build a new report type in the dispatcher worker switch?
- **For:** Backend (architect) / me
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Decides where PDF rendering, quarter math, and logo embedding land. `aaas-reporting` is the closest semantic fit (on-demand attribution, conversions-by-network, 403-capable) but lacks revenue + has a hardcoded static auth map (no Brightway, no plan concept) + has no tests on handlers/reports. `summary-report-cloud` already has revenue+conversions by network but no on-demand-at-download trigger wiring and no attribution-specific grouping. Need an architect call.

### Q5: Does the spec really require HTTP 403, given no service currently returns one?
- **For:** PM / Backend (architect)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-010/NFR-005 say 403. `reporting-api` returns 200+error-body; `mos-iam-server` returns 401. Matching the spec literally is a behavior change with existing-client risk. Need to confirm whether "server-side denial" (any of the above) satisfies the requirement or whether a true 403 is mandated.

### Q6: Is "standard plan" the existing `APIReportingAccess` entitlement, a `mos-iam-server` feature flag, or something new?
- **For:** PM / Entitlements
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** BR-5. No service models plan tiers. Need to know which existing entitlement expresses "standard-plan + report-enabled" so we gate correctly (and so Brightway is actually entitled).

### Q7: Is the FR-006 incomplete-week data-quality warning truly Phase-1?
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** It's net-new everywhere (no existing completeness check, and the PDF itself is new). Also seems odd for a "most recent *completed* calendar quarter" window. If deferrable, it meaningfully reduces Phase-1 scope against the deadline.

### Q8: Audit trail (NFR-008) — is `audit-api` the intended home, and does it fit?
- **For:** Backend / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** `audit-api` exists and does who-did-what → BigQuery, with an edge-auth model. It could record download/generation events via its ingest endpoint. Need to confirm that's the plan vs. just logging within the generation service. Append-only + async-write-with-alert-on-failure (NFR-008) needs a concrete owner.

---

## Rough Scope

Assuming **on-demand** wins (Q1) and we land generation in a worker service:

- **PDF rendering (net-new):** pick/add a Go PDF lib, render the one-table layout (channel rows × revenue/conversions cols), embed the Brightway logo pulled from `mos-iam-server` org record / `/logos/download`. This is the bulk of the work.
- **Quarter-window computation (net-new):** "most recent completed calendar quarter," computed at generation time. Lives upstream (request builder) — inject explicit `time_start`/`time_end` epochs (the request contracts already carry these). Avoid the dispatcher's broken `quarterly` approximation.
- **`delivery_format: "pdf"` plumbing:** relax `reporting-api`'s csv/json guard (`api.go:546` + scheduled guard `:1480`) to accept `pdf`; dispatcher is already format-agnostic (pass-through, no change needed); worker learns to build PDF.
- **Report query:** adapt an existing summary/attribution query to GROUP BY the chosen channel field, SUM revenue + count conversions, scoped to the quarter window + Brightway's account/dataset.
- **Revenue sourcing:** depends entirely on Q2 — could be "use existing `event_revenue`," could be a schema change in `pubsuite-attribution`.
- **Access enforcement:** server-side authz on download/retrieve, re-derived per request, fail-closed. Likely a Keto check (`mos-iam-server` pattern) or the `APIReportingAccess`/feature-flag gate. Reconcile 403 vs 401/200 (Q5).
- **Secure time-limited link (FR-011/NFR-006):** already have this — S3/GCS signed/presigned URLs with short TTL (aaas-reporting V4 signed URLs, summary-report-cloud S3link). Just set a ~15min validity default, tunable.
- **Audit entries (NFR-008):** download + generation outcomes → `audit-api` ingest (pending Q8), async, alert-on-write-failure.
- **Kill-switch (NFR-009):** config flag disabling generation + download; propagate within 5min. Needs a home — likely feature flag.
- **Named failure statuses + bounded retry (FR-007/NFR-004):** distinct statuses for timeout / dependency-unavailable / rate-limited; 3 attempts exp backoff honoring Retry-After. Preserve prior successful report. summary-report-cloud already has retry-aware status callbacks to build on.
- **Read-rate cap on the attribution data source (NFR-002):** conservative tunable default so generation doesn't starve other BQ consumers.
- **Monitoring (NFR-003):** error-rate >1%/5min alert, data-source read-utilization >80%/5min alert. Reuse existing Prometheus/Datadog wiring in summary-report-cloud.

**Frontend (not mine, but tiny):** add `pdf?: boolean` to the `OutputFormat` type in frontend-mos; the format dropdown is already data-driven off the backend template's `output_format` map, so it auto-renders PDF once the template advertises it.

---

## Dependencies on Other Roles

- **PM:** resolve Q1 (on-demand vs weekly), Q2 (revenue existence/definition), Q3 (channel definition), Q6 (plan gating), Q7 (FR-006 scope) — these block design, not just estimation.
- **Data:** confirm where attributed revenue lives in BigQuery and the authoritative attribution data source (KG did *not* re-confirm `pubsuite-attribution` at the seed's confidence — "verify authoritative source"). Confirm channel field.
- **Architect / backend lead:** pick the generation home (Q4); rule on 403 vs existing conventions (Q5); decide where quarter math + kill-switch config live.
- **Entitlements:** which flag = "standard-plan + report-enabled," and ensure Brightway is entitled (Q6). `aaas-reporting` also needs Brightway added to its auth path if that's the chosen home.
- **Frontend (frontend-mos):** add `pdf` to `OutputFormat` type once backend template advertises it; wire a report entry point if we want a dedicated one (otherwise reuse existing Reports UI).
- **Infra:** replica/topology for whichever worker runs this. Note `reporting-dispatcher` is deliberately single-instance (bare-metal `reporting02`, resets in-flight rows on startup) — safe for a scheduled loop. The cloud workers scale as competing Pub/Sub consumers. Deploy manifests for the k8s workers live in external `ko-k8s-apps` — couldn't verify replica counts from the service repos.

---

## Notes

- **Code anchors to reuse (not rebuild):**
  - Queue/dispatch: `reporting-api` `/v1.4/summary` (`api.go:4021`) → row in `kochava_reports.requested` → `reporting-dispatcher` loop (`root.go:486`, `sendReportJobs` `root.go:757`) → worker exec/Pub/Sub (`root.go:1381`, `pushSummaryToPubSub` `root.go:1520`).
  - Status/progress: `reporting-api` `/v1.4/progress` (`api.go:4252`); frontend polling `reporting.ts:665-712`; status chips `CompletedReports.vue:39-95`.
  - Format dropdown data-driven off `template.output_format` (`CustomizeReportModal.vue:233`), type at `ReportingInterface.ts:114`.
  - Revenue+conversions-by-network query already exists in `summary-report-cloud` `summary_query_builder.go:142-203` (`event_revenue`, `event_count`, network grouping).
  - On-demand attribution submit/poll + signed-URL delivery in `aaas-reporting` (`skad.go:43-302`, `gcs-exporter.go:151-202`).
  - Logo: `mos-iam-server` `/logos/download` + org `LogoURL` (`logoservice.go:135-196`).
  - Scheduled-cadence engine (if we go scheduled): `reporting-dispatcher` `addScheduledReportThatAreReady` `root.go:1717`, weekly/quarterly supported but quarterly window math is approximate/buggy (`root.go:1996-2004`).
- **Ruled out as implementation homes:** `reporting-subscriptions-worker` (run-once, subscriptions domain, no scheduler), `audit-api` (audit domain only — reference pattern for read endpoints + account_id RBAC, and possible audit-trail sink), `mos-iam-server` (authz + feature-flag + logo source only, no report generation).
- The `reporting-dispatcher` deliberately single-instance property means a scheduled weekly job added there gets no-overlap (FR-005) essentially for free — worth remembering if Q1 lands on scheduled.
- Couldn't verify from source: the actual worker repo that builds CSV/JSON files for `reporting-dispatcher` (referenced only by binary name in the type switch `root.go:1263`, and by comment re: `koch-query-api`); k8s replica counts (external repo); whether Brightway's BQ conversions schema has revenue.

---

*Captured by `/kochava:eng-context backend` on 2026-11-22*