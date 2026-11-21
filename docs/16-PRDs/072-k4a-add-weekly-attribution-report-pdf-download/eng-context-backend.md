# Engineering Notes: Weekly Brightway Attribution Report (K4A) - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-21
**Based on:** product-spec.md (Type A, confirmed) + Read-verified repo investigation across 10 candidate services

---

## Initial Thoughts

Did a real pass across all the service-map candidates. Good news: this is **mostly reuse, not greenfield** — there's already a complete report request/schedule/dispatch/generate/deliver pipeline in the Kochava stack. The spec's "service-map disagreement" resolves pretty cleanly now that I've actually read the code:

**The pipeline, end to end (confirmed seams):**
- `reporting-api` (Go) — request/scheduling **front door**. Validates requests, writes `kochava_reports.requested` (ad-hoc) / `kochava_reports.scheduled` (recurring). Already supports `weekly` + `RunOn` day-of-week. Does NOT generate or render anything.
- `reporting-dispatcher` (Go, **single-host tmux, not k8s**) — the actual weekly cron brain. Polls `scheduled`, promotes due rows into `requested`, dispatches to workers via Pub/Sub. `weekly` cadence already works here.
- `summary-report-cloud` (Go, Pub/Sub worker) — the **generation/delivery tail**. BigQuery → CSV/JSON → S3/SFTP → status POST back to reporting-api. Has an "aggregate" pipeline keyed by account_id with `network_name` (channel), conversions, and spend building blocks. **This is the closest fit for our report content.**
- `pubsuite-attribution` (Go) — **write-side data producer only**. Streams conversion rows to `aaas_<SAN>.conversions` BQ. This is where our source data originates. Ruled OUT of report generation.
- `mos-iam-server` (Go) — IAM/authz provider. Has `org_feature_mapping` per-org feature flags (clean BR-4 lever) and account-member access checks (BR-5). Ruled OUT of generation.
- `frontend-mos` (Vue) — K4A console, already has a Reports feature w/ Attribution tab, download via signed URL.

**Ruled fully OUT** (name/hostname matches, not real pipeline code):
- `kochava-k4a-mcp-server` — **docs/registry repo, zero source files**. Dead end.
- `k4a-mcp-server` — FastMCP tool proxy for LLM agents. No scheduler, no PDF, no generation. Not us.
- `aaas-reporting` — SKAdNetwork-focused reporting backend. A *possible* alt home but summary-report-cloud is the better-fit existing pipeline; aaas-reporting is more siloed to SKAd. Flagging as a reconcile-to-not-use unless there's a reason.
- `reporting-subscriptions-worker` — subscriptions (cancelguard/profile) worker, not attribution. Useful only as a pattern sibling.

Complexity-wise: medium. The scheduling/cadence/dispatch/delivery plumbing all exists. The genuinely-new backend work is narrow and well-bounded (see scope), but two of those — **PDF rendering** and **"last quarter" window** — are net-new everywhere I looked.

---

## Concerns & Risks

