# TG3: Run tests & validate build

## In short

Run the full build and tests for Kochava/app-match-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG3 |
| Repo | Kochava/app-match-api |
| Phase | Phase 4 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/app-match-api with every task above merged. Do not edit source.

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
- T801 - Add account_N_unredacted analytics fallback to the direct app match lookup

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/app-match-api` tasks in one PR on branch `impl/wi_dry088_a2-backend`.

---

**Parent Task:** [DRY RUN] Amazon Ads eventGrainEligibility Row-Level Redaction — Kochava/app-match-api (2 tasks · 1 PR)
**PRD:** 088-amazon-ads-eventgraineligibility-row-level-redacti