# TG4: Run tests & validate build

## In short

Run the full build and tests for Kochava/bqgolden with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG4 |
| Repo | Kochava/bqgolden |
| Phase | Phase 3 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/bqgolden with every task above merged. Do not edit source.

This task is part of [DRY RUN] Amazon Ads eventGrainEligibility Row-Level Redaction.

### BQR / AGG / bqgolden / MYD / PBK / QRY / Config — UNRESOLVED
**Technology**: Undetermined — owning repositories were not checked out.
**Changes**: Cannot be enumerated responsibly. See Open Questions OQ-2, OQ-4, OQ-5. Scope sketch (spec-derived, not code-verified) is retained in eng-context-backend.md "Rough Scope" and eng-research.md. These groups are **blocking** for full planning.

## Technical Context
**Language/Version**: Go (watson ingestion service, verified via `go test -race -count=1 -tags=unit`). Stack for all other services: **undetermined — not code-verified** (eng-research OQ-2/OQ-4/OQ-5).
**Primary Dependencies**: watson claim-build pipeline (`request.go`, `reconcile.go`, `SetDynamicField`); existing write-time redaction split (spec-asserted, not verified); BigQuery per-app/per-account datasets; MySQL detail workers; postback engine; `bqgolden`.
**Storage**: BigQuery (redacted `app_N`/`account_N.analytics` + unredacted `unredacted_app_N`/`account_N_unredacted`), MySQL detail-report source tables, BigQuery sanitization-config model. No new tables/schemas (BR-15, OOS-1). No `ko-db-migrations` changes (OOS-9).
**Testing**: Go unit tests in watson (`go test -race -count=1 -tags=unit`); integration/E2E/regression harnesses undetermined pending service checkouts; QA capture gate (FR-ENF-003) owned by QA.
**Target Platform**: Kochava measurement/reporting/ingestion/postback backend services; Mimir monitoring (optional, for AUD only).
**Performance Goals**: Row-level p95 within current baseline (NFR-001); detail-worker per-row check adds no new query/external call (NFR-002); aggregate scan/cost within baseline (NFR-003).
**Constraints**: Fail-closed at ingestion (BR-2/BR-3); 13954-only (BR-13); additive/backward-compatible (NFR-011); inert-until-enforced (BR-11, FR-ENF-001); no historical backfill (BR-12, FR-ENF-005); never log raw Amazon data (NFR-006, FR-AUD-003).

### Source Code Changes
```text
watson/  (VERIFIED code home for ING only)
└── cmd/
    └── watson/
        └── amazonadssan/
            ├── request.go     [MODIFY] add eventGrainEligibility field to response struct
            ├── reconcile.go   [MODIFY] buildMatchObj sets row_level_restricted + event_grain_eligibility via SetDynamicField; enforce-from gating
            └── *_test.go      [NEW/MODIFY] fail-closed truth table + enforce-from gating tests
<UNDETERMINED REPO>/  BQR write-time split          — OQ-2
<UNDETERMINED REPO>/  AGG aggregate routing (scoped SQL) — OQ-2/OQ-4
<UNDETERMINED REPO>/  bqgolden golden-image provisioning — OQ-4
<UNDETERMINED REPO>/  MYD amazonadssan detail-worker case — OQ-5
<UNDETERMINED REPO>/  PBK postback/CAP sanitization        — OQ-5/OQ-7
<UNDETERMINED REPO>/  QRY unredacted-dataset blacklist      — OQ-5
<UNDETERMINED REPO>/  Sanitization config (require_restriction_flag, enforce-from, keypath rows) — OQ-5
grafana-mimir-rules/  EXCLUDED — not an affected system (investigation CA-P1/P2/P3)
```
**Structure Notes**: watson is the only repository with a verified code home. All other paths are placeholders pending repository identification. `grafana-mimir-rules` is conclusively excluded (0 schema, 0 API, 0 required file changes).

### Phase 0: Repository Discovery & Test Coverage Foundation
**Goal**: Locate and check out the owning repositories for BQR/AGG/MYD/PBK/QRY/config/bqgolden; establish baseline test coverage for watson code being modified.
**Prerequisites**: None (this phase is gated on resolving OQ-2, OQ-4, OQ-5)
- [ ] Resolve OQ-2/OQ-4/OQ-5: obtain checkouts of ingestion-adjacent BQ split, aggregate readers, bqgolden, detail workers, postback engine, query engine, sanitization-config model
- [ ] Resolve OQ-3: confirm `watson-proto` untouched and top-level dynamic field carries cleanly through every stored/serialized claim copy
- [ ] Write/verify unit tests for `buildMatchObj` and `response` parsing paths being modified in watson
- [ ] Verify existing watson tests pass
...

## Implementation Guide

1. Implement: Run tests & validate build
2. Add or update the tests listed under Files to Modify
3. Run `go test ./...`

## Files to Modify

| File | Create/Modify/Test | Note |
|---|---|---|
| (none captured, check the plan) | - | - |

## Acceptance Criteria

- Build and existing tests still pass.
- Existing tests pass
- New functionality has test coverage
- Code follows repository conventions (see CLAUDE.md)

## Dependencies

**Blocked by:**
- T402 - Deploy unredacted_app_N golden image: run the bqgoldenutil sync job for all apps

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/bqgolden` tasks in one PR on branch `impl/wi_dry088_a2-backend`.

---

**Parent Task:** [DRY RUN] Amazon Ads eventGrainEligibility Row-Level Redaction — Kochava/bqgolden (3 tasks · 1 PR)
**PRD:** 088-amazon-ads-eventgraineligibility-row-level-redacti