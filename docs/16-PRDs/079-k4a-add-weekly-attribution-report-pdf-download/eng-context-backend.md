# Engineering Notes: Brightway Weekly Attribution PDF - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-22
**Based on:** product-spec.md + live investigation of 10 candidate repos

---

## Initial Thoughts

Did a real pass through the repos. Good news first: the **pipeline shape we need already exists end-to-end**. `kochava_reports.scheduled` (cadence) → dispatcher polls → inserts `kochava_reports.requested` row → worker generates → uploads to S3/SFTP → writes `req_report_link` back → UI reads the link. That's exactly BR-08's "pre-generate and store for retrieval" model, and it's idiomatic across reporting-api, reporting-dispatcher, summary-report-cloud, koch-query-api, reporting-subscriptions-worker. The stored-link retrieval pattern (`req_report_link` / `sch_last_report_token`) maps cleanly to "download latest stored last-quarter PDF."

So most of this feature is **reuse + extend**, not greenfield. BUT — three pillars the spec hinges on do **not** exist anywhere I looked, and they're the whole point of the feature:

1. **PDF.** Grepped every candidate repo. Zero PDF rendering anywhere. Formats are CSV/JSON only — hardcoded, at every layer (reporting-api validates `csv`/`json` only; summary-report-cloud + koch-query-api + subscriptions-worker all CSV/JSON writers only). PDF is net-new end-to-end, and FR-014 (Brightway logo) has no hook anywhere.
2. **"Last completed calendar quarter" window.** This is the sneaky one. The dispatcher's window math is *rolling / relative-to-run-time*. There is NO calendar-quarter window — `timerange` options stop at `PreviousMonth`, and the `quarterly` frequency uses a **65-day heuristic** (`dtStart -= 65 days`), not real Jan/Apr/Jul/Oct boundaries. For a finance year-end close (BR-09 finance-grade), a 65-day approximation is a defect, not a window.
3. **Finance-team-only entitlement (BR-04).** NONE of the backend services have per-user/role entitlement. reporting-api/dispatcher/koch-query-api are all account + API-key + app-scoped. The actual finance-team gate has to live in the K4A frontend's `AdvUserPermissions` (frontend investigation confirmed there's no finance key today — it's net-new there). The "entitlements-api" repo the KG flagged is a **name collision** — it's App Store IAP entitlements, totally different domain. Ruled out.

Complexity: **medium-to-high**. Not because any one piece is hard, but because it's genuinely multi-service and two of the three net-new pieces (PDF render, quarter window) need design decisions before estimating.

---

## Concerns & Risks

- **BR-07 + BR-01 tension is real, not theoretical.** The dispatcher couples `sch_frequency` to *both* cadence AND window on the default path. We want weekly cadence (every Monday) but a FIXED last-completed-quarter window that only advances when a quarter closes. Existing fields can't express this together. Only decoupling path today is the `traffic_filtering.timerange` override — but it has no quarter value. So: new `timerange` enum ("previous_completed_quarter") + new quarter-boundary math, OR a new mechanism. Needs design.
- **Which repo renders the PDF?** summary-report-cloud is the best content fit (it already produces revenue + conversions grouped by `network_name` = "channel" — exactly the Report Contents, CA-4). But it has no PDF lib and the aggregated path *drops revenue*. If someone routes the finance report through the "aggregate" template by mistake, the PDF silently has no revenue numbers. That's a BR-09 landmine.
- **Deploy topology for the scheduler is a blind spot.** The dispatcher is a **single-instance systemd daemon on `reporting02`** (confirmed from its README — resets `running`→`queued` on restart, assumes single writer). That's good for not double-firing the weekly run, but it's also a SPOF and I couldn't see any prod k8s/HA manifest in any repo. Worth knowing who owns that box before we hang a finance-deadline feature off it.
- **Signed-link TTL vs. stable stored artifact.** subscriptions-worker/koch-query-api deliver time-limited signed S3 URLs. BR-08 wants a stable weekly artifact + NFR-006 wants secure time-limited delivery. These aren't contradictory but need reconciling: store the object long-term, generate a short-lived signed link per download (K4A re-proxies / re-signs). Just needs to be explicit.
- **Audit (FR-012) + alerting (FR-003/FR-006/NFR-003).** audit-api exists but is account+Admin-scoped, writes only `audit_logs`, and has no "record this download" hook wired for us. alerting-api is config-CRUD only — it doesn't even SEND alerts (the firing worker is elsewhere, not in this checkout). So "raise operational alert on skip/fail" and "immutable download audit" are both net-new integrations, not existing hooks. Don't under-scope these.
- **Hard 5-week deadline + multi-service blast radius + net-new PDF.** This is the HIGH-risk combo the spec calls out and I agree with it.

