# Implementation Plan: Weekly Last-Quarter Attribution PDF

**PRD**: [product-spec.md](./product-spec.md)
**Date**: 2026-11-22
**Research**: [eng-research.md](./eng-research.md)
**Status**: Draft

## Summary

Deliver a self-service, on-demand downloadable PDF of last-quarter attribution data (total attributed revenue and conversions by channel) for Brightway Retail finance from the K4A console. The feature is overwhelmingly an **extension of the existing generate → poll → download report pipeline** (submit via `reporting-api`, generate in a BigQuery worker, deliver via signed URL, poll/download in `frontend-mos`), with net-new work concentrated in three backend areas: **PDF rendering + Brightway logo embedding**, **calendar-quarter window math**, and **attributed-revenue sourcing** (the single highest-risk unknown).

---

## Technical Context

**Language/Version**: Go (worker layer: Go 1.23–1.26 depending on repo; `summary-report-cloud`/`aaas-reporting` are the candidate generation homes), TypeScript/Vue 3 (frontend-mos).
**Primary Dependencies**: Existing report pipeline (`reporting-api` v1.4 queue, `reporting-dispatcher` passthrough, BigQuery worker), GCS/S3 signed URLs, `mos-iam-server` for org logo + feature flag. New: a Go PDF-rendering library (working default `maroto`, subject to Go-dependency approval).
**Storage**: Shared MySQL `kochava_reports.requested`/`.scheduled` (report state, no DDL needed — params ride in `req_request` JSON), BigQuery attribution datasets (read), GCS/S3 (PDF artifact + signed URL).
**Testing**: Go `testing` + testify for net-new worker code; existing report-pipeline integration harness; frontend-mos existing Vitest setup.
**Target Platform**: Kubernetes (cloud workers via `ko-k8s-apps`), Marathon/Mesos (`reporting-api`), bare-metal `reporting02` (dispatcher, single-instance).
**Performance Goals**: NFR-001 download p95 ≤ 3s (serving pre-generated file only — no synchronous generation on download path); NFR-002 read-rate cap on attribution source; generation latency of 1–2 min acceptable (OQ-008 resolved).
**Constraints**: 5-week hard deadline (year-end close); NFR-010 backward-compat (existing CSV/JSON consumers of shared services must not regress; `/v1` contract additive only); NFR-007 no sensitive data in logs.

---

## Standards Check

### Go Style Guide
- [x] Follows `/kochava:go-style-guide` patterns (net-new worker code follows existing package conventions in chosen generation home)
- [x] Error handling consistent with standards (distinct named failure statuses per FR-007/NFR-004)
- [ ] Package structure follows conventions — **NOTE**: candidate home `aaas-reporting` has thin test coverage on handlers/builders; net-new code lands in largely untested packages (Phase 0 addresses this).

### Testing Standards
- [x] Test strategy covers unit/integration needs (quarter math, revenue/channel query, PDF layout as units; pipeline flow as integration)
- [x] Uses approved frameworks (testify)
- [x] Coverage targets defined (net-new code: unit coverage mandatory; Phase 0 backfills any modified existing code)

### Dependency Standards
- [ ] All dependencies on approved list — **PDF library (`maroto`/`gofpdf`/HTML→PDF) is net-new and NOT yet approved.** Must pass `/kochava:go-dependency-analysis`. [DECISION REQUIRED]
- [x] Dependency versions pinned
- [x] No new frontend dependency (confirmed: PDF rendering is server-side; `frontend-mos` change is one optional type field)

### Documentation
- [x] CLAUDE.md updates planned (chosen generation home + optional `docs/4-Guides/Reporting/reporting-testing-guide.md` note that `pdf` is now a supported format)
- [x] API documentation planned (no new public endpoint; `delivery_format` enum value `pdf` is additive to existing contract)

**Standards Status**: PASS WITH NOTES (new PDF dependency requires approval; generation-home test coverage gap flagged).

---

## Specification Validation

*Summary of findings from eng-research.md and live investigation (investigation-verified = Read-confirmed in a live checkout).*

