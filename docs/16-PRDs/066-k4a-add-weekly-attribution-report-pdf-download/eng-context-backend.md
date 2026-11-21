# Engineering Notes: K4A Weekly Attribution Report PDF Download - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-20
**Based on:** product-spec.md + Read-verified repo investigation across 9 repos

---

## Initial Thoughts

Good news first: this is mostly an **extension of an existing, mature reporting pipeline**, not greenfield. The whole chain already exists end to end for CSV/JSON reports — weekly scheduling, dispatch, generation, S3 link delivery, status state machine, account-scoped access control. The investigation confirmed the attribution data we need (channel/installs/revenue) already lives in a BigQuery table (`summary-report-cloud`'s `network_name`/`install_count`/`event_revenue`/`install_spend`), and the summary query already produces it.

So the feature really boils down to a handful of concrete seams:
1. **PDF as a new output format** — genuinely net-new, doesn't exist anywhere. Threaded through ~10 branch sites in `summary-report-cloud`, plus format allow-list in `reporting-api`.
2. **Weekly schedule row** firing Monday 9am account-local time — scheduling exists, but hour-of-day for weekly is NOT a first-class concept.
3. **7-day secure link** — links exist but hardcoded to 10 days, and there's **no regenerate-from-UI flow anywhere**.
4. **Audit on download** — `audit-api` can ingest it as-is, no change needed to record.

Complexity feels **medium**, concentrated in two repos (`summary-report-cloud` for PDF rendering, `reporting-api` for schedule/format plumbing). The HIGH-risk label is more about the 5-week deadline + blast-radius uncertainty than deep technical complexity.

**The blast-radius ambiguity is now largely resolved by investigation** — see below. That's the single biggest de-risk from this pass.

---

## Concerns & Risks

- **Service ownership overlap, now mostly settled.** The spec flagged scheduling (dispatcher vs subscriptions-worker) and PDF-gen (reporting-api vs subscriptions-worker) ownership as unsettled. Investigation resolves it:
  - **Scheduling/firing** = `reporting-dispatcher` (reads `sch_next_schedule_dttm`, inserts into `requested`). Confirmed.
  - **Generation/rendering** = `summary-report-cloud` (Pub/Sub consumer, runs the summary query, builds the artifact). This is where the summary attribution query + the richest columns live. `reporting-subscriptions-worker` is a *different* worker producing subscriptions/entitlement reports — NOT attribution. I think the KG mislabeled it ("Weekly scheduling and PDF generation"). **It is the wrong service for this feature.**
  - **`reporting-api`** = request/schedule intake + status/link read. Medium-touch.
- **`k4a-mcp-server` / `kochava-k4a-mcp-server` / the "entitlement service" are RULED OUT.** The disputed 6th service (entitlement/plan-check) genuinely does not exist as a plan gate in any repo I looked at. OQ-004 (not gated) is confirmed by code: no plan-tier enforcement anywhere; the only entitlement-adjacent code is MMM-only exclusion in the MCP server (irrelevant + fails open). `kochava-k4a-mcp-server` is just a registry/docs repo — no code. So the "6 vs 8 service" confusion resolves to: entitlement service is a phantom; the two low-confidence extras (`koch-query-api`, `summary-report-cloud`) are REAL and important.
- **Monday 9am account-local SLA vs single-instance dispatcher.** `reporting-dispatcher` runs on **one hand-deployed box (`reporting02`), often in a tmux session, no HA/failover**. For a *hard fixed-time SLA* this is a real operational risk — if that box is down Monday morning, no report. Worth surfacing loudly.
- **Weekly `run_on` encodes day-of-week, not hour.** "Monday 9:00am" isn't natively expressible for weekly frequency — the 9am is baked into the schedule row's datetime, not a config field. Needs a small design decision.
- **Per-period idempotency (FR-012 / spec flow node N) does NOT exist.** Dispatcher advances `sch_next_schedule_dttm` so the same period isn't re-selected, but there's **no "already generated for this period → upsert/overwrite"** guard, and on restart in-flight jobs reset to `queued`. True period-level idempotency would be new work in dispatcher/reporting-api.
- **Hardcoded live-looking AWS + SendGrid credentials in `summary-report-cloud` source.** Given this is finance/billing-sensitive data, that's a real security finding (predates the feature, but we're touching this repo).
- **PDF compression carve-out.** `summary-report-cloud` / subscriptions-worker gzip files over a size limit — would produce `.pdf.gz` and break the download UX. Need a carve-out for the PDF type.
- **Link-expiry constant/comment mismatches everywhere** (864000s = 10 days, but comments claim "1 day" / "10 days" inconsistently in both `koch-query-api` and `summary-report-cloud`). Need to reconcile to 7 days and get the comments right.
- **Spend data source.** `pubsuite-attribution` (raw conversions) has installs + revenue but **no spend at all**. Spend comes from the cost/geo table (`install_spend` / `total_spend_on_traffic_selected`) that the summary query already joins — so I *think* spend is covered via the summary path, but want to double-check the summary query's spend column is populated for Brightway's traffic.

