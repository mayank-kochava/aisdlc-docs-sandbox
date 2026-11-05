# TG6: Run tests & validate build

## In short

Run the full build and tests for Kochava/koch-query-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG6 |
| Repo | Kochava/koch-query-api |
| Phase | Phase 2 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/koch-query-api with every task above merged. Do not edit source.

This task is part of Amazon Ads eventGrainEligibility Row-Level Redaction.

### koch-query-api
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Blacklist entry `"unredacted"` | Modify | L | `"_unredacted"` would miss `unredacted_app_N`. Closes an existing exposure for Snap/TikTok/Reddit; ships first |
Test: extend `TestBlacklist_RestrictedDatasetBlocked` (`unredacted_app_5.installs_primary`, `account_5_unredacted.analytics` blocked; `app_5.installs_primary` allowed).

## Technical Context
**Language/Version**: Go (watson, soccer-v2, sanitization-api, bqgolden, rollup-worker-cloud, analytics-query, app-match-api, koch-query-api, krepdetailworkerv14 `main` and `mysql` branches, krepdetailworkerv13); PHP 7.0 (Control); BigQuery SQL (Superset dataset yamls).
**Primary Dependencies**: watson claim build (`SetDynamicField`); soccer-v2 keypath walker; sanitization-api config + keypath rows; portal → bbq write of `app_N` / `unredacted_app_N`; Control `Postback::Sanitize`.
**Storage**: BigQuery redacted `app_N` / `account_N.analytics` and unredacted `unredacted_app_N` / `account_N_unredacted.analytics`; MySQL `clicks`, `impressions`, `attribution`, `event_attribution` (raw, unchanged). One new nullable BOOL column `require_restriction_flag` on `network_sanitization.configuration`. No `watson-proto` change.
**Testing**: Lean, test-first, 1-2 tests per core case, each repo's CI command (per change below). Control tests run in the `control-cover:ci` image (PHP 7.0). Detail workers have no CI: run locally and paste output into the PR. QA capture + QA E2E gate enforcement.
**Performance Goals**: soccer gate is one bool lookup per postback (events dominate: ~127M Amazon-attributed events / 7 days on app 57343). Detail-worker check is a string check over already-loaded blob columns.
**Constraints**: Fail-closed at ingestion; 13954 only; inert until enforce-from; no backfill (old claims never carry the flag); every existing sanitized network byte-identical.

### Source Code Changes
```text
watson/cmd/watson/amazonadssan/
├── request.go            [MODIFY] EventGrainEligibility json.RawMessage on response
├── reconcile.go          [MODIFY] parseEligibility, enforce-from check, dynamic fields in buildMatchObj, mock block (mockResponse :160-225)
└── reconcile_test.go     [MODIFY] eligibility + enforcement tests
watson config/env         [MODIFY] AMAZON_ADS_EGE_ENFORCE_FROM
sanitization-api/internal/dao/bqnetworksanitization/structs.go  [MODIFY] require_restriction_flag
sanitization-api create/update/get handlers + BQ migration       [MODIFY/NEW] nullable BOOL column
soccer-v2/internal/httpclients/sanitization/models/sanitization.go [MODIFY] RequireRestrictionFlag
soccer-v2/cmd/soccer-v2/main.go (Handle :261-300)                  [MODIFY] conditional sanitization
soccer-v2/internal/sanitization/sanitize.go                        [MODIFY, only if QA capture needs it] typed non-string leaves, repeated fields
control/source/var/www/lib/KochavaControl/Event/Postback.php       [MODIFY] keypath path (:1640-1695, retrieveKeyPaths :1688) honors the flag
control/.../SanitizationApiClient.php                              [MODIFY] read require_restriction_flag
control/tests/lib/KochavaControl/Event/Postback_test.php           [MODIFY]
krepdetailworkerv14 branch mysql (cmd/krepdetailworkerv14/main.go, sanitize.go)  [MODIFY] per-row flag check
krepdetailworkerv14 main (legacy RunReportGen path)                             [MODIFY] same
krepdetailworkerv13 (src/krepdetailworkerv13/main.go)                           [MODIFY] same
koch-query-api/cmd/query/blacklist.go (:3-37) + blacklist_test.go  [MODIFY] "unredacted" entry
bqgolden (OPTIONAL, only if the rollup missing-table guard is dropped)
├── app_golden_image.go (:52-72)                          [MODIFY] CoreUnredactedGoldenImage → unredacted_app_%d
├── cmd/bqgoldenutil/config/app_golden_image.go (:5-29)   [MODIFY] flag golden-image-core-unredacted
└── cmd/bqgoldenutil/subcmd/app_golden_images.go (:10-32) [MODIFY] explicit list + all-tables fallback
rollup-worker-cloud/cmd/rollup-worker-cloud/buildquery.go          [MODIFY] src() UNION ALL for installs/san_claims/events
rollup-worker-cloud/cmd/rollup-worker-cloud/main.go (:724)         [MODIFY] pass ReadProjectID + app id separately
rollup-worker-cloud/cmd/rollup-worker-cloud/buildquery_test.go     [NEW] (repo has no tests today)
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
- T301 - Blacklist unredacted_* datasets in the query API

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/koch-query-api` tasks in one PR on branch `impl/prd089-backend`.

---

**Parent Task:** Amazon Ads eventGrainEligibility Row-Level Redaction — Kochava/koch-query-api (2 tasks · 1 PR)
**PRD:** 089-amazon-ads-eventgraineligibility-row-level-redacti