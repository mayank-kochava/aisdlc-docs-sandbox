# Research: Weekly Last-Quarter Attribution PDF

**Feature**: Weekly Last-Quarter Attribution PDF (Brightway Retail finance self-service download)
**Date**: 2026-11-22
**Phase**: 0 - Research & Technology Decisions

## Overview

This research validates the product spec for the weekly last-quarter attribution PDF against the actual Kochava reporting codebase. Nine repositories were investigated with live Read access (`frontend-mos`, `reporting-api`, `reporting-dispatcher`, `pubsuite-attribution`, `reporting-subscriptions-worker`, `summary-report-cloud`, `aaas-reporting`, `audit-api`, `mos-iam-server`) and corroborated by knowledge-graph evidence. A backend engineering context pass synthesized across all nine.

The headline finding: **this feature is overwhelmingly an extension of an existing report generate → poll → download pipeline, not greenfield work.** The on-demand submit/poll UX, the status state machine, the signed-URL delivery, the feature-flag gating, and the org-logo asset store all already exist. The genuinely net-new work concentrates in three backend gaps — **PDF rendering + logo embedding**, **calendar-quarter window math**, and **attributed-revenue sourcing** — plus reconciling several spec assertions (403 vs. actual 401/200, "standard plan" as an entitlement flag, on-demand vs. weekly cadence) against codebase reality.

This document covers technology, architecture, performance, error-handling, and testing decisions, and flags the open product/architecture questions that gate design. It covers all requirement prefixes present in the spec: **BR-1..BR-6, FR-001..FR-011, NFR-001..NFR-011**, and the three eng-context/role concerns (backend, frontend UI surface, infra/deploy topology).

---

## Technology Stack Decisions

### Decision: PDF rendering library and host service

**Chosen**: Add a Go PDF-rendering capability to the report-generation **worker** layer (the service that already runs the BigQuery query and uploads the result file), rendering the one-table layout and embedding the Brightway logo. `aaas-reporting` is the leading candidate generation home; `summary-report-cloud` is the fallback. The concrete library choice (e.g. `maroto`, `gofpdf`, or an HTML→PDF path) is deferred to the planning phase with `maroto` as the working default. **See [DECISION REQUIRED] on generation home below.**

**Rationale** (investigation-verified):
- PDF output exists **nowhere** in the stack. Verified absent in `reporting-api` (`api.go:546` hard-rejects anything not `csv`/`json`; same guard at `api.go:1480`), `summary-report-cloud` (`delivery_format.go:3-9`, `aggregated_insights_report_builder.go:161-166` — only `csv`/`json`), and `aaas-reporting` (`reports.go:34-40`, `gcs-exporter.go:112-117` — only CSV/JSON to GCS). A repo-wide grep for `gofpdf/maroto/unidoc/wkhtml/chromedp` returned nothing in any report service.
- The dispatcher is **format-agnostic** — `reporting-dispatcher` passes `delivery_format` straight through (`root.go:45`, grep confirms no branching, CA-10), so PDF is purely a worker concern there. No dispatcher change is needed for format.
- `frontend-mos` has **no client-side PDF library** (verified zero matches for jsPDF/pdfmake/html2pdf), confirming rendering is server-side (CA-1).

