# Engineering Notes: Weekly Attribution Report (Brightway) - Backend

**Engineer:** _unassigned_
**Date:** 2026-11-22
**Based on:** product-spec.md + live repo investigation (reporting-api, reporting-dispatcher, summary-report-cloud, koch-query-api, pubsuite-attribution, entitlements-api, reporting-subscriptions-worker, audit-api, alerting-api)

---

## Initial Thoughts

Did a real pass through all the candidate repos. Good news first: the *shape* of this system already exists end-to-end. There's a working pipeline — console/API creates a row in `kochava_reports.requested`, `reporting-dispatcher` polls that table and either execs a worker or pushes to Pub/Sub, a worker (`summary-report-cloud`) runs the BigQuery query, writes a file, uploads to S3, and POSTs status+link back. Weekly scheduling, summary reports grouped by `network`, revenue/conversion metrics — all real and in the code today. So we are extending a proven path, not greenfield.

Bad news: the three things the spec actually cares about most — **PDF output, the rolling-most-recent-completed-quarter window, and bounded (stops-at-year-end) weekly scheduling** — are exactly the three things that do NOT exist anywhere I looked. Every repo accepts only `csv`/`json`. No PDF library is a real (non-transitive) dependency in any of them. The quarter-window math and the "stop after year-end" cadence are both net-new. So the complexity is concentrated and I know where it is.

Also — the service-ownership question the spec flags is real and I think I can mostly resolve it: `summary-report-cloud` is the attribution-summary builder (it has the `network`-grouped revenue+conversions query), `pubsuite-attribution` is the upstream *writer* of conversion rows but has no read/query side, and `entitlements-api` / `reporting-subscriptions-worker` / `audit-api` / `alerting-api` are **ruled out** as content owners. More below.

Complexity for backend: **medium-high**, driven almost entirely by PDF + quarter-window + where PDF rendering should live. The plumbing is easy; those three decisions are the hard part and they're coupled to the deadline.

---

## Concerns & Risks

- **PDF is the single biggest gap.** No PDF/render capability exists in `reporting-api`, `summary-report-cloud`, `koch-query-api`, or `reporting-subscriptions-worker`. `gofpdf`/`rsc.io/pdf` show up only as transitive go.sum hashes, never imported. A `"pdf"` delivery_format would fall straight through the csv/json switches and produce a file with no extension and no body writer. So: new rendering stage, new accepted format value in `reporting-api` intake (`api.go:546`, `:1480` only allow csv/json), plus logo compositing. **Where** this renderer lives (extend summary-report-cloud worker? new service? render-from-CSV step?) is the key architectural decision and it's on the critical path against a hard 5-week deadline.

- **Rolling trailing-quarter, recalculated weekly, does not map cleanly onto any existing mode.** In `reporting-dispatcher`, the `weekly` window is pure day-count math (`root.go:1978-1981`); the `quarterly` case approximates a quarter as ~65–95 days and realigns to month start — but it runs on a *quarterly cadence*, not weekly. So "weekly cadence that always covers the most-recent-completed calendar quarter" is a genuine new window-resolution. `koch-query-api`/`summary-report-cloud` just run whatever `time_start`/`time_end` they're handed — so the quarter math has to live upstream in whatever creates the request row. A weekly+90-day window is NOT the same as "most recent completed calendar quarter" and finance will notice at a quarter boundary.

- **Bounded weekly cadence (stop at year-end) has no first-class support.** `kochava_reports.scheduled` has no expiry/end-date column (`queries.go:104`). Existing auto-stop is only the 7-failure → `sch_status='error'` path. BR-2's "lighter-weight interim scheduling, stops after year-end" means either a new expiry mechanism or someone operationally flipping `sch_status` after close. Needs a decision — I'd lean toward operational deactivation given the deadline, but flag it.

- **Failure-handling semantics don't match the spec.** Spec wants "retry once, then operational alert + email finance owner." What exists: dispatcher increments `sch_failed` and unschedules after **7** failures; `summary-report-cloud` has Pub/Sub nack redelivery + configurable status-write retries; neither does "retry exactly once then alert + email." `alerting-api` only *stores* alert configs — it has **no send logic at all** (no SMTP, no outbound HTTP), so the actual alert-send engine is a separate service I couldn't find in this checkout. The retry-once + ops-alert + finance-email flow is net-new and I don't yet know which layer owns it.