---

## Questions & Answers

### Q1: Which repo owns PDF rendering, and is there any existing PDF capability org-wide I didn't see?
- **For:** Backend (eng-plan) / whoever owns summary-report-cloud
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks estimating the single biggest net-new piece. summary-report-cloud is my lead candidate (right data shape) but has no PDF lib. Need to decide: add rendering to that worker vs. a new small render step vs. render in reporting-api. Can't size BR-02/FR-014 until this is settled.

### Q2: For BR-07/BR-08 — do we add a "previous completed calendar quarter" timerange value to the dispatcher and run it on a weekly cadence, or build a separate mechanism?
- **For:** Backend (eng-plan)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks the window-computation work. Existing `quarterly` frequency (65-day heuristic) is NOT finance-grade and can't be reused as-is. This is an architecture decision that affects dispatcher, reporting-api request schema, and correctness (BR-09).

### Q3: Where exactly does finance-team entitlement (BR-04/FR-010) get enforced, and what's the authoritative source re-derived per request (NFR-005)?
- **For:** PM + Frontend + Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-010 wants server-side enforcement on the download path even for direct requests bypassing the UI. Frontend can add a `v1-if`/route gate, but a direct-request bypass needs a *backend* check too. No backend service here has a finance-team concept. Need to know: is there a real entitlements/Sprinkler-backed source we re-derive from, and which service does the per-request check? Blocks FR-010/NFR-004/NFR-005.

### Q4: The spec's working assumption (DEC-004) is pre-generate-and-store; OQ-001 in the register says the OPPOSITE (generate fresh on demand). Which is it?
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** The whole backend architecture branches on this. BR-08/FR-001 and everything I investigated assumes pre-generate-and-store (and that fits the existing pipeline cleanly). But OQ-001 is marked "Resolved — generated fresh on demand." These directly contradict. If it's actually on-demand, NFR-001 latency (p95 ≤ 3s) is at risk because we'd be rendering a PDF + running BigQuery at click time. Need this nailed before any sizing.

### Q5: Which worker actually delivers the attribution figures, and can we guarantee they reconcile to the authoritative product figures (BR-09/NFR-011)?
- **For:** Backend / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** summary-report-cloud's SUMMARY path has revenue+conversions by network (good), but its AGGREGATE path drops revenue (CA-5). koch-query-api is the authoritative BQ path but has no canned "by channel" attribution rollup. Need a single source-of-truth + reconciliation check (NFR-011) or we risk a finance PDF whose numbers disagree with the dashboard — the exact failure mode NFR-014 calls the biggest risk.

### Q6: Where do FR-012 download-audit and FR-003/FR-006 operational alerts actually get written/fired?
- **For:** Backend (eng-plan)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** audit-api doesn't have a "report download" event hook and is Admin/account-scoped, not wired for us. alerting-api is config-only and doesn't send. Both FR-012 (immutable, async, append-only 24mo) and the skip/fail alerts are net-new integration work. Need to identify the real alert-firing service (not in any checkout here) and confirm the audit write path.

### Q7: Kill-switch (FR-013/NFR-012) — what's the config mechanism that reaches both the dispatcher (systemd daemon) AND the download path within a 5-min propagation window?
- **For:** Backend / Infra
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** The dispatcher is a single systemd box polling a loop; the download path is in reporting-api + frontend. A config-based switch that flips both without a deploy needs a shared config source. Blocks FR-013.

