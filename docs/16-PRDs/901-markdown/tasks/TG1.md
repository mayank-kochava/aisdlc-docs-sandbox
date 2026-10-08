# TG1: Run tests & validate build

## In short

Run the full build and tests for mayank-kochava/coding-agent-sandbox with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | mayank-kochava/coding-agent-sandbox |
| Phase | Phase 6 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for mayank-kochava/coding-agent-sandbox with every task above merged. Do not edit source.

This task is part of URL Shortener (practice PRD 901).

## Engineering plan
- **Repository:** `mayank-kochava/coding-agent-sandbox` (Go module `github.com/mayank-kochava/coding-agent-sandbox`, Go 1.22). Every task targets this repo.
- **Stack:** Go standard library only (`net/http`, `encoding/json`, `crypto/sha256`, `sync`). No new dependencies, no `go.mod` changes.
- **Packages:**
  - `shortener/store.go`: in-memory `Store` (map + `sync.RWMutex`) with `Save(url) (code string)` and `Lookup(code) (url string, ok bool)`. Code = first 6 chars of base62-encoded sha256 of the URL (deterministic, so FR-4 holds); on a collision with a different URL, append a counter and re-hash.
  - `shortener/validate.go`: `ValidateURL(raw string) error` for FR-2.
  - `shortener/handler.go`: `NewHandler(*Store) http.Handler` routing `/shorten`, `/r/{code}`, `/healthz` (use Go 1.22 `http.ServeMux` method+path patterns).
  - `cmd/shortener/main.go`: listens on `:8081` (env `PORT` overrides), graceful shutdown on SIGINT/SIGTERM.
- **Testing:** table-driven unit tests for store and validate; `net/http/httptest` tests for every handler path and status code. `go test ./...` must pass; `go vet` clean.
- **Order:** store and validate first (independent), then handler (depends on both), then main (depends on handler), then a short README section.
- **Out of scope:** persistence, auth, metrics, custom codes, expiry.


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
- T004 - Add cmd/shortener/main.go with PORT override and graceful shutdown Files: - Create: cmd/shortener/main.go - Create…
- T007 - Document the service in README.md Files: - Modify: README.md (append the section if the file exists, otherwise create…

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `mayank-kochava/coding-agent-sandbox` tasks in one PR on branch `impl/prd901-backend`.

---

**Parent Task:** URL Shortener (practice PRD 901) — mayank-kochava/coding-agent-sandbox (8 tasks · 1 PR)
**PRD:** local/901-markdown