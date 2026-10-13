# TG1: Run tests & validate build

## In short

Run the full build and tests for mayank-kochava/coding-agent-sandbox with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | mayank-kochava/coding-agent-sandbox |
| Phase | Phase 4 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for mayank-kochava/coding-agent-sandbox with every task above merged. Do not edit source.

This task is part of Rate Limiter (practice PRD 905).

## Repository: mayank-kochava/coding-agent-sandbox
**Technology**: Go 1.22
**Task Range**: T001-T099

## Engineering plan
- **Repository:** `mayank-kochava/coding-agent-sandbox` (Go module `github.com/mayank-kochava/coding-agent-sandbox`, Go 1.22). Every task targets this repo.
- **Design:** one struct holding `tokens float64`, `last time.Time` and a `sync.Mutex`. Each call refills `min(burst, tokens + elapsed*rate)`, then spends. Time comes through a `Clock` interface so tests use a fake clock and never sleep.
- **Stack:** Go standard library only. No new dependencies, no `go.mod` changes.
- **Testing:** table-driven tests with a fake clock; `go test -race ./...` passes, `go vet` clean.


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
- T003 - Runnable example
- T004 - Window boundary and idle-reset edge-case tests
- T005 - Independent limiters and invalid-limit edge-case tests

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `mayank-kochava/coding-agent-sandbox` tasks in one PR on branch `impl/prd905-backend`.

---

**Parent Task:** Rate Limiter (practice PRD 905) — mayank-kochava/coding-agent-sandbox (6 tasks · 1 PR)
**PRD:** local/905-markdown