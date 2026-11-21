# Engineering Notes: Brightway Weekly Attribution Report (PDF) - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-21
**Based on:** product-spec.md

---

## Initial Thoughts

Spent time in the actual repos (reporting-api, summary-report-cloud, koch-query-api, reporting-dispatcher, reporting-subscriptions-worker, entitlements-api). Good news first: a LOT of the plumbing already exists. The flow boxes in the spec map pretty cleanly onto an existing pipeline:

- **reporting-dispatcher** — already a generic frequency-driven scheduler. Reads `kochava_reports.scheduled`, computes a window, inserts a `queued` row into `kochava_reports.requested`, and for summary-type reports **publishes a `ReportStruct` JSON to a prod Pub/Sub topic**. The "weekly generation (system-triggered)" box is basically a `scheduled` row with `sch_frequency='weekly'`, `sch_type='summary'`. Likely config/data, not new dispatcher code.
- **summary-report-cloud** — the Pub/Sub consumer that actually generates. Confirmed it **already produces revenue + conversions grouped by `network`/`network_name` (= channel)** from BigQuery `hourly_rollup`. So the content finance wants is computable TODAY. This validates the DEC "reuse summary-report-cloud" decision.
- **reporting-api** — the intake + status/link API. Download = poll `/progress`, get back the S3 link the worker wrote. Maps onto the self-service pull model.

So the skeleton is there. The pain is concentrated in a few specific gaps — PDF, the last-quarter window, the frozen-snapshot model, retention, and plan/finance access control — and those are spread across services and (worryingly) across teams.

Gut feeling on complexity: **medium-high.** Not because any one piece is huge, but because it's cross-repo, cross-team, and two of the gaps (PDF rendering, access control) have *no existing hook at all* to extend.

---

## Concerns & Risks

- **PDF is the single biggest gap, and it's confirmed absent everywhere.** Grepped all three generation/intake repos — `csv`/`json` only, zero `pdf`. reporting-api intake guards reject anything but csv/json (`api.go:546`, `:1480`). summary-report-cloud writes csv/json (`sqllite.go:310-347`, `main.go:772-776`). koch-query-api exports csv only. A PDF renderer has to be built *new* in summary-report-cloud, touching ≥4 functions, plus accepted as a `delivery_format` value through dispatcher + intake. The "one simple table + Brightway logo" shape argues for a narrow new PDF builder rather than retrofitting the CSV/SQLite pipeline.

- **"Weekly cadence + last-quarter window" is NOT expressible today.** This is the subtle one. In reporting-api/dispatcher the data window is a function of the *cadence* — a `weekly` schedule produces a rolling N-day window; only the `quarterly` *frequency* snaps to calendar boundaries. We want weekly cadence (runs every week) but a window pinned to the last **completed calendar quarter**. That combination doesn't exist. summary-report-cloud just takes `TimeStart`/`TimeEnd` epochs verbatim — so if the scheduler injects correctly-snapped epochs, the worker needs no window logic. The real question is *who computes the snap* and where it lives.