---

## Rough Scope

Reuse where I can, net-new where I must:

**Reuse (existing patterns):**
- `kochava_reports.scheduled` weekly cadence (`frequency=weekly`, `run_on`=Monday) — exists
- `kochava_reports.requested` + `req_report_link` stored-artifact pattern — exists
- summary-report-cloud SUMMARY query: revenue + conversions grouped by `network_name` — exists, this IS "by channel"
- koch-query-api as authoritative BQ data path — exists
- S3 upload + signed-link delivery — exists

**Net-new / extend:**
- **PDF rendering + Brightway logo** (FR-002, FR-014) — new writer/render step wherever we land it (likely summary-report-cloud or a new render stage)
- **"previous completed calendar quarter" window logic** (BR-07) — new timerange enum + quarter-boundary math in dispatcher; must NOT reuse the 65-day `quarterly` heuristic
- **`delivery_format: pdf`** accepted in reporting-api intake (`api.go` csv/json guard) + plumbed through dispatcher pass-through + honored by the worker
- **Finance-team entitlement check** on the server-side download path (FR-010/NFR-004/NFR-005) — new, fail-closed
- **Download audit write** (FR-012) — new, async, append-only, 24mo retention
- **Operational alerts** on skip/fail (FR-003/FR-006/NFR-003) — new integration to the real alert-firing service
- **Kill-switch config** (FR-013) reaching dispatcher + download path
- **Idempotency / no-overlap** on the weekly window key (NFR-009/NFR-010) — dispatcher already has single-writer + failure-count; confirm window-key idempotency
- **Reconciliation check** before serving (NFR-011)

**Services likely touched (backend):** reporting-api (intake + stored-link read + maybe download auth), reporting-dispatcher (window math + weekly schedule + routing), summary-report-cloud (data + PDF render), koch-query-api (authoritative figures), plus whatever the real alert-firing + audit services turn out to be.

**Ruled OUT after real investigation:** entitlements-api (name collision — IAP entitlements), audit-api (wrong scope/no hook as-is), alerting-api (config-only, doesn't send), reporting-subscriptions-worker (subscriptions data, not attribution — reference pattern only), reporting-qa (QA harness, not prod — but useful as the e2e test home + schema source of truth).

---

## Dependencies on Other Roles

- **Frontend (frontend-mos):** owns the finance-team entitlement key in `AdvUserPermissions` (net-new there), the download button gate, and the "report not available" / retry-error UI. They need the backend to populate the new permission + expose the stored-link endpoint. We need to agree on the FR-010 server-side check so a direct request can't bypass their UI gate.
- **PM:** resolve the DEC-004 vs OQ-001 contradiction (Q4), confirm the finance entitlement source (Q3), confirm BR-09 reconciliation expectations.
- **Data:** confirm the authoritative attribution figures source and that summary-report-cloud's SUMMARY path reconciles to it (Q5).
- **Infra / whoever owns `reporting02`:** the dispatcher is a single systemd daemon on one box — need to know HA/ownership and how the kill-switch config reaches it (Q7).
- **Whoever owns the real alert-firing service + audit write path:** not in any repo I checked — need pointed (Q6).

---

## Notes

- Strongest existing-pattern references worth copying: the `scheduled → requested → req_report_link` flow (documented cleanly in reporting-qa README + schema snapshots), and reporting-subscriptions-worker's generate→store→deliver + status/retry state machine.
- Shared contract to watch: `report_failure_max = 7` is duplicated across reporting-dispatcher and koch-query-api and must stay in sync — relevant to the skip/fail alert behavior (FR-003/FR-006).
- KG hint "reporting-api calls frontend-mos" is **wrong** — verified it's the inverse (frontend calls INTO reporting-api). Don't model that outbound edge.
- summary-report-cloud is a horizontally-scalable Pub/Sub consumer (each message delivered once), so the single-replica concern applies to the *dispatcher scheduler*, not the generation worker.
- reporting-qa is where I'd add the end-to-end test for this feature once built.

---

*Captured by `/kochava:eng-context backend` on 2026-11-22*