---

## Questions & Answers

### Q1: Is PDF rendering owned by `summary-report-cloud`, and do we reuse the existing summary (network-level) query, or the newer aggregated-report builder?
- **For:** Backend (self/team, in review) + Architect
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Biggest single build decision. The legacy `bq.go`/`summary_query_builder.go` path has the richest attribution columns; the `aggregated-report/` package is cleaner and better-tested. Determines where the new PDF `ReportBuilder` lands and how much PDF plumbing we thread through branch sites. Also need to pick/vet a Go PDF library (none in go.mod today).

### Q2: How do we express "Monday 9:00am account-local time" for a weekly schedule when `run_on` is day-only?
- **For:** Backend + PM (to confirm DST/timezone basis — CG-004 was deferred)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Blocks NFR-004 (fixed availability deadline). Options: new hour-of-day field on `scheduled`, or encode 9am into `sch_next_schedule_dttm` and let the dispatcher's TZ conversion handle it. The dispatcher already does `CONVERT_TZ(..., 'UTC', timeZone)` so account-local is feasible — but the DST basis for the deadline is still an open product question.

### Q3: Does the single-instance, manually-managed `reporting-dispatcher` meet a hard Monday-9am SLA, or do we need HA/alerting before relying on it?
- **For:** Infra + Backend + PM (risk acceptance)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** NFR-004 reliability + NFR-017 rollback story. If the one box dies Monday morning the report silently doesn't generate. May need a "schedule didn't fire" alert at minimum, even if full HA is out of scope for MVP.

### Q4: Do we need true per-period idempotent upsert (FR-012), or is the existing "advance next-schedule-time" behavior acceptable for MVP?
- **For:** PM + Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-012 / spec flow node N claims upsert-on-retry; it doesn't exist. Building real period-level dedup on the `requested` table is non-trivial new work in dispatcher/reporting-api. Need to know if the implicit scheduling guard is "good enough" for the deadline.

### Q5: Where does the 7-day secure-link **regeneration from UI** (FR-005a) get implemented — new endpoint in `koch-query-api`/`reporting-api`, or re-trigger of the generation worker?
- **For:** Backend + Frontend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-005a / FR-006 / NFR-012. No regeneration endpoint exists anywhere today; links are generated once and stored. This is genuinely net-new surface. Frontend confirmed they need a backend endpoint to call. Need to decide which service owns re-deriving authz server-side and re-issuing the presigned link.

### Q6: Confirm spend data is actually populated in the summary query's cost columns for Brightway's traffic.
- **For:** Backend + Data
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-003 "spend per channel." Spend isn't in attribution conversions; it comes via the cost/geo join in the summary query (`install_spend` / `total_spend_on_traffic_selected`). Want to verify it's non-empty for this account before committing to the summary path.

### Q7: FR-014 wants distinct failure statuses (timeout vs dependency-unavailable vs not-ready). Current status models are coarse (free-text detail or a single `error`). Add new status enum values?
- **For:** Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-014 / NFR-016. `summary-report-cloud` has distinct failure *strings* but the `requested` status model collapses to error/completed. Distinguishing the three conditions the spec wants needs schema/status additions across dispatcher + worker + reporting-api.

