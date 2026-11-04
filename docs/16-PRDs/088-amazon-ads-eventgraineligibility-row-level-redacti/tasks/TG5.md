# TG5: Run tests & validate build

## In short

Run the full build and tests for Kochava/control with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG5 |
| Repo | Kochava/control |
| Phase | Phase 2 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/control with every task above merged. Do not edit source.

This task is part of Amazon Ads eventGrainEligibility Row-Level Redaction.

## Technical Context
**Language/Version**: Go (watson, soccer-v2, sanitization-api, bqgolden, rollup-worker-cloud, analytics-query, app-match-api, koch-query-api, krepdetailworkerv14 `main` and `mysql` branches, krepdetailworkerv13); PHP 7.0 (Control); BigQuery SQL (Superset dataset yamls).
**Primary Dependencies**: watson claim build (`SetDynamicField`); soccer-v2 keypath walker; sanitization-api config + keypath rows; portal → bbq write of `app_N` / `unredacted_app_N`; Control `Postback::Sanitize`.
**Storage**: BigQuery redacted `app_N` / `account_N.analytics` and unredacted `unredacted_app_N` / `account_N_unredacted.analytics`; MySQL `clicks`, `impressions`, `attribution`, `event_attribution` (raw, unchanged). One new nullable BOOL column `require_restriction_flag` on `network_sanitization.configuration`. No `watson-proto` change.
**Testing**: Lean, test-first, 1-2 tests per core case, each repo's CI command (per change below). Control tests run in the `control-cover:ci` image (PHP 7.0). Detail workers have no CI: run locally and paste output into the PR. QA capture + QA E2E gate enforcement.
**Performance Goals**: soccer gate is one bool lookup per postback (events dominate: ~127M Amazon-attributed events / 7 days on app 57343). Detail-worker check is a string check over already-loaded blob columns.
**Constraints**: Fail-closed at ingestion; 13954 only; inert until enforce-from; no backfill (old claims never carry the flag); every existing sanitized network byte-identical.

### Confirmed Assumptions
| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| watson owns the eligibility decision | `watson cmd/watson/amazonadssan` (`request.go:3-9` response, `reconcile.go:438-538` `buildMatchObj`) | Today watson drops unknown response keys (`reconcile.go`), so the field can't be observed until parsed |
| A write-time redacted/raw split exists for Snap/TikTok/Reddit | `soccer-v2 cmd/soccer-v2/main.go:261-300` `Handler.Handle` | `postbacks` (sanitized) + `postbacks_unredacted` (original) when config has `SanitizeKeypaths` |
| No proto change; the flag reaches every stored copy | `control Attribution.php populateFromWatsonClaim`, `SelfAttributing.php` `json_encode($claim)`, `RawAttribution.php:138-153,298-317`, `AttributionDetails.php:169-186` | Whole claim stored as-is; later events merge every claim key. Prod rows already carry watson dynamic fields (`partner_request_id`, `attributed_by_ip`) |
| Keypath walker reaches the Amazon fields | `soccer-v2 internal/sanitization/sanitize.go:80-160` | Reaches typed messages by reflection, then one key into a Struct; `properties` / `identifiers` reachable (seen redacted on a prod Reddit row) |
| A missing unredacted table fails the reader | BigQuery semantics | No fallback to the redacted copy, so bqgolden must sync first |
| Every Control receiver rides the soccer payload | `Postback.php:2106-2128` | `bigquery`, `analytics`, `collective`, `cookie`; soccer covers them all |
| Superset drill-to-detail is off in prod | Superset prod config | No drill-down exposure from `unredacted_*` |

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
...

## Implementation Guide

1. Implement: Run tests & validate build
2. Add or update the tests listed under Files to Modify
3. Run `vendor/bin/phpunit`

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
- T014 - PBK: Apply the postback keypaths only to flagged conversions when the option is set

**Blocks:**
- none

## Testing Notes

```bash
vendor/bin/phpunit
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/control` tasks in one PR on branch `impl/wi_a8a2f1cb9bf4c32e30c5-backend`.

---

**Parent Task:** Amazon Ads eventGrainEligibility Row-Level Redaction — Kochava/control (2 tasks · 1 PR)
**PRD:** 088-amazon-ads-eventgraineligibility-row-level-redacti