- **Download model mismatch for BR-7.** The existing workers *push* — upload to S3 and either email a presigned link (`koch-query-api`, `reporting-subscriptions-worker`) or POST the link back for the console to poll (`summary-report-cloud` → `reporting-api` `/progress`). The spec wants on-demand console download of a *durable* artifact with a *time-limited secure link* (NFR-005) retained **2 years** (NFR-007). Presigned S3 links expire quickly; need to confirm where the artifact is durably stored and who re-mints the time-limited link on each download. This is a reporting-api-side concern, not the worker's.

- **Multi-replica duplicate-generation risk is in the dispatcher, not reporting-api.** `reporting-dispatcher` is deliberately **single-instance** (one box `reporting02`, started by hand in tmux; on restart resets `running*`→`queued`). FR-004/NFR-009 (no overlapping generation, idempotent on account+cycle+window) lean on that single-instance assumption. Need to confirm nobody runs two dispatchers, and that the idempotency key actually holds.

- **"Attributed revenue" and "channel" are not clean columns at the source.** In `pubsuite-attribution` the conversions BQ rows have no `revenue` and no `channel` column — revenue is a *string* inside `event_metadata` JSON (`price`/`sum`/`currency`, all untyped strings), and "channel" maps at best to `partner` or `event_product`. In `summary-report-cloud` the summary query *does* have `event_revenue` and a `network`/`network_name` grouping — so the summary-rollup path is the viable source, not raw conversions. Need the data/analyst definition of exactly which column = "attributed revenue" for finance.

- **Audit trail = 2-year retention, immutable, async, alert-on-write-failure (NFR-007).** `audit-api` exists and has a `pkg/auditclient` emitter, but the spec doesn't currently require report events to be audited and no repo wires it in. This is net-new integration if we honor NFR-007 literally. Flagging as real scope.

---

## Questions & Answers

### Q1: Where does PDF rendering (+ Brightway logo) live?
- **For:** Backend (self/team) + PM for deadline tradeoff
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Biggest architectural decision and on the critical path. Options: (a) add a render stage inside `summary-report-cloud` after the existing BQ→CSV step; (b) a new small render service consuming the CSV/summary output; (c) render in `reporting-api` at serve time. Blocks estimation of the whole generation side.

### Q2: Is a weekly-cadence + ~90-day window "close enough" to "most recent completed *calendar* quarter," or must it be calendar-quarter-aligned (BR-3)?
- **For:** PM / finance
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Determines whether we can lean on existing day-count window math (cheap) or must build true calendar-quarter resolution upstream of the request row (more work). Materially changes effort.

### Q3: Which column is the authoritative "attributed revenue," and what is "channel" (network? partner? product?)?
- **For:** Data/analyst + PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** `summary-report-cloud` offers `event_revenue` + `network` grouping (viable), but currency normalization and the exact metric choice aren't settled. Blocks the query projection for BR-6 and the "one simple table" shape.

### Q4: Where does the "retry once → operational alert + email finance owner" failure flow live, and which service actually sends the alert/email?
- **For:** Backend (self) + PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Existing retry model (7-strike unschedule) doesn't match. `alerting-api` only stores config — the real sender is an un-found service. Blocks FR-003/NFR-008 design.

### Q5: How is the bounded cadence (stop at year-end, BR-2) enforced — new schedule-expiry column, or operational deactivation of `sch_status`?
- **For:** PM / team
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No first-class expiry support in `kochava_reports.scheduled`. Operational flip is cheap and deadline-friendly; a real expiry mechanism is more durable but more work.

### Q6: For NFR-005/NFR-007 — where is the generated PDF durably stored for 2-year retention, and who mints the per-download time-limited secure link?
- **For:** Backend (self) + infra + PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Existing S3 presigned links are short-lived. Durable storage + re-minting on download is a reporting-api/serving concern that needs design. Blocks download-side work.

### Q7: Is report-event auditing (NFR-007) actually in scope for MVP given the deadline, and if so do we use `audit-api`'s `pkg/auditclient`?
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Net-new integration not currently wired anywhere. Could be deferrable to reduce deadline risk, or could be a hard compliance requirement for a finance export. Need a ruling.

### Q8: Confirm `reporting-dispatcher` runs strictly single-instance in prod (so FR-004/NFR-009 non-overlap holds)?
- **For:** Infra / backend owner of reporting02
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Idempotency/non-overlap guarantees depend on this. If it ever runs 2x, duplicate generation is possible.