---

## Rough Scope

What I think we'll touch (backend):

- **`summary-report-cloud`** (highest touch): new PDF `ReportBuilder` / writer branch; add `"pdf"` arm to the ~10 `DeliveryFormat` branch sites; PDF compression carve-out; S3 ContentType; reuse existing summary attribution query (channel/installs/revenue/spend). Vet + add a Go PDF lib. Clean up hardcoded creds.
- **`reporting-api`**: add `pdf` to the csv/json delivery-format allow-list (ad-hoc + scheduled); weekly-schedule plumbing + possible hour-of-day field; reuse `/summary` + `/schedule/summary` + existing access-control middleware. Possibly host the link-regeneration endpoint.
- **`reporting-dispatcher`**: register/route a new weekly attribution report type (new `req_type` + worker mapping, OR reuse summary worker w/ `delivery_format=pdf`); weekly `scheduled` row for Brightway; possibly the "schedule didn't fire" alert for SLA. Reconcile the `glDelayForSceduled` comment/value mismatch while here.
- **`koch-query-api`**: 7-day link TTL (change 864000 → 7 days), reconcile comment; possibly the new UI-facing authenticated download/link-reissue endpoint lives here.
- **`audit-api`**: likely **no code change** to record downloads (free-form ingest). *Maybe* one line adding `"download"` to `ValidActions` IF we need the events queryable by action — decide in review.
- **Kill-switch (FR-017)**: config-based disable. Dispatcher has an `inMaintenance` flag but it's hardcoded `false` with no runtime toggle — needs a real config lever. Nav-level `featureRollout` exists on frontend for rollout gating.
- **Enrollment**: operationally configure Brightway's weekly `scheduled` row (no self-service UI in MVP).

Explicitly **NOT touching**: `reporting-subscriptions-worker` (wrong report domain — subscriptions/entitlement), `k4a-mcp-server`, `kochava-k4a-mcp-server`, `pubsuite-attribution` (data producer only, no spend anyway), any entitlement/plan-check service (doesn't exist).

---

## Dependencies on Other Roles

- **Frontend (`frontend-mos`):** needs us to expose `output_format.pdf: true` on the template (then their radio auto-renders), and a backend endpoint for link regeneration (FR-005a). Download UX + status chips already exist — low frontend lift.
- **Data:** confirm the summary query's spend/cost columns are populated for Brightway's traffic (Q6); confirm the attribution BigQuery dataset is the right source.
- **Infra:** weigh in on the single-box dispatcher SLA risk (Q3) and the kill-switch config propagation mechanism; replica/HA topology lives in `ko-k8s-apps` which I can't see from the service repos.
- **PM:** resolve the deferred DST/timezone basis (CG-004) for the Monday-9am deadline; accept or reject the single-instance SLA risk; decide MVP idempotency bar (Q4).
- **Architect:** PDF ownership + query-path decision (Q1); link-regeneration ownership (Q5).

---

## Notes

- Strong reuse story overall — the spec's "no self-service export exists today" framing is partially inaccurate for the backend: scheduled CSV/JSON attribution-by-network reports already exist. The genuinely new pieces are **PDF format + weekly finance packaging + UI link regeneration + fixed-time SLA**.
- Cross-service failure contract already exists: `report_failure_max = 7` duplicated/coordinated between `reporting-dispatcher` and `koch-query-api`. Keep that in mind for resilience work.
- `reporting-api` → `frontend-mos` outbound call (a KG hint) is **false** — the seam is inbound (frontend calls reporting-api). Drop it from the flow graph.
- Observability patterns to follow: InfluxDB metrics + heartbeat staleness in dispatcher; OpenTelemetry + error classification in `koch-query-api`; jittered-retry status updates in `summary-report-cloud`.
- Deploy topology caveat: dispatcher + workers are scp'd Debian binaries on `reporting-01/02.aerospike.internal`, not k8s. SOPS-encrypted configs. Any new flags (PDF path, 7-day TTL) go in those SOPS configs.

---

*Captured by `/kochava:eng-context backend` on 2026-11-20*