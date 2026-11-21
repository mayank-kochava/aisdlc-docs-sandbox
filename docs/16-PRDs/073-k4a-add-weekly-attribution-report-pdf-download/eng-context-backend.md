# Engineering Notes: K4A Weekly Attribution Report PDF Download - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-21
**Based on:** product-spec.md + live investigation across 11 repos

---

## Initial Thoughts

Did a real investigation pass across the whole reporting stack. Good news / bad news situation.

**Good news:** the scheduling + dispatch + "list reports" + secure-download-link machinery all already exists. This is NOT greenfield. Weekly cadence is a solved problem in 3 different places (reporting-dispatcher `addScheduledReportThatAreReady`, sam-report-scheduler `repeatType===7`, reporting-api `frequency=weekly`). Time-limited signed download links exist in at least 4 services (koch-query-api, aaas-reporting, reporting-subscriptions-worker, sam-report-scheduler). The spec's self-service download flow maps mostly onto reuse.

**Bad news / the three things that are net-new everywhere I looked:**
1. **PDF output** — grepped every candidate repo. Zero PDF support. CSV/JSON only. The only PDF generator in the whole stack is sam-report-scheduler's `html-pdf` (PhantomJS, deprecated, Slack-only, never persisted). PDF is a genuine build, not a format flag.
2. **"Most recent completed quarter, recalc'd weekly"** — nobody has calendar-quarter window logic. reporting-dispatcher's `weekly` frequency computes *day-based* windows and does NOT snap to quarter boundaries. sam-report-scheduler is rolling-N-days only. aaas-reporting defaults to trailing-1-month. This is new date math wherever it lands.
3. **Finance-team access scope** — every backend auth model is API-key → account/app ownership. There is NO user/role/named-user/"finance team" concept anywhere in the reporting stack. The closest RBAC reference is audit-api's Oathkeeper/Sprinkler gateway + account-ownership-→403 pattern.

**Complexity: HIGH, and I agree with the spec's HIGH-risk call.** The 6-service blast radius is real, and the data-source ambiguity is still unresolved after investigation (see below).

---

## The data-source question — partially resolved, still needs a design-review call

The spec flags a KG discrepancy on the attribution data source. I investigated all the candidates. Here's where I actually landed:

- **pubsuite-attribution** = the attribution *computation / writer*. It writes per-conversion rows to BigQuery `aaas_<SAN>.conversions`. It does NOT aggregate, window, or serve reports. It's the **producer of the underlying data**, not a report backend.
- **koch-query-api** = a **query/export backend** — runs caller-supplied SQL against BQ `app_<AppId>` attribution tables, exports CSV via reporting-api. This is the "two query backends" reading.
- **aaas-reporting** = the other **query backend**, but **SKAdNetwork-only** today (README literally says non-SKAd is "soon"). Probably NOT the source for a general K4A attribution report.
- **reporting-api** = NOT a generator at all — it's validate + persist only. Generation happens in an out-of-repo worker/dispatcher it feeds via `kochava_reports.requested`/`scheduled`.
- **audit-api, alerting-api, mqa** = ruled OUT (audit = audit-log service; alerting = customer-facing marketing alerts with no dispatch path; mqa = transport library).

**My read:** the realistic flow is reporting-dispatcher (or sam-report-scheduler) schedules → worker runs quarter-scoped SQL via koch-query-api against BQ `aaas_<SAN>.conversions` / `app_<AppId>` attribution tables → export. But I can't fully confirm the SQL path because the scheduler→executor wiring crosses repos I couldn't fully trace. **This MUST be settled at design review before build** (NFR-013).

---

## Concerns & Risks