### Confirmed Assumptions

| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| On-demand generate→poll→download UX with 1–2 min latency (BR-1, OQ-008) | **Investigation-verified** — `frontend-mos` CompletedReports.vue:39-95, reporting.ts:659-712; `reporting-api` /v1.4/progress api.go:4252; `aaas-reporting` skad.go:43-302 | Entire UX ships for CSV/JSON; reuse directly. |
| Time-limited secure link, not public URL (FR-011/NFR-006) | **Investigation-verified** — `aaas-reporting` gcs-exporter.go:184-202 (V4 signed URL on complete); `summary-report-cloud` main.go:628-700 (S3link) | Set ~15min TTL default (NFR-006), tunable. `aaas-reporting` default is 7 days (gcs.go:30-36) — must override. |
| K4A download button is the delivery mechanism (OQ-002, BR-4) | **Investigation-verified** — `frontend-mos` CompletedReports.vue download `<v-btn :href>`; format path is data-driven end-to-end | Frontend change is a single optional type field (`pdf?: boolean` on `OutputFormat`, ReportingInterface.ts:114). |
| Brightway logo is an existing managed org asset (BR-3 branding) | **Investigation-verified** — `mos-iam-server` logoservice.go:135-196, `logo_url` first-class org column (orgmgmt.go:14), exposed on org-GET | PNG/JPEG only, no SVG. Renderer pulls from org record / `/logos/download`. |
| Report params need no new schema (format rides existing fields) | **Investigation-verified** — `reporting-api` req_request JSON blob (queries.go:18-24); `reporting-dispatcher` req_type varchar(25), format passthrough (00-init.sql:60, root.go:45); `summary-report-cloud` reportStruct DeliveryFormat (main.go:57-92) | No DDL anywhere in the pipeline. `delivery_format:"pdf"` is just a new string value. |
| Conversions-by-channel data shape exists | **Investigation-verified** — `summary-report-cloud` summary_query_builder.go:142-203 (network grouping + `event_revenue`); `aaas-reporting` reports.go:45-52 (conversions by network, no revenue) | Revenue is the gap (see Corrected). |

### Corrected Assumptions

