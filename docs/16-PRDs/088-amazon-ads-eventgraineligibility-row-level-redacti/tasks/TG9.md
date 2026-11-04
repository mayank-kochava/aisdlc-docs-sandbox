# TG9: Run tests & validate build

## In short

Run the full build and tests for Kochava/rollup-worker-cloud with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG9 |
| Repo | Kochava/rollup-worker-cloud |
| Phase | Phase 4 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/rollup-worker-cloud with every task above merged. Do not edit source.

This task is part of [DRY RUN] Amazon Ads eventGrainEligibility Row-Level Redaction.

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
- T501 - Route network 13954 rows to unredacted_app_N in SUMMARY and DETAIL rollup queries

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/rollup-worker-cloud` tasks in one PR on branch `impl/wi_dry088_a2-backend`.

---

**Parent Task:** [DRY RUN] Amazon Ads eventGrainEligibility Row-Level Redaction — Kochava/rollup-worker-cloud (2 tasks · 1 PR)
**PRD:** 088-amazon-ads-eventgraineligibility-row-level-redacti