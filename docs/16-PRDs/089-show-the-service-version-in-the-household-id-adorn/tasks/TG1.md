# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/household-id-adornment-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/household-id-adornment-api |
| Phase | Phase 2 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/household-id-adornment-api with every task above merged. Do not edit source.

This task is part of Product: K4A

Title: Show the service version in the household ID adornment API .

### household-id-adornment-api (aka `uadd`) — PRIMARY
**Technology**: Go service (`urfave/cli`, `gorilla/mux`, kotel/OTel), verified
**Key Files**:
- health check handler (`internal/healthcheck/healthcheck.go` or `internal/handlers/healthcheck.go`) — the handler being modified
- `cmd/<app>/main.go` and/or `internal/commands/server/server.go` — version wiring site (`cliCtx.App.Version` / `gopkgver.Version` in scope)
- `internal/commands/reader/reader.go` — second call site **if** the handler is shared (one investigation state only)
- `Dockerfile` — ldflags target **if** still broken (one investigation state only)
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Confirm current HEAD state (handler body, ldflags target, middleware) | New (investigation) | L | Phase 0 gate — resolves the state conflict |
| Fix Dockerfile ldflags `-X 'main.Version=...'` → `-X '${PKG_VER_PATH}=...'` | Modify | L | **ONLY if still broken.** Matches geoip-adornment-api reference pattern |
| Add `version string` param to health check handler + write it additively | Modify | L | Preserve any existing body prefix; or introduce body/header per current state |
| Pass version at registration/construction site(s) | Modify | L | `cliCtx.App.Version` / `gopkgver.Version` already in scope |
| Update `reader` call site to compile | Modify | L | **ONLY if handler is shared** |
| Add `"unknown"` sentinel fallback + WARN log when version empty/`"dev"` | Modify | L | Reuse existing `host="unknown"` idiom; see DEC-003 |
| Add `healthcheck_test.go` (happy path, fallback, non-breaking assertion) | New | M | Package currently has zero tests |
| Post-deploy smoke check (version == deployed tag) | New | L | NFR-011 correctness guard; lives in CI/CD |

## Summary
Expose the deployed release tag as an additive element of the `/healthcheck` response in the household ID adornment API. The compliant approach is to fix the build-time version-stamping chain (ldflags), thread the already-available `gopkgver.Version` package global into the health check handler, and surface it in the response while preserving the existing response behavior — with an explicit `"unknown"` sentinel fallback when the version cannot be resolved.
> **⚠️ Material evidence conflict the implementer MUST resolve first.** Two live investigation passes of what appears to be the owning service repo disagree on the current code state, and this disagreement changes the scope:
> - One investigation (reported as `household-id-adornment-api`) found the health check returns plain-text `"OK - %s"`, the Dockerfile ldflags target is **broken** (`-X 'main.Version=...'` against a nonexistent symbol), and the handler is shared with a `reader` command.
> - A second investigation (reported as `uadd`, the same deployed artifact per the deploy repos) found the health check handler writes **no body at all** (bare `WriteHeader(200)`), the Dockerfile ldflags target is **already correct** (`-X '${PKG_VER_PATH}=...'`), and `gopkgver.VersionMiddleware` is **already applied to every route** via `router.Use(...)`.
>
> These cannot both be the current `main` state of the same running service. The most likely explanation (noted in the `uadd` pass) is a recent merge that corrected the ldflags and changed the handler. **Phase 0 of this plan is to pin the actual current state of the deployed service's HEAD before writing feature code.** The affected-systems and phase tasks below are written to be correct under *either* state, with the divergent sub-tasks called out explicitly. See [OQ: current-repo-state] in the Register.

### Source Code Changes (owning service repo — exact paths pending Phase 0 state confirmation)
```text
household-id-adornment-api (aka uadd) /
├── Dockerfile                                   [MODIFY — only if ldflags target is still broken]
├── cmd/<app>/main.go                            [MODIFY — pass version into handler constructor/registration]
├── internal/healthcheck/healthcheck.go          [MODIFY — add version param; write additively]
│   (OR internal/handlers/healthcheck.go)
├── internal/commands/server/server.go           [MODIFY — pass cliCtx.App.Version (if this layout)]
├── internal/commands/reader/reader.go           [MODIFY — compile-forced (if handler is shared)]
└── internal/healthcheck/healthcheck_test.go     [NEW — first tests for this package]
    (OR internal/handlers/healthcheck_test.go)
```
**Structure Notes**: The two investigation states describe two slightly different package layouts (`internal/healthcheck/*` vs `internal/handlers/*`). The implementer must confirm the live layout in Phase 0 and apply the change in whichever is real. The logical change is identical either way.
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
- T001 - Fix Dockerfile ldflags so release tag reaches gopkgver.Version
- T002 - Health handler appends version with "unknown" fallback; update both call sites

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/household-id-adornment-api` tasks in one PR on branch `impl/prd089-infra`.

---

**Parent Task:** Product: K4A

Title: Show the service version in the household ID adornment API  — Kochava/household-id-adornment-api (3 tasks · 1 PR)
**PRD:** 089-show-the-service-version-in-the-household-id-adorn