| Spec Statement | Actual Reality | Impact |
|----------------|----------------|--------|
| "Deliver as PDF" is a format toggle | **Investigation-verified: no PDF anywhere** — all report services CSV/JSON only (reporting-api api.go:546; summary-report-cloud delivery_format.go:3-9, aggregated_insights_report_builder.go:161-166; aaas-reporting reports.go:34-40). No PDF lib in any go.mod. | PDF rendering + logo embedding is the single biggest net-new build, in the worker layer. |
| "Last quarter = most recent completed calendar quarter" is supported | **Investigation-verified: no calendar-quarter logic anywhere** — reporting-api window is rolling (api.go:5330-5365); dispatcher `quarterly` is a buggy ~65-day approximation (root.go:1996-2004); no `last quarter` named range (helpers.go:190-203); workers take pre-computed epochs. | Quarter math is net-new; inject explicit `time_start`/`time_end` epochs at generation time (upstream request builder). Avoids dispatcher's broken approximation → zero dispatcher change (reporting-dispatcher CA-18). |
| Report shows "attributed **revenue** by channel" | **Investigation-verified, nuanced:** `pubsuite-attribution.conversions` has no `revenue` column, BUT `event_metadata` JSON carries `price`/`sum`/`currency`/`item_quantity` (request.go:25-64, auditlog.go:244-357). `summary-report-cloud` has first-class `event_revenue` (summary_query_builder.go:142-203). `aaas-reporting` has none. "channel" has no column — candidates `partner` (first-class, auditlog.go:344), `network`, `claim_traffic_type`. | **Highest-risk gap.** Revenue definition + source AND channel mapping must be resolved before build. Likely path: extract from existing `event_metadata.price`/`.sum` (no schema change) OR use `summary-report-cloud.event_revenue` if it covers Brightway's dataset. A new validated column won't backfill last quarter. See [DECISION REQUIRED]. |
| Authorization denial is a "server-side 403" (FR-010/§2 flow) | **Investigation-verified: no clean 403 on the K4A path** — `reporting-api` returns 200+`{status:"Error"}` (appCheckerMiddleware.go:100-120); `mos-iam-server` returns 401 (logoservice.go:174); only `aaas-reporting` returns 403 (respond.go:35-60). | Matching "403" literally on the reporting-api path is a behavior change with existing-client risk. Reconcile whether "server-side denial" (any form) satisfies FR-010. See [DECISION REQUIRED]. |
| "Standard-plan account" is a plan/tier check (BR-5) | **Investigation-verified: no service models plan tiers.** `pricing_plans.tier` holds account *classification* (`"aa"`), not subscription plans (reporting-api api.go:3180). Gating mechanisms are `feature_rollout` whitelist (reporting-api queries.go:438-442), `org_feature_mapping` (mos-iam-server orgmgmt.go:204-214), or `APIReportingAccess`. | BR-5 is satisfied by default (no tier gate blocks a standard-plan account). If the feature must be *restricted* to Brightway, use a `feature_rollout` row or `org_feature_mapping` flag — **data, not schema**. See [DECISION REQUIRED]. |
| `pubsuite-attribution` is the authoritative read source | **Investigation-verified: it is write-only to BQ** (runner.go:135-185); never reads back. KG did not re-confirm at seed confidence. The read side is built in a worker. | Verify authoritative read source during design (resolved to: the generation worker reads BigQuery `aaas_<SAN>.conversions` or `summary-report-cloud`'s dataset). |
| FR-006 incomplete-week data-quality warning has an existing equivalent | **Investigation-verified: none.** `summary-report-cloud` `_PARTIAL` means row-cap-exceeded, unrelated. | Net-new logic, and net-new in a PDF that doesn't exist yet. Also in tension with "most recent *completed* quarter." Confirm Phase-1 scope. See [DECISION REQUIRED]. |
| FR-004/FR-005 (idempotent, non-overlapping) apply under on-demand | **Investigation-verified:** these only make sense under the **scheduled** model (week+window key, cycles). Under pure on-demand there is no "week" key. | Their design is gated on the on-demand-vs-weekly ruling (resolved on-demand per DEC-001/DEC-007, so FR-004/FR-005 are effectively N/A for Phase 1 — see [DECISION REQUIRED] on cadence). |

---

## Project Structure

### Documentation (PRD Folder)

```text
docs/16-PRDs/NNN-weekly-last-quarter-attribution-pdf/
├── product-spec.md
├── eng-context-backend.md
├── eng-research.md
├── eng-plan.md          # This document
├── data-model.md        # Generated by this skill
└── contracts/           # (none — no new API surface; see note)
```

### Source Code Changes

```text
# Generation home (DECISION REQUIRED: aaas-reporting vs summary-report-cloud)
summary-report-cloud/  (fallback home — has event_revenue + clean format switch)
├── go.mod                                              [MODIFY] add PDF lib
├── report_runner.go                                   [MODIFY] route pdf template (:18-54)
├── delivery_format.go                                 [MODIFY] add "pdf" to allow-list (:3-9)
├── aggregated-report/aggregated_insights_report_builder.go [MODIFY] buildPDF + ext case (:161,:235)
└── (new) logo asset + logo fetch from mos-iam-server

# OR aaas-reporting (leading home — closest semantic fit, but no revenue + static auth map)
aaas-reporting/
├── go.mod                                              [MODIFY] add PDF lib
├── pkg/reports/reports.go                             [MODIFY] PDFFormat enum, channel+revenue projection (:34,:45,:171)
├── pkg/reports/exporter.go                            [MODIFY] new rows→PDF Exporter (:16-24)
├── pkg/reports/gcs-exporter.go                        [MODIFY] rows→render→upload path, logo fetch (:49-149)
├── internal/handlers/skad.go                          [MODIFY] accept "pdf" format (:60-70)
├── cmd/server.go                                      [MODIFY] Brightway auth-map entry (:83-145)
└── cmd/config/gcs.go                                  [MODIFY] override signed-URL TTL to 15min (:30-36)

reporting-api/
└── internal/versions/v1.4/api/api.go                 [MODIFY] relax delivery_format guard, admit "pdf" (:546)

frontend-mos/
└── packages/advertiser/src/types/ReportingInterface.ts [MODIFY] add pdf?: boolean to OutputFormat (:114)
    + 3 test files [MODIFY] add pdf case

# ko-k8s-apps (deploy topology) — see Open Questions on worker tier
ko-k8s-apps/
└── reporting/<worker>/{base,prod,qa}/*.yaml           [MODIFY/NEW] resources bump (PDF is memory-heavy) and/or new worker variant
```

**Structure Notes**: Generation home selection gates where the bulk of net-new code lands. `reporting-dispatcher` requires **zero code change** if quarter epochs are injected upstream and an existing `req_type` is reused (CA-18); if a new `req_type` is introduced it needs cases in two switches + a concurrency-gate case (CA-16/CA-17).

---

## Affected Systems

### reporting-api

**Technology**: Go, Marathon/Mesos, MySQL + JSON `req_request` blob (investigation-verified).
**Key Files**:
- `internal/versions/v1.4/api/api.go:546` — on-demand `delivery_format` validation guard
- `internal/versions/v1.4/api/api.go:1480` — scheduled-path guard (only if weekly model chosen)
- `kochava_accounts.feature_rollout` (data) — optional per-account gating

**Changes**:

| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Admit `"pdf"` in delivery_format guard (`api.go:546`) | Modify | L | The single code edit in this repo. Additive only (NFR-010). |
| Optional `feature_rollout` row for Brightway gating | New (data) | L | Only if feature must be restricted; data not DDL (CA-DM2). |

### Generation worker (summary-report-cloud OR aaas-reporting) — [DECISION REQUIRED: generation home]

**Technology**: Go, Pub/Sub consumer, BigQuery, GCS/S3 (investigation-verified).
**Key Files** (summary-report-cloud shown; aaas-reporting equivalents in Project Structure):
- `go.mod` — add PDF library
- `aggregated-report/aggregated_insights_report_builder.go:161,:235` — format + extension switch
- `report_runner.go:18-54` — template routing
- `delivery_format.go:3-9` — format allow-list/validation

**Changes**:

| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Add PDF render library | New (dep) | M | Needs dependency approval. Working default `maroto`. |
| `buildPDF` renderer: one-table layout (channel rows × revenue/conversions), embed Brightway logo | New | H | Single biggest net-new piece. Logo from mos-iam-server (PNG/JPEG). |
| Channel-grouped + revenue query (GROUP BY channel, SUM revenue, COUNT conversions, quarter window) | New/Modify | H | Gated on revenue + channel [DECISION REQUIRED]. `summary-report-cloud` already has `event_revenue`+network grouping to adapt. |
| `.pdf` extension + format-switch cases | Modify | L | Two sites in summary-report-cloud (main.go:772, aggregated builder:235). Prefer aggregate path (clean switch) over 400-line CSV-coupled summary path (CA-P6). |
| Signed-URL TTL ~15min default | Modify | L | Reuse existing signed-URL mechanism; override TTL. |
| Named failure statuses (timeout/dependency-unavailable/rate-limited) + bounded retry (3, exp backoff, Retry-After) | Modify | M | Build on existing retry-aware status callback (summary-report-cloud main.go:444-465). Preserve prior successful report. |
| Read-rate cap on attribution source (NFR-002) | New | M | Keep BatchPriority BQ queries + conservative concurrency/QPS cap, tunable (provisional default below). |
| Generation monitoring (NFR-003) | New | L | Error-rate >1%/5min + source read-util >80%/5min via existing Prometheus/Datadog wiring. |
| Audit entries (NFR-008) | New | M | Download + generation outcomes, async, alert-on-write-failure. Sink = audit-api ingest vs in-service logging [DECISION REQUIRED]. |
| Brightway auth-map entry (aaas-reporting only) | New (data/code) | L | aaas-reporting has hardcoded static auth map + Spanner publisher→network-ID resolution (cmd/server.go:81-146). |

### reporting-dispatcher

**Technology**: Go, single-instance bare-metal `reporting02`, format-agnostic passthrough (investigation-verified).
**Key Files**:
- `root.go:1263-1335` — worker routing switches (only if new `req_type`)
- `root.go:1071-1147` — concurrency gate (no default case — new type bypasses gating; CA-17)
- `helpers.go:190-261` — named-range window (only if quarter math lands here)

**Changes**:

| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| **None (recommended path)** | — | — | If `summary` type reused + epochs injected upstream → zero change. |
| Add new `req_type` to both routing switches + concurrency gate case + `kochava_reports.dispatcher` row | Modify | M | Only if a new report type is chosen. Concurrency-gate omission (CA-17) means a new type dispatches ungated unless explicitly added. |

### frontend-mos

**Technology**: Vue 3 + Pinia + TS, data-driven format dropdown (investigation-verified).
**Key Files**:
- `packages/advertiser/src/types/ReportingInterface.ts:114` — `OutputFormat`
- 3 test files (CustomizeReportModal.spec.ts, reports_card.spec.ts, reporting.spec.ts)

**Changes**:

| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Add `pdf?: boolean` to `OutputFormat` | Modify | L | **Only mandatory frontend code change.** Optional so existing mocks stay green. Dropdown auto-renders PDF once backend template advertises `pdf:true`. |
| Add PDF cases to 3 test mocks | Modify | L | Additive assertions. |

### mos-iam-server

**Technology**: Go, Spanner, Keto authz (investigation-verified).
**Changes**:

| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| **None required (data-only)** | — | — | Logo already exposed via `logo_url` org field + `/logos/download`. BR-5 enablement is a `POST /admin/organization/:id/feature-flags` data call. Only a *new* authz operation would touch `internal/common/constants.go` — a design choice, not a necessity. |

### Resolved KG discrepancies — NOT affected systems

The following were investigated live and **ruled out** (zero data-model/API/file changes): `reporting-subscriptions-worker` (run-once CLI, wrong domain), `audit-api` (pattern donor only unless chosen as audit sink), `pubsuite-attribution` (write-only source; conditional one-file revenue column only if the validated-column path is chosen over `event_metadata` extraction), `adsmeasurement-storage-api`, `koch-query-api` (executor only; conditional if design routes sourcing through `/async`), `krepcustomworker` (custom-SQL only, ignores format), `csi` (related ops console, optional status/type-dropdown sync only), `botler`, `clog`, `constants`, `geoip`, `koapi`, `kocache`, `kog-legacy`, `mqa`, `pubsuite-config-api`, `segment`, `uadd`, `docs` (optional doc note only).

---

## Implementation Phases

### Phase 0: Test Coverage Foundation

**Goal**: Establish test coverage for the thinly-tested generation home before modifying it.
**Prerequisites**: Generation-home decision.

- [ ] If `aaas-reporting`: write baseline unit tests for `pkg/reports` builder/exporter (currently only filters_test.go exists)
- [ ] Verify existing CSV/JSON tests pass in chosen home (NFR-010 regression guard)
- [ ] Set up BigQuery/pipeline integration test fixtures for quarter window + channel grouping

**Deliverables**: Test coverage for affected code paths; green baseline.

---

### Phase 1: Infrastructure & Data

**Goal**: Foundational, decision-unblocked changes.
**Prerequisites**: Phase 0.

- [ ] Add PDF library to generation-home `go.mod` (pending dependency approval)
- [ ] (If gated) Add `feature_rollout` row / `org_feature_mapping` flag for Brightway
- [ ] Configure kill-switch (NFR-009): config/feature-flag disabling generation + K4A download affordance, propagating within 5 min — realistically same mechanism as BR-5 gate
- [ ] Set signed-URL TTL default to 15 min (NFR-006) in chosen home

**Deliverables**: Infrastructure ready for core implementation.

---

### Phase 2: Core Implementation

**Goal**: Primary functionality.
**Prerequisites**: Phase 1; revenue + channel + generation-home decisions resolved.

- [ ] Calendar-quarter window computation (most recent completed calendar quarter at generation time) → inject explicit `time_start`/`time_end` epochs upstream (pure-function, unit-tested)
- [ ] Channel-grouped revenue + conversions query (GROUP BY resolved channel field, SUM revenue, COUNT conversions, scoped to quarter + Brightway dataset)
- [ ] `buildPDF` renderer: one-table layout + Brightway logo embed (fetched from mos-iam-server org `logo_url`)
- [ ] Relax `reporting-api` `delivery_format` guard to admit `"pdf"` (api.go:546)
- [ ] Add `pdf?: boolean` to frontend-mos `OutputFormat`
- [ ] Unit tests for quarter math, query, PDF layout

**Deliverables**: Core feature functional end-to-end for Brightway.

---

### Phase 3: Integration

**Goal**: Connect components and verify end-to-end.
**Prerequisites**: Phase 2.

- [ ] Named failure statuses + bounded retry (FR-007/NFR-004); preserve prior successful report
- [ ] Server-side access enforcement on download/retrieve, re-derived per request, fail-closed (reconcile 403 vs convention per decision)
- [ ] Audit entries for download + generation outcomes (NFR-008)
- [ ] Integration tests: queued row → dispatcher pickup → worker → status/link → download; signed-URL TTL expiry; failure-status paths
- [ ] (If in scope) FR-006 incomplete-week data-quality warning in PDF

**Deliverables**: Feature working end-to-end with error handling.

---

### Phase 4: Polish & Documentation

**Goal**: Finalize for release.
**Prerequisites**: Phase 3.

- [ ] Read-rate cap tuning (NFR-002) against observed BQ capacity
- [ ] Generation monitoring alerts (NFR-003)
- [ ] ko-k8s-apps resources bump / worker-tier config (memory-heavy PDF rendering)
- [ ] CLAUDE.md + optional reporting-testing-guide.md note (`pdf` now supported)
- [ ] Code review cleanup; final regression pass (NFR-010)

**Deliverables**: Feature ready for release.

---

## Data Model & Contracts

- **Data Model**: [data-model.md](./data-model.md)
- **API Contracts**: None — no new API surface. `delivery_format:"pdf"` is an additive enum value on the existing `reporting-api` `/v1.4/summary` + `/v1.4/progress` contract; report params ride existing JSON fields; delivery is the existing signed-URL + download button.

---

## Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Attributed revenue has no usable source for Brightway's dataset | M | H | Resolve revenue source early (Phase 0 blocker). Likely `event_metadata.price`/`.sum` extraction (no schema change) or `summary-report-cloud.event_revenue`. A new column won't backfill last quarter — must confirm existing data covers the window. |
| PDF rendering + quarter math + revenue span multiple services under 5-week deadline | M | H | Scope held to Phase-1 MVP; working defaults for every open decision so engineering is never blocked; reuse existing pipeline/signed-URL/status machinery. |
| Breaking existing CSV/JSON consumers (NFR-010) | L | H | All changes additive (new enum value, new build branch); explicit allow-list validation; regression tests in Phase 0/4. |
| "403" requirement forces a behavior change on reporting-api path | M | M | Reconcile whether server-side denial (200+error/401) satisfies FR-010 before coding; avoid changing existing client-visible status codes. |
| PDF generation memory pressure on worker fleet | M | M | Resources bump in ko-k8s-apps; prefer KEDA-per-request worker model for bursty on-demand generation. |
| Logo is PNG/JPEG only (no SVG); renderer dependency on mos-iam-server | L | M | Confirm Brightway `logo_url` is populated and PNG/JPEG before build; cross-service read on generation path only (not download path). |

---

## Dependencies & Blockers

| Dependency | Owner | Status | Needed By |
|------------|-------|--------|-----------|
| Revenue source + definition + channel mapping confirmation | PM / Data | Open | Phase 2 (blocks report content) |
| Generation-home selection | Backend Architect | Open | Phase 0 |
| PDF library approval | Go Dependency reviewers | Open | Phase 1 |
| 403-vs-convention ruling | PM / Backend Architect | Open | Phase 3 |
| Standard-plan gating decision | PM / Entitlements | Open | Phase 1 (if gated) |
| ko-k8s-apps worker-tier / resources | Infra | Open | Phase 4 |
| Audit sink decision (audit-api vs in-service) | Backend | Open | Phase 3 |

---

## Appendix

### Related Documents
- [Product Spec](./product-spec.md)
- [Engineering Research](./eng-research.md)
- [Backend Eng Context](./eng-context-backend.md)

### Decision Log

| Date | Decision | Rationale | Decided By |
|------|----------|-----------|------------|
| 2026-11-22 | On-demand generate→poll→download model | DEC-001/DEC-007 resolved on-demand; least new infra; reuses existing pipeline | PM (spec) |
| 2026-11-22 | Inject quarter epochs upstream (not dispatcher) | Avoids dispatcher's buggy quarterly approximation; model-agnostic; zero dispatcher change | Eng (research) |

### References
- Reporting pipeline map: `docs/4-Guides/Reporting/reporting-testing-guide.md`
- OpenAPI 3.0 spec (for reference only — no new contract this feature)

---
## Open Questions & Decisions Register

_Durable, ID'd reference for the next HITL gate — inline `[DECISION REQUIRED: DEC-00N]` tags in the sections above point back here. Reflects the latest round only._

### Decisions Required (business/technical trade-offs — needs a human call)

_Each carries an **Owner** and a **Provisional Value** engineering may build against until the decision lands._

| ID | Decision | Owner | Provisional Value (build against until confirmed) |
|----|----------|-------|-----------------------------------------------------|
| **DEC-008** _(Code-only)_ | Generation home for the PDF worker: aaas-reporting (closest semantic fit — on-demand attribution, conversions-by-network, 403-capable — but no revenue column, hardcoded static auth map lacking Brightway, thin test coverage) vs summary-report-cloud (has first-class event_revenue + network grouping + clean format switch, but no on-demand-at-download trigger and a 400-line CSV-coupled summary path). | Backend Architect | summary-report-cloud via its aggregate builder path — it already has event_revenue + network grouping and a clean format switch (aggregated_insights_report_builder.go:161), avoiding the aaas-reporting revenue gap and static-auth-map work; revisit if revenue is confirmed to live in aaas-reporting's dataset instead. |
| **DEC-009** _(Code-only)_ | Attributed-revenue source and definition, plus the channel-dimension field mapping. No revenue column exists in pubsuite-attribution.conversions or aaas-reporting; event_metadata JSON carries price/sum/currency/item_quantity, and summary-report-cloud has first-class event_revenue. 'channel' has no column (candidates: partner, network, claim_traffic_type). Must define what 'attributed revenue' means numerically and which table/column holds it for Brightway's last-quarter data. | PM / Data Engineering | Use summary-report-cloud's existing event_revenue summed per network as revenue, and map 'channel' to the network/partner field, IF that dataset covers Brightway's attribution data for the window; fall back to event_metadata.price (×item_quantity) extraction (no schema change) only if event_revenue does not cover Brightway. Do NOT add a new validated column — it cannot backfill last quarter within the deadline. |
| **DEC-010** _(Code-only)_ | On-demand vs weekly-scheduled cadence. Spec has an internal contradiction (BR-1 on-demand generate-fresh-at-download vs FR-001 weekly system-triggered). FR-004 (idempotent per week+window) and FR-005 (no overlapping runs) only make engineering sense under the scheduled model. | PM | On-demand (per DEC-001/DEC-007), reusing the existing one-time queue path; treat FR-004/FR-005 as effectively N/A for Phase 1 since there is no weekly cycle to key or overlap. Confirm so FR-004/FR-005 scope can be finalized. |
| **DEC-011** _(Code-only)_ | Whether FR-010's 'server-side 403' is mandatory, given no service on the K4A path returns a clean 403 (reporting-api returns 200+error-body, mos-iam-server returns 401). Matching 403 literally is a behavior change with existing-client risk. | PM / Backend Architect | Accept any server-side denial (reporting-api's existing 200+{status:Error}, or 401) as satisfying FR-010's intent (re-derived per request, fail-closed), rather than forcing a new 403 on the shared reporting-api path. Only emit a true 403 if generation lands in aaas-reporting (which already returns 403). |
| **DEC-012** _(Code-only)_ | Whether the feature should be restricted to Brightway (gated) or open to all standard-plan accounts, and via which mechanism (feature_rollout whitelist row vs org_feature_mapping flag). No service models plan tiers, so BR-5 is satisfied by default unless deliberately gated. | PM / Entitlements | Gate to Brightway initially via a kochava_accounts.feature_rollout whitelist row (data, not DDL) that also serves as the NFR-009 kill-switch home; confirm the exact flag name and that Brightway is entitled. |
| **DEC-013** _(Code-only)_ | Whether the FR-006 incomplete-week data-quality warning is truly Phase-1. It is net-new everywhere (no existing completeness check; the PDF itself is new) and is in tension with a 'most recent completed calendar quarter' window where weeks should already be complete. | PM | Defer FR-006 incomplete-week warnings out of Phase 1 (the completed-quarter window makes still-processing weeks unlikely), keeping Phase 1 to the fixed one-table content; re-add if finance requires it. Confirm before Phase 3. |
| **DEC-014** _(Code-only)_ | Audit-trail sink for NFR-008 download/generation events: audit-api ingest endpoint (append-only, async, alert-on-write-failure) vs in-service structured logging within the generation worker. | Backend | Emit audit entries to audit-api's ingest endpoint (async, with an operational alert on write failure per NFR-008), reusing its who-did-what→BigQuery model; fall back to in-service append-only logging only if audit-api ingest cannot meet the async-no-block requirement. |
| **DEC-015** _(Code-only)_ | Deploy topology for the PDF generation worker: reuse an existing fleet vs a new worker tier, and resource sizing for memory-heavy PDF rendering (ko-k8s-apps). Precedent: pubsuite-lift-report-worker runs generation as a separate single-replica Pub/Sub consumer run-mode; summary-worker is static-20-replica; detail-worker is KEDA-per-message. | Infra / Backend Architect | Run PDF generation on the chosen home's existing worker fleet with a memory/resources bump (aaas-reporting is only 512Mi today and would need raising); prefer a KEDA-per-message ScaledObject for bursty on-demand generation if a new variant is introduced. No cron tier exists, so no single-replica no-overlap concern applies under on-demand. |

