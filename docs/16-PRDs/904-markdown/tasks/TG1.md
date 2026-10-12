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

This task is part of LRU Cache (practice PRD 904).

## Repository: mayank-kochava/coding-agent-sandbox
**Technology**: Go 1.22
**Task Range**: T001-T099

## Engineering plan
- **Repository:** `mayank-kochava/coding-agent-sandbox` (Go module `github.com/mayank-kochava/coding-agent-sandbox`, Go 1.22). Every task targets this repo.
- **Design:** hash map from key to a node of a doubly linked list. The list head is the most recently used entry and the tail the least; evict from the tail. Write the list by hand (no `container/list`) so the design is visible.
- **Stack:** Go standard library only. No new dependencies, no `go.mod` changes.
- **Testing:** table-driven tests; `go test ./...` passes, `go vet` clean.


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

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `mayank-kochava/coding-agent-sandbox` tasks in one PR on branch `impl/prd904-backend`.

---

**Parent Task:** LRU Cache (practice PRD 904) — mayank-kochava/coding-agent-sandbox (4 tasks · 1 PR)
**PRD:** local/904-markdown