- **Frozen weekly snapshot (DEC-003) has no modeling anywhere.** `req_report_link` is just "the latest worker output" — no versioning, no immutability, no per-week identity-locking. And DEC-003 is explicitly NOT a confirmed decision (reviewer declined). If we honor it we need real storage modeling (store the week's artifact immutably, serve the same bytes on re-download). If re-generation overwrites in place (FR-004 idempotency), that *conflicts* with "identical re-download within the week" unless we freeze. Needs a clear answer before I can model the schema.

- **2-year retention (DEC-006) — no column, no job.** `kochava_reports.requested`/`.scheduled` have no TTL/expiry/retention field. summary-report-cloud has no S3 lifecycle config. This is likely an S3 bucket lifecycle policy + a DB marker — but it's net-new and probably an infra coordination item too.

- **Plan eligibility ("standard plan") has NO existing hook.** reporting-api has no plan/eligibility code. dispatcher gates on account `status='activated'` + API-key→app access, NOT plan tier. entitlements-api — despite the name — is app-store *subscription products*, not plan tiers; it has no plan/feature-gating mechanism and no user/role model. So the KG's "entitlements-api = authorization source" (low confidence) looks **wrong**, confirmed by reading it. There is no server-side authoritative plan-eligibility source identified yet. FR-013/NFR-009 (server-side, every request, fail-closed) need a real source and it doesn't exist in any repo I looked at.

- **"Finance-team members" access control doesn't fit the existing model.** Backend auth throughout is **account + app scoped by API key** — there's no user-role / "finance team" concept server-side. FR-010/FR-011 want per-user eligibility re-derived from an authoritative source on each request. Where does "is this user a finance-team member" come from? Not identified. This is a real design gap, not just wiring.

- **Audit trail (FR-017/NFR-012) — append-only, immutable, async, alert-on-write-failure.** There's a Raptor audit client in entitlements-api and a reporting-logging API referenced from summary-report-cloud, but nothing that gives me a confirmed immutable report-access audit store. Need to confirm what satisfies "cannot be edited/deleted by any role or code path."

- **Secret-in-repo (incidental):** hardcoded service-account private key in summary-report-cloud `bq.go:35-47`. Out of scope for this feature but flagging — if we're touching that repo it should get raised.

- **Cross-team coordination risk.** The work is genuinely spread: dispatcher (single-replica systemd box on reporting02), summary-report-cloud (Pub/Sub worker), reporting-api (intake/status), frontend-mos (download UI). HIGH-risk + multi-service is exactly why NFR-014 mandates a kill-switch. No existing kill-switch found; that's additive work too.

---

## Questions & Answers

### Q1: Where does the last-quarter calendar-boundary snap get computed — dispatcher, or the schedule-creation path (koch-query-api / K4A)?
- **For:** Backend (dispatcher owner) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Determines whether we add a new `ReportTimeRange` "last calendar quarter" case in dispatcher (`getFromToForTimeRange`, `helpers.go:190-263`) OR inject pre-snapped `time_start`/`time_end` epochs at schedule-creation. Blocks the window-logic estimate. Note: dispatcher's existing `quarterly` math is a coarse ~65/95-day approximation and does NOT snap to true calendar boundaries — so it is NOT reusable as-is for DEC-001.

### Q2: Is DEC-003 (frozen weekly snapshot) actually required for Phase 1? Reviewer declined to decide.
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** If yes → net-new immutable per-week artifact storage + download-consistency modeling, and we have to reconcile it against FR-004 overwrite-in-place idempotency. If no → we can lean on existing "latest link" behavior with far less work. Big schema/effort swing.

### Q3: What is the authoritative server-side source for "standard plan" eligibility?
- **For:** PM / Backend (platform auth owner)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No plan/eligibility code exists in reporting-api, dispatcher, or entitlements-api. entitlements-api (KG's suggested source, low confidence) is app-store subscription products — NOT plan tiers — so it looks like the wrong service. FR-013/NFR-009 (server-side, every request, fail-closed) are un-buildable until a real source is named.

### Q4: What is the authoritative source for "is this user a finance-team member," and does the backend even have a per-user identity on the download request?
- **For:** PM / Backend / Frontend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Backend auth is API-key→account/app scoped today, no user-role concept. FR-010/FR-011 require per-user eligibility re-derived on each request. Need to know what carries user identity to the backend and what store answers "finance team." Blocks access-control design.

### Q5: What satisfies the immutable, append-only report-access audit requirement (FR-017/NFR-012)?
- **For:** Backend / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Need a confirmed store that is truly non-editable/non-deletable by any code path, written async, with alert-on-write-failure. Raptor (entitlements-api) / reporting-logging API exist but I haven't confirmed either meets the immutability bar. Blocks audit design + retention-of-audit-records estimate.

### Q6: Where does PDF rendering land — a new builder inside summary-report-cloud, or a separate render step?
- **For:** Backend (summary-report-cloud owner)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** summary-report-cloud is the content owner and the natural home, but it has no PDF lib today (gofpdf/rsc.io/pdf are only transitive). Decision affects effort and whether we add a dependency vs. a new render service. Also affects where the Brightway logo asset lives (FR-019).

### Q7: Is the downstream data path really summary-report-cloud → BigQuery direct, or does koch-query-api sit in the middle?
- **For:** Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Investigation shows summary-report-cloud queries BigQuery `hourly_rollup` **directly** — it does NOT call koch-query-api (grep clean). The spec/KG say koch-query-api "supplies attribution query data." These disagree. If summary-report-cloud direct-query is the path, koch-query-api may need **no change** (or no involvement) for Phase 1. NFR-003 (rate-cap on shared query source) applies to whichever source generation actually hits — need to confirm which.

### Q8: Where does the config-based kill-switch (NFR-014) live and what's its propagation mechanism across dispatcher + worker + reporting-api?
- **For:** Backend / Infra
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No existing kill-switch found. "Disable within 5 min, no code deploy, in-flight downloads complete under prior behavior" needs a shared config path across three services with different deploy models (dispatcher is deb/systemd on a single box). Additive and cross-cutting.

---

## Rough Scope

Backend work I think we'll touch (by service):

**reporting-dispatcher** (likely light)
- Add/seed a `scheduled` row: `sch_frequency='weekly'`, `sch_type='summary'` for the Brightway app.
- Possibly a new "last calendar quarter" window mode in `getFromToForTimeRange` / `ReportTimeRange` (`helpers.go:190-263`) — OR inject pre-snapped epochs upstream (pending Q1).
- Pass `delivery_format='pdf'` through `ReportStruct` (dispatcher just passes it through — no validation there).

**summary-report-cloud** (heaviest content lift)
- **New PDF render path** — new builder for "one simple table, totals by channel" + Brightway logo. Touches `generateFile` (`main.go:751-780`), format branch in `createFileFromDB` (`sqllite.go:310-347`), no-table short-circuit (`bq.go:303-307`), `normalizeDeliveryFormat`.
- Confirm the by-channel aggregation (`network_id`/`network_name`, `event_revenue`, conversions) maps to the required finance table — looks good, needs shaping to "totals."
- Honor NFR-005 (bounded retry, 3 attempts, `Retry-After`) + NFR-003 (rate cap) on the query source.

**reporting-api** (intake + download)
- Accept `pdf` as a valid `DeliveryFormat` (`api.go:546`, `:1480`).
- Possibly a "last completed calendar quarter" window mode in `calculateStartEndForReport` (`api.go:5284`) if window logic lands here (pending Q1).
- Frozen-snapshot modeling in `kochava_reports.requested`/`.scheduled` if DEC-003 confirmed (pending Q2) — immutable per-week artifact, retention marker.
- Plan-eligibility + finance-user access gate, server-side, fail-closed (FR-010–014, NFR-007/009) — **net-new, no hook exists** (pending Q3/Q4).

**Cross-cutting / net-new**
- Immutable report-access audit (FR-017/NFR-012) — store TBD (Q5).
- 2-year retention — S3 lifecycle policy + DB marker (DEC-006) — infra-adjacent.
- Config-based kill-switch across services (NFR-014) — net-new (Q8).
- Named, distinct failure conditions (NFR-006): timeout / dependency-unavailable / rate-limited / auth-source-unreachable.

**Explicitly OUT (confirmed by investigation, don't build):**
- reporting-subscriptions-worker — subscriptions (LTV/churn) worker, wrong content/format/delivery/data source. Rule out.
- reporting-dispatcher push/email/SFTP delivery path — DEC-002 pull chosen.
- FR-008/FR-009 secure time-limited link — only needed for out-of-session delivery, which is the push path we're NOT building for Phase 1. (Flagging: spec keeps FR-009 "in" conceptually but there's no push in Phase 1, so I'm treating the secure-link work as not-Phase-1 unless told otherwise — see note.)

---

## Dependencies on Other Roles

- **Frontend (frontend-mos):** small/additive per their investigation — surface the new backend template via existing `ReportsList`/`CompletedReports`, gate with a `featureRollout` flag, extend `OutputFormat` to include `pdf`. They need the backend template to expose `pdf` in `output_format` and a `report_url` + "completed" status. We need to agree on the feature-flag key for plan gating (frontend has no "plan" primitive, uses feature flags).
- **PM:** answers on Q2 (snapshot), Q3 (plan source), Q4 (finance-user source) — these three are hard blockers for the access-control + storage design.
- **Platform / Auth owner:** who owns the authoritative plan + user-role source (Q3/Q4). This likely isn't a service I've found yet.
- **Infra:** S3 2-year lifecycle policy (retention), kill-switch config propagation across deb/systemd dispatcher + k8s/Marathon services, and the summary-report-cloud hardcoded-key issue.
- **Design:** Brightway logo asset for the PDF (FR-019) — where it's stored and how the renderer picks it up.

---

## Notes

- Strong reuse story: dispatcher (scheduler) → Pub/Sub `ReportStruct` → summary-report-cloud (generator, already does by-channel revenue/conversions) → reporting-api (`/progress` returns S3 link) → frontend-mos (download list). The pipeline is real; we're mostly adding **PDF, quarter-snap, snapshot/retention, and access control** onto it.
- KG low/medium-confidence calls that the real reads corrected:
  - `koch-query-api → frontend-mos` edge — **does not exist** (grep clean in both repos).
  - `reporting-api → frontend-mos` — **does not exist** (grep clean).
  - `koch-query-api` supplies summary-report-cloud's data — **not via a call**; summary-report-cloud queries BigQuery `hourly_rollup` directly. koch-query-api may be a sibling path, not an upstream. (Q7)
  - `entitlements-api` as plan/eligibility authorization source (KG low confidence) — **looks wrong**; it's app-store subscription products, no plan-tier / user-role model. (Q3)
  - `reporting-subscriptions-worker` + `reporting-dispatcher` push role — confirmed **out of scope** for Phase 1 pull model.
- Key file anchors for later: dispatcher `root.go:1717` (scheduled pull), `root.go:1520-1623` (Pub/Sub publish), `root.go:34-71` (`ReportStruct`); summary-report-cloud `summary_query_builder.go:143,198-204` (by-channel metrics), `sqllite.go:310-347` (format branch); reporting-api `api.go:546/:1480` (format guard), `api.go:5284` (window calc), `01-create-kochava_reports.sql` (schema, no retention col).
- Deploy-topology gotcha: dispatcher is a **single-instance systemd process on reporting02** (restart resets running→queued). A second replica would double-dispatch — so no "make it HA" temptation, and kill-switch has to account for this box specifically.

---

*Captured by `/kochava:eng-context backend` on 2026-11-21*