### Open Questions (evidence gaps — needs more input)

- **OQ-009** _(Code-only)_: Which specific worker repository actually builds the CSV/JSON files for reporting-dispatcher today (referenced only by binary name in the type switch at root.go:1263 and by a koch-query-api comment) — needed to confirm the exact generation home and whether a new req_type must be added to the dispatcher switches/concurrency gate.
- **OQ-010** _(Code-only)_: Does Brightway's BigQuery conversions/attribution dataset actually contain populated revenue data (event_revenue, or event_metadata.price/sum) for the most recent completed calendar quarter — confirmable only against live data, not from code; blocks report content if absent.
- **OQ-011** _(Code-only)_: What are the actual k8s replica counts / HPA / run-mode for the candidate cloud workers? Deploy manifests live in the external ko-k8s-apps repo; aaas-reporting is static 3-replica/512Mi, summary-worker static 20-replica — exact prod topology and whether a memory bump suffices for PDF rendering must be confirmed in that repo.
- **OQ-012** _(Code-only)_: Where is the external ingress route for any download/status endpoint served from? reporting-api's external hostname (reporting.api.kochava.com) is not fronted by an HTTPRoute in ko-k8s-apps reporting/mesh/prod — the edge/gateway that exposes it is outside the investigated repos and must be confirmed if any routing change is needed.