- **PDF is net-new in EVERY repo.** Grepped `reporting-api`, `reporting-dispatcher`, `summary-report-cloud`, `aaas-reporting`, `reporting-subscriptions-worker` — zero PDF libs, zero renderers. All existing workers stream row-at-a-time to CSV/JSON writers, which does **not** map to paginated/tabular PDF layout. This is the single biggest new build and it needs a PDF library choice + vetting + layout work + the Brightway logo (FR-019). Likely lands in `summary-report-cloud` as a new `delivery_format="pdf"` branch + renderer.
- **"Last quarter" window (BR-3) doesn't exist as a concept.** In `reporting-dispatcher` the window is coupled to frequency — a *weekly* cadence produces a *weekly* window, and the `timerange` override enum has no "last quarter" value. So "weekly cadence + last-completed-quarter window" is genuinely unsupported today. Cleanest fix is a new `ReportTimeRange` value + `getFromToForTimeRange` case in dispatcher's `helpers.go` (decouples window from cadence). Must recalc the completed-quarter boundary each week.
- **"Partial data" marking (BR-7/FR-013) has a trap.** Every worker has a `_PARTIAL` file-rename mechanism — but it triggers on **row-quota overflow**, NOT on data-readiness ("some attribution data still processing"). Semantically wrong for our case. We need a real data-readiness signal (an attribution watermark) that I did **not** find consumed anywhere. Reusable naming convention, net-new trigger semantics + source signal.
- **No "fire operational alert" primitive for no-data (FR-014 / flow box E).** Dispatcher records failures to `sch_failed` and emits InfluxDB metrics, but there's no PagerDuty/Slack dispatch in-repo. Alerting on "no data at all" would be built on metrics downstream or added new.
- **403 isn't the dominant pattern in `reporting-api`.** That service overwhelmingly signals errors as HTTP 200 + `{"status":"Error"}` body; true 401/403 appear only in a couple handlers. The spec's flow + FR-012/FR-016 mandate a real 403. `aaas-reporting` and `mos-iam-server` DO have clean 403 paths. Need to decide where auth/download serving lives and make sure it returns a real 403.
- **Account gating (BR-4) doesn't exist as "gate feature to one account."** `reporting-dispatcher` has hardcoded account-ID routing (386, 2270) but no plan/feature gate. `reporting-api` AppCheckerMiddleware gates on activated + reporting-access only. The *right* lever is `mos-iam-server`'s `org_feature_mapping` flag on Brightway's org — but that means a cross-service call to read the flag and enforce it server-side. Net-new wiring.
- **Where does the end-user PDF download endpoint actually live?** This is the gap I'm least sure about. `reporting-api` surfaces `req_report_link` (S3 link) via status endpoints but I did NOT confirm a dedicated "download the weekly PDF" handler. `frontend-mos` just renders `report_url` as an href. The secure, time-limited link + server-side re-auth on download (FR-010/FR-012/FR-015) needs a confirmed owner — likely reporting-api, but unconfirmed.
- **Dispatcher single-replica is load-bearing.** On startup it resets ALL running rows back to queued — a second concurrent instance stomps in-flight jobs. Deliberately single-host tmux, not k8s. Relevant to NFR-004/NFR-005 (idempotency/overlap) and to anyone proposing a new always-on component.
- **Revenue + channel + prior-week delta aren't fully in the data.** `pubsuite-attribution` writes `partner`, `attribution_settings`, `event_metadata` — but **no explicit "revenue" or "channel" column**. `summary-report-cloud` has `network_name`/conversions/spend but **no "attributed revenue" metric and no week-over-week delta** (single-window queries only). So report content needs: map "revenue"/"channel" onto the schema (derive from partner/metadata?) + add a two-window comparison query.
- **Multi-service blast radius is real** even with reuse — touching reporting-api + dispatcher + summary-report-cloud + an auth seam + frontend. Kill-switch (FR-017/NFR-015) is critical. Note none of these Go workers has a built-in config kill-switch I found; that's net-new too.
- **Hard year-end deadline** + several open product decisions still gating buildable requirements. PDF + window are the long poles and don't depend on the open decisions, so we can start those in parallel.

---

## Questions & Answers

### Q1: Where does the end-user PDF **download** endpoint + secure time-limited link + per-request re-auth (FR-010/FR-012/FR-015) actually live?
- **For:** Backend (architect) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks locking the auth seam and the secure-download design (NFR-008/009/010/011). Couldn't confirm a dedicated download handler in reporting-api — it surfaces S3 links via status endpoints but no clear "download the weekly PDF" route. Need to know if we extend reporting-api or build new.

### Q2: Which repo owns **PDF rendering** — `summary-report-cloud` (my recommendation, best-fit existing aggregate pipeline) or `aaas-reporting`?
- **For:** Backend (architect)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Biggest net-new build. summary-report-cloud already has account-keyed aggregate + channel/conversions/spend building blocks; aaas-reporting is SKAd-siloed. Need this settled before estimating PDF + logo (FR-019) work.

### Q3: What is the **data-readiness / "is this window final?"** signal that drives BR-7/FR-013 partial marking?
- **For:** Backend / Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No worker consumes a readiness/watermark signal today; the existing `_PARTIAL` mechanism is row-quota based (wrong semantics). Without a real "data still processing" signal we can't correctly implement "generate against final data + mark partial." Need to know if pubsuite-attribution / the ingestion pipeline exposes a watermark.

### Q4: How do "**attributed revenue**" and "**channel**" map onto the actual attribution data?
- **For:** Data / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** `pubsuite-attribution` writes no explicit revenue/channel column. Is "channel" = `partner`/`network_name`? Is "revenue" derived from `event_metadata`, or sourced elsewhere? Report content (FR-006) can't be built until this is nailed down.

### Q5: Confirm **account gating** (BR-4) is enforced via `mos-iam-server`'s `org_feature_mapping` flag on Brightway's org, read server-side on each request (FR-016)?
- **For:** Backend (architect) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** This is the clean, idiomatic lever (no hardcoded account IDs). But it means a cross-service authz call on every generation/download request + a fail-closed path (NFR-010) if IAM is unreachable. Need sign-off on this approach before wiring.

### Q6: FR-002 says **system-triggered** weekly generation, but BR-6 / resolved OQ-003 say finance **manually triggers from the console**. Which is it?
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Both models already exist in the stack (dispatcher cron scheduling a `scheduled` row, vs. console-initiated ad-hoc `requested` row). They lead to different wiring. Spec is internally inconsistent here — need a definitive pick before building the trigger path.

