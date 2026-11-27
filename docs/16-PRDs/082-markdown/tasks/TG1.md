# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/links-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/links-api |
| Phase | Phase 2 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/links-api with every task above merged. Do not edit source.

This task is part of PRD 082 slice: attribution settings tables (E2E dry run).

## Repository: Kochava/links-api
**Technology**: Go, MySQL


## Implementation Guide

1. Implement: Run tests & validate build
2. Add or update the tests listed under Files to Modify
3. Run `go test -tags=unit ./...`

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
- T003 - Create the cross_platform_attribution_settings table migration
- T004 - Create the toggle_audit_log table migration

**Blocks:**
- none

## Testing Notes

```bash
go test -tags=unit ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/links-api` tasks in one PR on branch `impl/prd082-backend`.

---

**Parent Task:** PRD 082 slice: attribution settings tables (E2E dry run) — Kochava/links-api (3 tasks · 1 PR)
**PRD:** local/082-markdown