**Alternatives Considered**:
1. **Client-side PDF generation in frontend-mos**
   - Pros: no backend rendering work.
   - Cons: finance data would be assembled client-side; conflicts with NFR-007 (no sensitive data where it shouldn't be) and the backend-only content model; no lib exists today.
   - Rejected: contradicts the data-driven, backend-shaped content model (frontend only links to the produced file, CA-1/CA-4).
2. **New dedicated PDF microservice**
   - Pros: clean separation.
   - Cons: new deployable, new blast radius, against the 5-week deadline; duplicates query/upload/signed-URL logic that already exists in the workers.
   - Rejected: deadline risk and needless duplication.

**Implementation Notes**:
- Logo source is confirmed: `mos-iam-server` stores org logos in GCS with the URL on the org record (`logoservice.go:135-196`, `LogoURL` first-class org field), **PNG/JPEG only, no SVG** (`logoservice.go:78-85`). The renderer pulls from the org record `LogoURL` or `/logos/download` — a cross-service read dependency.
- Reusable delivery exists: `aaas-reporting` V4 signed URLs (`gcs-exporter.go:184-202`), `summary-report-cloud` S3link path (`main.go:628-700`).

---

### Decision: Attribution data source and revenue sourcing

**Chosen**: Source the report from the attribution conversions BigQuery data. **Attributed revenue has no confirmed backing column today** and must be resolved before build — this is the single highest-risk unknown. Working default for planning: treat `summary-report-cloud`'s existing `event_revenue` as the revenue source **if and only if** it covers Brightway's attribution dataset; otherwise a schema/definition change is required. **See [DECISION REQUIRED] on revenue below.**

**Rationale** (investigation-verified):
- `pubsuite-attribution` writes `aaas_<SAN>.conversions` (`auditlog.go:337-378`) — a repo-wide grep for `revenue` returned **zero matches**. No revenue column, no channel column. Conversions = row count; closest channel-like dimensions are `partner` and `claim_traffic_type`.
- `aaas-reporting`'s summary report groups conversions by network/campaign with `sum(coalesce(count,1)) as count` (`reports.go:45-52`) — **no revenue column** either.
- `summary-report-cloud` **does** have `sum(t.event_revenue) as event_revenue` + `event_count` grouped by `network` (`summary_query_builder.go:142-203`) — but it is unconfirmed this is the same attribution data the spec intends for Brightway.
- The KG did **not** re-confirm `pubsuite-attribution` as the authoritative source at the Context Seed's stated confidence (medium). `reporting-api` confirmed it has **no BigQuery dependency at all** (data sources are MySQL/InfluxDB/outbound APIs) — the BQ querying lives in the external worker.

**Alternatives Considered**:
1. **Derive revenue from `event_metadata`/`claim_metadata` free-form maps in `pubsuite-attribution`**
   - Pros: no schema change.
   - Cons: these are free-form `map[string]string` (`auditlog.go:245,307`); no confirmation any client stores revenue there.
   - Rejected pending data confirmation: unverifiable from code; needs PM/data.
2. **Add a new revenue column to `pubsuite-attribution.conversions`**
   - Pros: authoritative, clean.
   - Cons: write-path change + backfill; **will not populate last quarter retroactively** — fatal for a last-quarter report under a 5-week deadline.
   - Rejected as primary unless no existing revenue source covers Brightway.

**Implementation Notes**:
- "Channel" must be mapped to a real field before the GROUP BY can be written — candidates `network`/`network_name`, `partner`, `claim_traffic_type`. `reporting-api`'s `groupLevel1` already lists `network` (`api.go:55`) and `includeValid` lists `revenue` (`api.go:67`) as vocabulary, but actual computation is in the worker.

---

## Architectural Patterns

### Decision: On-demand generate → poll → download, reusing the existing pipeline

**Chosen**: Implement Phase 1 on the **on-demand one-time generation path** — frontend POSTs a report request, a `queued` row lands in `kochava_reports.requested`, the dispatcher picks it up (~10s), a worker generates the PDF and writes status + signed link, and the frontend polls `/progress` and surfaces the download. This resolves the spec's internal BR-1-vs-FR-001 tension in favor of BR-1 (on-demand), consistent with DEC-001/DEC-007. **The on-demand-vs-weekly ruling is itself a required product decision — see [DECISION REQUIRED] below.**

**Rationale** (investigation-verified):
- The entire generate→poll→download flow already ships for CSV/JSON. Frontend: download button + status chips (queued/running/completed/error) + 10s polling all exist (`CompletedReports.vue:39-95`, `reporting.ts:665-712`, CA-1/CA-2/CA-3). Backend queue: `reporting-api` `/v1.4/summary` → `kochava_reports.requested` (`api.go:4021-4043`) → dispatcher loop (`root.go:486`, `sendReportJobs` root.go:757) → worker exec/Pub/Sub (CA-2/CA-5/CA-6). Status/progress: `/v1.4/progress` (`api.go:4252-4302`).
- The dispatcher picks up **any** `queued` row within one ~10s cycle regardless of who inserted it (`queries.go:33-125`, CA-3) — on-demand needs **no dispatcher change** beyond a PDF-capable worker.
- This directly satisfies the spec's "in-progress → ready" UX and the "minute or two latency acceptable" model (OQ-008 resolved). `aaas-reporting`'s two-phase async submit (`skad.go:43-171`) + result poll (`skad.go:179-302`, statuses running/complete/failed) is the closest existing on-demand shape.

**Spec-internal tension (flagged, not silently resolved)**: BR-1 says on-demand/generate-fresh-at-download; FR-001 and the §2 flow describe weekly system-triggered cadence. Both paths exist in code: on-demand = the one-time queue path above; weekly = `reporting-dispatcher`'s generic scheduled loop (`addScheduledReportThatAreReady` root.go:1717, inserts `queued` rows from `kochava_reports.scheduled`, weekly frequency supported root.go:1978/2045). **FR-004 (idempotent per week+window) and FR-005 (no overlapping runs) only make engineering sense under the scheduled model** — under pure on-demand there is no "week" key and no cycle to overlap. This must be reconciled before FR-004/FR-005 can be designed.

**Alternatives Considered**:
1. **Weekly scheduled generation (FR-001 literal)**
   - Pros: FR-004/FR-005 map cleanly; `reporting-dispatcher`'s single-instance deploy (`reporting02`, resets in-flight rows on startup, CA-11) gives no-overlap (FR-005) essentially for free; weekly cadence engine already exists.
   - Cons: contradicts BR-1/DEC-001/DEC-007; pre-generates/stores reports the spec says not to; the scheduled window math is coupled to cadence and only yields rolling or approximate-quarter windows (CA-8), not "most recent completed calendar quarter."
   - Rejected as primary: conflicts with resolved delivery model; kept as the fallback the decision-owner may still elect.

**Implementation Notes**:
- `reporting-subscriptions-worker` and `audit-api` are **ruled out as generation/scheduling homes** (investigation-confirmed). The former is run-once per `--report-id`, subscriptions domain, no scheduler (CA-1/CA-3/CA-7); `isScheduled()` only renames files. The latter is audit-domain only, no attribution data, no scheduler (CA-3/CA-6). The KG "reusable weekly-cadence worker" is really the **generic scheduled loop in `reporting-dispatcher`**, not a subscription service.

---

### Decision: Calendar-quarter window computed upstream at generation time

**Chosen**: Compute "most recent completed calendar quarter" at generation time and inject it as explicit `time_start`/`time_end` epochs into the report request. This is **net-new logic** and lives in the request builder (frontend or reporting-api), not in the worker's window math.

**Rationale** (investigation-verified):
- **No calendar-quarter logic exists anywhere.** `reporting-api`'s window is a rolling `now − offset − previous_time` with no calendar alignment (`calculateStartEndForReport` api.go:5330-5365, CA-5). `reporting-dispatcher`'s `quarterly` frequency is a 65-day approximation with a suspicious double-subtraction bug (`root.go:1996-2004`, CA-8) and couples cadence to window (CA-8). No `last quarter` named range exists (`helpers.go:190-203`, CA-9).
- Workers take `time_start`/`time_end` **pre-computed** in the request and never compute windows themselves (`summary-report-cloud` `summary_query_builder.go:30-64` CA-6; `aaas-reporting` `reports.go:61-80` CA-4). `pubsuite-attribution` filters on `event_timestamp` (CA-1). The request contracts already carry `time_start`/`time_end` (`ReportStruct` root.go:34-71; `reportStruct` main.go:57-92; `ReportRequest` ReportingInterface.ts:125-156).
- `audit-api` demonstrates the precedent of computing a window at request time via `time.Now().UTC()` in `Validate()` (`filters.go:90-121`), but has no quarter helper either.

**Alternatives Considered**:
1. **Add a `last quarter` named range to `reporting-dispatcher` (`helpers.go`)**
   - Pros: reusable, declarative.
   - Cons: only helps the scheduled path; still needs true calendar-quarter math; touches a shared file.
   - Rejected for Phase 1: explicit epoch injection is simpler and model-agnostic.

**Implementation Notes**: Frontend already converts date bounds to unix (`CustomizeReportModal.vue:533-534`); the quarter computation can live frontend- or backend-side with no blocker found (frontend CA-5). Avoid the dispatcher's broken `quarterly` approximation.

---

### Decision: Secure, time-limited delivery via signed URLs (FR-011 / NFR-006)

**Chosen**: Deliver the PDF via a short-lived signed URL (S3 presigned / GCS V4 signed), default ~15-minute validity, tunable. **Reuse existing mechanisms — no new delivery infra.**

**Rationale** (investigation-verified): Signed-URL delivery already exists — `aaas-reporting` V4 signed URLs (`gcs-exporter.go:184-202`, returned only on `complete`), `summary-report-cloud` `S3link` (`main.go:628-700`). `reporting-api` delivery is an S3 link surfaced via `GetProgress`, never a streamed byte response (CA-7). Frontend downloads via existing `<v-btn :href="report_url" target="_blank">` (CA-1). This satisfies NFR-001's requirement that serving a pre-generated file not trigger synchronous generation.

**Implementation Notes**: Set link TTL to the NFR-006 default (15 min); the browser download uses the existing `report_url`/`req_report_link` column (`00-init.sql:56-82`, CA-13).

---

## Performance Considerations

### Decision: Read-rate cap on the attribution data source (NFR-002) with conservative default

**Chosen**: Apply an explicit, conservative, tunable read-rate cap during generation against the shared attribution BigQuery source. **The concrete cap value is a [DECISION REQUIRED] — provisional default below.**

**Rationale**: NFR-002 mandates capping generation read load so it doesn't degrade other consumers of the shared source. The workers already run BigQuery in `BatchPriority` by default (`aaas-reporting` `reports.go:181-186`), which is the lower-impact query tier — a good baseline. No existing explicit rate cap was found, so this is net-new tuning.

**Implementation Notes**: Default to BatchPriority queries plus a conservative concurrency/QPS cap engineering tunes against observed BQ capacity. NFR-001's p95 ≤ 3s applies only to **serving an already-generated file**, not generation — the signed-URL/link path (above) satisfies it since no synchronous generation is on the download path.

### Decision: Generation monitoring (NFR-003)

**Chosen**: Emit alerts on generation error-rate >1% sustained 5 min and attribution-source read-utilization >80% sustained 5 min, reusing the existing observability stack.

**Rationale** (investigation-verified): `summary-report-cloud` already wires Prometheus + Datadog + Zipkin + OpenCensus views (`main.go:155-164`, 306-311, CA-11). New generation steps emit through these existing exporters — no new observability infra needed.

---

## Error Handling Strategy

### Decision: Distinct named failure statuses with bounded retry (FR-007 / NFR-004)

**Chosen**: On attribution-source failure, retry with bounded exponential backoff up to 3 attempts honoring any Retry-After; record **distinct named statuses** for timeout, dependency-unavailable, and rate-limited; preserve the prior successful report unchanged.

**Error Categories**:
- **Timeout** — query/export exceeds budget (`aaas-reporting` has a 3-min query timeout, `skad.go:142`).
- **Dependency-unavailable** — attribution source unreachable/erroring.
- **Rate-limited** — honor Retry-After per NFR-004.
- **Succeeded** — current report available for download.

**Rationale** (investigation-verified): `summary-report-cloud` already has a retry-aware status callback (`updateStatusWithRetry` main.go:444-465, linear backoff + jitter) and a running→completed/error state machine (`main.go:501-513`, CA-8) to build distinct statuses on. `reporting-dispatcher` auto-disables a scheduled report after 7 failures (`RecordScheduledFailure` root.go:2488, CA-14) — relevant if the scheduled model is chosen.

**Implementation Notes**: Per NFR-007, never log raw report payloads or user-identifying data — log identifiers + status only. The "prior successful report remains available" guarantee maps to not overwriting the last-good link on failure.

### Decision: Server-side access enforcement — reconcile 403 vs. codebase convention (FR-010 / NFR-005)

**Chosen**: Enforce authorization server-side on every download/retrieve request, re-derived from the authoritative source, failing closed if auth is unreachable. **The spec's literal "403" does not match any investigated service — this is a flagged correction, see Corrected Assumptions and [DECISION REQUIRED] below.**

**Rationale** (investigation-verified): No service returns a clean 403. `reporting-api` returns **HTTP 200 + `{status:"Error"}`** on denial (`appCheckerMiddleware.go:100-120`, `writeResponse` keeps status 200, CA-6). `mos-iam-server` returns **401** (`logoservice.go:174`, `ketoadapter.go:392`, CA-3). `aaas-reporting` is the only one with a real **403** path (`respond.go:35-60`, CA-8). Matching "403" literally against the K4A path (reporting-api) is a behavior change with existing-client risk, since `reporting-api` clients expect 200+error-body. Authorization re-derivation on every request is well-precedented (Keto tuple check in `mos-iam-server`, `X-User` from Oathkeeper; `audit-api` account_id scoping CA-4).

**Implementation Notes**: "Standard plan" has no tier model anywhere — see Corrected Assumptions. If generation lands in `aaas-reporting`, Brightway must be added to its **hardcoded static auth map** (`cmd/server.go:81-146`, CA-7) and resolved through the Spanner publisher→network-ID path (`cmd/server.go:63-79`, CA-5).

---

## Testing Strategy

### Decision: Unit + integration coverage concentrated on net-new code, in Go

**Chosen**: Go unit tests for the three net-new areas (PDF rendering, quarter-window math, revenue/channel query), integration tests through the existing queue→dispatch→worker→status path, and a frontend type/render check for the new PDF format option.

**Rationale** (investigation-verified): The likely generation homes are **thinly tested** — `aaas-reporting` has only `pkg/reports/filters_test.go` (no handler/builder/exporter tests, CA investigation "could not verify" #5); net-new PDF/quarter work would land in largely untested packages. `summary-report-cloud` has format tests (`delivery_format_test.go`) and status-lifecycle code to build on. Quarter math is pure-function testable.

**Test Types**:
| Test Type | Scope | Tools |
|-----------|-------|-------|
| Unit | Calendar-quarter boundary computation; channel GROUP BY / revenue aggregation; PDF layout + logo embed | Go `testing` |
| Integration | `queued` row → dispatcher pickup → worker → status/link; signed-URL TTL; failure statuses + retry | Go, existing report pipeline harness |
| Frontend | `pdf?: boolean` on `OutputFormat`; data-driven dropdown renders PDF; download via existing `:href` | existing frontend-mos test setup |

**Implementation Notes**: Idempotency/no-overlap tests (FR-004/FR-005) are only meaningful if the scheduled model is chosen; defer until that ruling lands.

---

## Specification Validation

### Confirmed Assumptions

| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| On-demand generate→poll→download with ~1–2 min latency, in-progress/ready states (BR-1, OQ-008) | frontend `CompletedReports.vue:39-95`, `reporting.ts:665-712`; `reporting-api` `/v1.4/progress` api.go:4252; `aaas-reporting` skad.go:43-302 | Entire UX already ships for CSV/JSON; reuse directly. |
| Download delivered via time-limited secure link, not public URL (FR-011/NFR-006) | `aaas-reporting` gcs-exporter.go:184-202; `summary-report-cloud` main.go:628-700; `reporting-api` CA-7 | Signed/presigned URLs exist; set ~15min TTL. |
| K4A download button is the delivery mechanism (OQ-002, BR-4) | frontend `CompletedReports.vue:39-95` (download `<v-btn :href>`); `reporting-api` is called-by frontend (KG edge direction corrected) | Button + status chips already exist; frontend change is small. |
| Weekly cadence mechanism exists to reuse (FR-001, if scheduled chosen) | `reporting-dispatcher` addScheduledReportThatAreReady root.go:1717, weekly root.go:1978/2045 | Generic scheduled loop, not a subscription-specific worker. |
| Brightway logo is an existing managed org asset (BR-3 branding) | `mos-iam-server` logoservice.go:135-196, `LogoURL` org field | PNG/JPEG only, no SVG; renderer pulls from org record / `/logos/download`. |
| "Standard plan" expressible via existing entitlement/feature-flag mechanism (BR-5) | `mos-iam-server` org_feature_mapping / org-feature-flag.go:17-78; `reporting-api` APIReportingAccess flag CA-6; frontend `featureRollout` k4aAccount.ts:133 | No tier model exists; a feature flag is the realistic gate — needs entitlements confirmation (see Corrected). |
| Conversions-by-channel data shape exists (report contents, FR-002) | `summary-report-cloud` summary_query_builder.go:142-203 (network grouping); `aaas-reporting` reports.go:45-52 | Conversions/channel shape exists; revenue is the gap (see Corrected). |
| Retry-aware failure handling + preserve prior report (FR-007/NFR-004) | `summary-report-cloud` updateStatusWithRetry main.go:444-465; `reporting-dispatcher` RecordScheduledFailure root.go:2488 | Build distinct named statuses on existing callback machinery. |
| Observability stack already wired for NFR-003 alerts | `summary-report-cloud` main.go:155-164, 306-311 | Prometheus/Datadog/OpenCensus; emit new metrics through these. |

### Corrected Assumptions

| Spec Statement | Actual Reality | Impact on Implementation |
|----------------|----------------|-------------------------|
| "Deliver as PDF" (BR-3) is a format toggle | **No PDF anywhere** — all report services do CSV/JSON only (`api.go:546`, `delivery_format.go:3-9`, `reports.go:34-40`); no PDF lib in any `go.mod` | PDF rendering + logo embedding is the single biggest net-new build, in the worker layer. |
| "Last quarter = most recent completed calendar quarter" is a supported window | **No calendar-quarter logic exists**; `reporting-api` window is rolling (api.go:5330-5365); dispatcher `quarterly` is a buggy 65-day approximation (root.go:1996-2004); no `last quarter` named range (helpers.go:190-203) | Quarter math is net-new; inject explicit `time_start`/`time_end` at generation time. |
| Report shows "attributed **revenue** by channel" | **No revenue column** in `pubsuite-attribution.conversions` (zero grep hits, auditlog.go:337) or `aaas-reporting` summary; only `summary-report-cloud` has `event_revenue` (unclear it's Brightway's attribution data). No `channel` column — closest is `partner`/`network`/`claim_traffic_type` | **Highest-risk gap.** Revenue definition + source, and channel mapping, must be resolved by PM/Data before build (see [DECISION REQUIRED]). A new column won't backfill last quarter. |
| Authorization denial is a "server-side 403" (FR-010/§2 flow) | No clean 403 on the K4A path: `reporting-api` returns **200+error-body** (appCheckerMiddleware.go:100-120); `mos-iam-server` returns **401** (logoservice.go:174); only `aaas-reporting` returns 403 (respond.go:35-60) | Matching "403" literally is a behavior change with existing-client risk. Reconcile whether "server-side denial" (any form) satisfies the requirement (see [DECISION REQUIRED]). |
| "Standard-plan account" is a plan/tier check (BR-5) | **No service models plan tiers.** Gating is `APIReportingAccess` (reporting-api), per-org `org_feature_mapping` flags (mos-iam-server), or `status='activated'` (dispatcher) | BR-5 = a feature-flag/entitlement, not a tier check. Confirm which flag means "standard-plan + report-enabled" and that Brightway is entitled. |
| `pubsuite-attribution` is the authoritative attribution data source (Context Seed, medium KG confidence) | KG did **not** re-confirm at that confidence; `reporting-api` has no BQ dependency at all; `pubsuite-attribution` is write-only to BQ via Pub/Sub (runner.go:135-185) and never reads it back | Verify the authoritative read source during design; the read side is built in a worker, not pubsuite-attribution. |
| KG edge "reporting-api calls out to frontend-mos" | **Inverted** — `frontend-mos` calls `reporting-api` (`reporting.api.kochava.com/v1.4`); no frontend-mos reference in reporting-api code | Correct the data-flow direction; download affordance is a frontend-mos change. |
| FR-006 incomplete-week data-quality warning has an existing equivalent | **None.** `summary-report-cloud` `_PARTIAL` means row-cap-exceeded, unrelated (CA-9). No attribution-completeness check anywhere | Net-new logic, and net-new in a PDF that doesn't exist yet. Also in tension with "most recent *completed* quarter" — confirm Phase-1 scope. |
| FR-004/FR-005 (idempotent, non-overlapping) apply under on-demand | These only make sense under the **scheduled** model (week+window key, cycles) | Their design is gated on the on-demand-vs-weekly ruling. |

---

## Role & Requirement Coverage Notes

- **Backend (eng-context-backend.md):** All eight of its open questions (generation home, revenue existence, channel definition, 403 convention, standard-plan gating, FR-006 scope, audit-api fit, on-demand-vs-weekly) are reflected above as Decisions, Corrected Assumptions, or [DECISION REQUIRED] items.
- **Frontend (UI surface):** `frontend-mos` was investigated live. The only required change is adding `pdf?: boolean` to `OutputFormat` (`ReportingInterface.ts:114`); the data-driven dropdown auto-renders PDF once the backend template advertises `pdf: true` (CA-4). No separate admin/key-management UI surface was named by this spec or investigation. Route/menu registration pattern for any dedicated entry point is `router/index.ts:745-762` + `useAdvertiserMenu.ts:356-372` (CA-9).
- **Infra/deploy topology:** Flagged but **not fully verifiable from service repos** — k8s manifests (replica counts, HPA) for the cloud workers live in the external `ko-k8s-apps` repo, not in any investigated checkout (confirmed for `aaas-reporting`, `summary-report-cloud`, `pubsuite-attribution`, `mos-iam-server`). `reporting-dispatcher` is deliberately single-instance on bare-metal `reporting02` (CA-11) — gives FR-005 no-overlap for free if scheduled is chosen. `reporting-api` deploys to Marathon/Mesos with SOPS-encrypted replica counts (unreadable). This gap is carried as an open question.
- **Audit trail (NFR-008):** `audit-api` exists and does who-did-what → BigQuery with edge-auth (read-side pattern CA-1/CA-2, account_id RBAC CA-4), but is **not** an attribution data source and is ruled out as a generation home. Whether it is the intended audit sink (via its ingest endpoint) vs. in-service logging is an open design question.
- **Kill-switch (NFR-009) & backward-compat (NFR-010):** Kill-switch needs a config/feature-flag home propagating within 5 min — realistically the same entitlement mechanism as BR-5. NFR-010 is constrained by the backend note that `reporting-api`'s `/v1` contract must stay additive (new `delivery_format` enum value, no field changes/removals).

---

## Summary

Research is substantially complete. The feature is a well-understood **extension** of an existing, live report pipeline: the on-demand UX, status machine, signed-URL delivery, feature-flag gating, and org-logo store all exist and are citable from live investigation. The net-new engineering is concentrated and identified: **PDF rendering + logo embedding**, **calendar-quarter window math**, and **attributed-revenue sourcing** — the last being the one genuinely blocking unknown.

Five decisions require product/architecture owners before design can finalize: the on-demand-vs-weekly cadence ruling (gates FR-004/FR-005 and scope), revenue existence/definition and channel mapping (gates the whole report content), the generation-home selection, the 403-vs-convention ruling, and the standard-plan entitlement mapping. Working defaults are provided for each so engineering is not blocked. Infra deploy topology for the cloud workers (`ko-k8s-apps`) and the audit-sink decision remain open.

**Decisions Made**: 11 (with working defaults where owner input pending)
**Open Questions**: 4 (generation-home, revenue/channel, 403 convention, standard-plan flag) + infra topology + audit sink + FR-006 scope
**Status**: Ready for Implementation Planning on reused components; **Blocked on revenue-sourcing confirmation** for report content and on the cadence ruling for FR-004/FR-005.

---
## Open Questions & Decisions Register

_Durable, ID'd reference for the next HITL gate — inline `[DECISION REQUIRED: DEC-00N]` tags in the sections above point back here. Reflects the latest round only._

### Decisions Required (business/technical trade-offs — needs a human call)

_Each carries an **Owner** and a **Provisional Value** engineering may build against until the decision lands._

| ID | Decision | Owner | Provisional Value (build against until confirmed) |
|----|----------|-------|-----------------------------------------------------|
| **DEC-008** _(Code-only)_ | On-demand (BR-1) vs weekly-scheduled (FR-001) generation model — the spec asserts both; FR-004/FR-005 (idempotency, no-overlap) only make sense under scheduled, and the resolved delivery model (DEC-001/DEC-007) says on-demand. Pick one before design. | PM + Backend architect | On-demand one-time generation path (reuse reporting-api /summary → dispatcher queued-row pickup → PDF worker); treat FR-004/FR-005 as N/A under this model unless scheduled is later chosen. |
| **DEC-009** _(Code-only)_ | Does 'attributed revenue' exist as data for Brightway's attribution dataset, where does it live, and what field is 'channel'? No revenue column exists in pubsuite-attribution.conversions or aaas-reporting; only summary-report-cloud has event_revenue. A new column won't backfill last quarter. | PM + Data | Use summary-report-cloud's existing event_revenue + event_count grouped by network as revenue/channel, IF it covers Brightway's attribution data; confirm before build. Map 'channel' = network. |
| **DEC-010** _(Code-only)_ | Which service is the generation home for PDF rendering + quarter math + logo embed: adapt aaas-reporting (closest semantic fit, but no revenue + hardcoded auth map + untested), extend summary-report-cloud (has revenue, no on-demand-at-download wiring), or new report type in the dispatcher worker switch? | Backend architect | aaas-reporting as primary (on-demand attribution, conversions-by-network, 403-capable), with summary-report-cloud as fallback if its event_revenue is the authoritative source. |
| **DEC-011** _(Code-only)_ | Does FR-010/NFR-005 require a literal HTTP 403, given reporting-api returns 200+error-body and mos-iam-server returns 401 today? Matching 403 literally on the K4A path is a behavior change with existing-client risk. | PM + Backend architect | Accept 'server-side denial in the path's existing convention' (reporting-api 200+error-body / mos-iam 401) as satisfying the requirement; do not force a true 403 unless a client contract demands it. |
| **DEC-012** _(Code-only)_ | Which existing entitlement expresses 'standard-plan + report-enabled' (BR-5), and is Brightway actually entitled? No service models plan tiers; gating is APIReportingAccess / org_feature_mapping flags / status='activated'. | PM + Entitlements | Introduce a new per-org feature flag (e.g. weekly_attribution_report) in mos-iam-server org_feature_mapping, checked server-side on the download path and surfaced to frontend via featureRollout; enable it for Brightway. |

### Open Questions (evidence gaps — needs more input)

- **OQ-009** _(Code-only)_: Which repository hosts the k8s deploy manifests (replica counts, HPA) for the chosen cloud-worker generation home? Investigation confirmed these live in the external ko-k8s-apps repo, not in any investigated service checkout — needs that repo examined before deploy-topology / single-replica-background-tier decisions can be made.
- **OQ-010** _(Code-only)_: Use audit-api through its ingest endpoint for the report access and generation audit trail (NFR-008).
- **OQ-011** _(Code-only)_: Is the FR-006 incomplete-week data-quality warning truly Phase-1? No completeness check exists anywhere and it is in tension with a 'most recent completed calendar quarter' window; if deferrable it meaningfully reduces Phase-1 scope against the 5-week deadline.