### Q7: What's the intended **kill-switch** mechanism (FR-017/NFR-015) across these Go services?
- **For:** Backend (architect) / Infra
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** None of the workers I read has a built-in config-driven feature disable. Need a config/flag mechanism that reflects within 5 min with no deploy, usable by on-call. Could piggyback on the IAM feature flag (Q5) — flip the org flag off = feature disabled. Worth considering as the single kill-switch.

### Q8: Is `aaas-reporting` in or out of scope for this feature?
- **For:** Backend (architect)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** It's a real reporting backend but SKAd-focused with its own report→BQ→GCS pipeline and its own static-API-key auth (no account concept). My read is OUT (summary-report-cloud is the better fit), but the KG flagged it low-confidence to reconcile. Confirming OUT shrinks blast radius.

---

## Rough Scope

Assuming summary-report-cloud + reporting-api + dispatcher + an IAM auth seam:

- **`reporting-dispatcher`:** add a "last-quarter" `ReportTimeRange` value + `getFromToForTimeRange` case (BR-3), decoupling window from the weekly cadence. Otherwise cadence reuse — create a `scheduled` row with `sch_frequency='weekly'`. _(Likely the single concrete change here.)_
- **`summary-report-cloud`:** new `delivery_format="pdf"` branch + a PDF renderer (new lib) + Brightway logo (FR-019); new/extended aggregate query for attributed-revenue + conversions by channel + **prior-week delta** (second-window comparison); real data-readiness partial-marking (vs. the row-quota `_PARTIAL`).
- **`reporting-api`:** accept new `pdf` delivery_format value; add server-side **account eligibility gate** (BR-4) in AppCheckerMiddleware; likely home for the **secure time-limited download** endpoint + real **403** on auth failure (deviates from its 200+body convention — call out).
- **Auth seam (`mos-iam-server`):** set Brightway org `org_feature_mapping` flag; consumers read it server-side per request; fail-closed on IAM unreachable (NFR-010).
- **Alerting:** new "no data" operational alert dispatch (flow box E / FR-014) — dispatcher only does Influx metrics today.
- **Kill-switch** config mechanism (FR-017) — net-new; consider reusing the IAM flag.
- **Idempotency/overlap:** mostly inherited (dispatcher single-replica + account+window keying) — confirm `requested`/`scheduled` keying gives us FR-004/FR-005 for free.
- **Audit trail** (NFR-013) — append-only generate/download records; confirm whether an existing audit store covers this or it's new.
- **No sensitive data in logs** (FR-018/NFR-012) — review log lines across touched services.

---

## Dependencies on Other Roles

- **Data:** the data-readiness/watermark signal (Q3) and the revenue/channel mapping (Q4) — I'm blocked on correct partial-marking and report content until these land.
- **Frontend (`frontend-mos`):** needs a "partial data" status/badge (none today — CA-5 in their findings) and an explicit "no report available" empty state (also missing). Also a backend token field exposing the partial flag. PDF auto-renders once the backend template declares `output_format.pdf` — no frontend format-code change needed. Coordinate the signed-URL download href + 403 UX.
- **Infra:** deployment topology for dispatcher (single-host tmux — is that staying?) and any new always-on component; kill-switch config delivery (Q7); alerting target for FR-014.
- **PM:** the FR-002-vs-BR-6 trigger contradiction (Q6), account-gating approach sign-off (Q5), and the still-open assumptions (DEC-002/003/004/005/007/008) that gate acceptance.
- **Architect:** settle the service-map (PDF owner Q2, download owner Q1, aaas-reporting in/out Q8) before scope locks.

---

## Notes

- Shared `kochava_reports.requested` / `kochava_reports.scheduled` tables (MySQL/MariaDB `db-global.kochava.net:3308`) are the central contract. Account is **not** a column on `scheduled` — it's implicit via the api_key embedded in the `sch_request` JSON blob.
- reporting-api owns the `RequestPost` request schema (shared Go module used by the workers) — schema changes ripple.
- Pub/Sub prod topics: `ko-p-restmeas-pst-summary-report-requests-1-3` (summary) / `ko-p-restmeas-pst-detail-report-requests-1-4` (detail).
- frontend-mos reporting traffic → `reporting.api.kochava.com/v1.4` via shared `ReportsApi` client, `Authentication-Key` + `withCredentials`.
- NFR-017 (backward compat): must confirm no existing scheduled job / report output / attribution-source query path changes for current callers — all our changes should be additive (new delivery_format value, new ReportTimeRange value, new report template branch).
- Rejected alternative: there's a `window.print()` client-side PDF in frontend-mos (SoW export) — does NOT fit (we need server-generated downloadable snapshot). Noting so eng-plan doesn't rediscover it.

---

*Captured by `/kochava:eng-context backend` on 2026-11-21*