- **PDF rendering is the biggest single build item and it's unowned.** reporting-api/dispatcher/subscriptions-worker all pass format opaquely to a downstream worker. I never found the worker that would render the PDF. Who owns PDF generation? That worker repo wasn't in any checkout I had.
- **Scheduling ownership is split 3 ways** (reporting-dispatcher, reporting-subscriptions-worker, sam-report-scheduler) and they're different stacks (Go daemon on bare host vs. Go worker vs. Node Cloud Function). Picking the wrong one mid-build burns the 5-week deadline.
- **Quarter-window math is net-new and finance-correctness-sensitive (NFR-011).** Getting "most recent completed quarter, recalc weekly" wrong = wrong figures feeding year-end close. This needs explicit, tested date-boundary logic — not a reuse of the day-based `weekly` window.
- **EU data residency (CA-2, pubsuite-attribution):** `conversions` dataset name is `aaas_<SAN>` with NO EU variant, unlike the audit log which splits `aaas`/`aaas_eu`. If Brightway is EU-resident, the read side must independently verify where that dataset physically lives. Finance-data compliance landmine.
- **Operational alerting on failure (FR-004/FR-005, NFR-004) has no in-repo home.** reporting-dispatcher and koch-query-api only emit Influx metrics + flip `sch_status='error'` after 7 tries. No paging/Slack/PagerDuty path found anywhere. The "fire operational alert to owning team" requirement needs a real dispatch mechanism we don't have yet.
- **Finance-team authz is net-new with no backend primitive.** Spec treats it as a working assumption (DEC-002), but even once confirmed, there's nothing to extend — needs a new auth dimension or sourcing from dash-api/sprinkler.
- **Deploy topology for the scheduler is a real fork.** reporting-dispatcher is single systemd/tmux daemon on `reporting02` (single-writer, good — no dup-dispatch risk). But aaas-reporting/pubsuite-attribution are stateless multi-replica (a scheduler bolted on would run duplicated per pod). FR-007 "no overlapping runs" depends heavily on where this lands.
- **Found a secret-in-source:** sam-report-scheduler hardcodes a SendGrid API key (`email-service.js:8,39`). If we touch that repo, flag it. Also koch-query-api signed-link expiry is 864000s (10 days) but comment says "1 day" — real mismatch, matters for NFR-010/FR-013 finance-data sensitivity.
- **sam-report-scheduler has zero tests** (`npm test` is a stub). For finance-grade correctness that's a problem if it becomes the home.

---

## Questions & Answers

### Q1: Where does PDF rendering actually live? Which worker owns it?
- **For:** Backend (design review) / Architecture
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Biggest single build item. reporting-api/dispatcher/subscriptions-worker all pass format opaquely downstream; I never found the PDF-rendering worker in any checkout. Can't scope PDF work (incl. FR-003a logo) until we locate/decide the owner. Blocks ~40% of the estimate.

### Q2: Which scheduler owns the weekly K4A run — reporting-dispatcher, reporting-subscriptions-worker, or sam-report-scheduler?
- **For:** Backend (design review)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** NFR-013 explicitly calls this out as a primary delivery risk. Three different stacks/deploy models. Must be settled before build or scope gets discovered mid-build against the 5-week deadline.

### Q3: Confirm the authoritative attribution data source + the exact SQL/BQ tables for the quarter-scoped query.
- **For:** Backend (design review) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** I narrowed it to "koch-query-api running SQL against BQ `aaas_<SAN>.conversions` / `app_<AppId>` attribution tables," but couldn't verify the scheduler→executor wiring across repos. NFR-011 (exact-match financial figures) can't be guaranteed until the source and query are pinned.