### Q9: Does the new report reuse the user-facing `schedule/` path (so finance could self-serve) or is it backend-only system-triggered?
- **For:** PM / frontend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Weekly recurring scheduling already exists in the console + `schedule/tokens`. If system-triggered, no UI schedule work; if self-serve, the primitive exists. Affects how/where the schedule row gets created.

---

## Rough Scope

What I think backend will need to do (service-by-service):

- **reporting-api** (intake/validation/serve):
  - Add `pdf` as an accepted `delivery_format` (currently rejects anything but csv/json at `api.go:546`, `:1480`).
  - Serve/durable-storage + time-limited link for on-demand console download (NFR-005), with server-side entitlement re-check on each download (NFR-004) — follow the existing API-key→account authz pattern (`AccountId` is already server-authoritative, `api.go:520`).
  - Possibly own the quarter-window resolution when creating the scheduled report row.
- **reporting-dispatcher** (schedule → requested row):
  - New window logic for most-recent-completed-quarter on a *weekly* cadence (today weekly=day-count only).
  - Bounded cadence / stop-at-year-end mechanism (or confirm operational deactivation).
  - Reconcile retry-once + alert semantics vs. the existing 7-strike model.
- **summary-report-cloud** (content builder — most likely PDF home):
  - Narrow the existing wide summary query to the "one simple table" projection: `event_revenue` + conversions grouped by `network` (BR-6).
  - Add PDF rendering + Brightway logo compositing after the BQ step (if we decide PDF lives here).
  - Add a `pdf` branch to `normalizeDeliveryFormat` + the extension/body-writer switches.
- **Failure/alerting**: wire retry-once → operational alert + email finance owner; find the actual alert-send service (not `alerting-api`, which is config-only).
- **Audit**: if in scope, emit generation + download audit events via `audit-api`'s `pkg/auditclient`, 2-year retention.
- **Kill-switch (NFR-010)**: config-flag disable without deploy; new requests reflect within 5 min, in-flight completes under prior behavior.

Service-ownership resolution (my read from the investigation):
- **Content owner = `summary-report-cloud`** (has the network-grouped revenue+conversions summary query).
- **Upstream data producer = `pubsuite-attribution`** (writes `aaas_<SAN>.conversions`), but no read side — not the report builder.
- **Ruled out as content source:** `entitlements-api` (subscription-billing only, no channel/attribution), `reporting-subscriptions-worker` (cancelguard/profile only), `audit-api` (config-change logs), `alerting-api` (alert configs).
- **`koch-query-api`** is a generic SQL executor — candidate *executor* only if content is expressed as supplied SQL; not the content owner.

---

## Dependencies on Other Roles

- **Frontend (frontend-mos):** low effort on their side — download surface (`CompletedReports.vue`) + status chips already exist; format list is data-driven from the backend template's `output_format`, so they only need a one-line `pdf?: boolean` type add + possibly an explicit empty-state message. **But** their PDF option only appears if **our backend template advertises `pdf: true`** — so backend has to deliver that first. Confirm sequencing.
- **Data/analyst:** need the authoritative definitions for "attributed revenue" column + "channel" mapping + currency handling (Q3). Blocks the query.
- **Infra:** confirm `reporting-dispatcher` single-instance (Q8); durable PDF storage + 2-year retention + secure link lifecycle (Q6); kill-switch config mechanism; deploy topology for any new render stage/service.
- **PM:** the three critical rulings — PDF home/approach, calendar-quarter strictness, and what's deferrable for the deadline (audit? self-serve scheduling?).

---

## Notes

- Existing pipeline anchors for reference: dispatcher scheduled-engine `root.go:1717-1927`; window math `root.go:1929-2007`; summary revenue/network query `summary_query_builder.go:142-203`; S3 upload + status POST-back `summary-report-cloud/main.go:782-857`, `:1104-1153`; intake format gate `reporting-api/api.go:546`.
- `reporting-api` does **not** call `frontend-mos` (verified — it's consumed *by* the console via CORS, not a caller). Rules out that KG hint.
- The `reportFailureMax = 7 // matches koch-query-api report_failure_max` comment confirms cross-repo coupling on the failure model — any change to retry semantics has to be reconciled across dispatcher + koch-query-api.
- Deadline reality check: PDF + quarter-window + storage/retention are the real cost. The plumbing is reuse. Scope discipline (defer audit/self-serve if possible) is the main lever to hit 4 weeks.

---

*Captured by `/kochava:eng-context backend` on 2026-11-22*