### Q4: What exactly is the "standard K4A attribution report metric set" (FR-003)?
- **For:** PM / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** The raw columns exist (`conversions` table, CA-1 pubsuite-attribution; sam-report-scheduler's goal/ROAS/ARPU columns, CA-9). But "the standard metric set" is an aggregation nobody has defined on the read side. Spec says generate-and-mark-partial (DEC-003), so not fully blocking, but needed to know what to aggregate.

### Q5: Where does the operational-alert-to-owning-team on generation failure get dispatched (FR-004/005, NFR-004)?
- **For:** Backend / Infra
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No paging/Slack/PagerDuty path exists in any reporting repo — only Influx metrics + status flips. Need to know if we build on existing Influx→alert rules or wire a new dispatch. Blocks FR-004/005 and NFR-004.

### Q6: Is Brightway EU-data-resident, and if so where does `aaas_<SAN>.conversions` physically live?
- **For:** PM / Infra
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Dataset naming doesn't encode region (CA-2, pubsuite-attribution). If EU, the read side must target the right dataset or we serve wrong/empty data — a finance-correctness + compliance issue.

### Q7: Where does finance-team access scope get enforced, and against what identity source?
- **For:** PM (DEC-002 owner) / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No backend reporting service has a user/role/team model. Enforcement point (reporting-api? K4A web tier? new gateway check?) and identity source (dash-api/sprinkler?) both undecided. Blocks FR-011/FR-012 server-side enforcement.

### Q8: Where does the "standard plan" eligibility check live (business rule)?
- **For:** PM / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** No plan-tier concept anywhere in the stack (reporting-api has only `api_reporting_access` bool + `status='activated'`; dispatcher selects `account_type` but doesn't gate on it). Needs a decision on where plan eligibility is enforced.

### Q9: Where does the Brightway logo asset come from (account branding record vs. supplied asset)?
- **For:** PM / Design
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-003a. No per-account report-logo mechanism exists in backend or frontend (confirmed absent in frontend-mos too). sam-report-scheduler hardcodes a SAM header image. Need the asset source before PDF layout work.

---

## Rough Scope

Assuming reuse-on-dispatch + new-PDF-worker, roughly:

- **Scheduling config:** add weekly attribution report as a `kochava_reports.scheduled` row (dispatcher side mostly data-config, not code) — *if* reporting-dispatcher is chosen.
- **New window mode:** add "most-recent-completed-quarter" date logic — new `ReportTimeRange` token in reporting-dispatcher (`helpers.go:192-203`) OR new frequency/window mode. Net-new, tested.
- **New report type routing:** `startProcess` in reporting-dispatcher (`root.go:1233-1342`) has no `attribution` type — add a case OR reuse summary/detail with a PDF format.
- **PDF rendering worker:** the big unknown — locate or build. Includes logo/layout (FR-003a). Likely a Puppeteer/headless-Chrome or server render, NOT `html-pdf`.
- **Format enum additions:** add `pdf` to `DeliveryFormat` validation in reporting-api (`api.go:546,1480`) + dispatcher passes it opaquely.
- **Download/list read path:** largely reuse reporting-api `scheduledTokensQuery` (`queries.go:193-217`) + existing signed-link mechanism.
- **Finance-team authz layer:** net-new, probably at reporting-api or K4A web tier — gate list/download server-side, fail-closed (FR-011/012).
- **Standard-plan eligibility check:** net-new gate, location TBD.
- **Audit logging (NFR-009):** emit list/download events — audit-api has the inbound ingestion seam + `pkg/auditclient`; this is the one clean reuse for audit.
- **Operational alerting (FR-004/005):** new dispatch or wire onto Influx metrics.
- **Kill-switch feature flag (NFR-012):** config-based, on-call can flip without deploy.
- **Idempotency/concurrency (FR-006/007):** mostly satisfied by existing single-writer dispatcher + account+week key; verify.

---

## Dependencies on Other Roles

- **Frontend (frontend-mos):** needs backend to emit a `pdf`-format recurring token and surface it via the reports/audiences API. Frontend reuses existing `CompletedReports.vue` + recurring-token path; they need the `OutputFormat` type extended to include `pdf`, and the access-denied state rendered (enforcement is ours). Also needs the finance-team visibility flag (server-driven `available_to_user`-style) decided jointly.
- **Infra:** deploy-topology decision for the scheduler tier (single-replica vs. multi-replica safety for FR-007); alerting pipeline for FR-004/005; EU dataset location for Q6; replica/HPA facts live in `ko-k8s-apps` which I couldn't read.
- **PM:** owns DEC-002 (access scope), metric-set definition (Q4), logo asset source (Q9), plan eligibility (Q8), EU residency (Q6), and the delivery-mechanism email decision (DEC-001/OQ-002 — affects whether subscriptions/dispatch email path is in scope at all).
- **Architecture / design review:** must settle Q1/Q2/Q3 (PDF owner, scheduler owner, data source) BEFORE build — these are the spec's own NFR-013 gate.

---

## Notes

- Key reuse anchors worth bookmarking:
  - Dispatch/scheduling: `reporting-dispatcher/cmd/reporting-dispatcher/root.go:1717-1927` (`addScheduledReportThatAreReady`)
  - List/download read path: `reporting-api/internal/.../queries.go:193-217` (`scheduledTokensQuery`)
  - Signed link TTL pattern: `koch-query-api/cmd/query/results_delivery.go:430` and `aaas-reporting/pkg/reports/gcs-exporter.go:184-202`
  - Audit sink: `audit-api/pkg/auditclient` + `POST /api/v1/audit`
  - Attribution source columns: `pubsuite-attribution/internal/attribution/auditlog.go:337-378`
- Shared constant `report_failure_max=7` links reporting-dispatcher and koch-query-api (confirmed sibling in the flow) — relevant to FR-004 retry/fail behavior + NFR-005 (3-attempt backoff; note existing behavior is 7-try auto-unschedule, different semantics — reconcile).
- reporting-api is Marathon/Mesos, dispatcher is bare-host systemd, sam-report-scheduler is GCP Cloud Function, aaas/pubsuite are k8s — this is a genuinely heterogeneous stack, which is part of the blast-radius risk.
- I could NOT locate the PDF-rendering worker in any checkout — that repo needs to be found and investigated before we can commit to an estimate.

---

*Captured by `/kochava:eng-context backend` on 2026-11-21*