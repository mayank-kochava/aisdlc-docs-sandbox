# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/household-id-adornment-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/household-id-adornment-api |
| Phase | Phase 3 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/household-id-adornment-api with every task above merged. Do not edit source.

This task is part of Product: K4A

Title: Show the service version in the household ID adornment API .

### household-id-adornment-api / uadd (the service binary — PRIMARY, and only required code change)
**Technology**: Go gRPC/HTTP service; `gopkgver` build-time version; `gorilla/mux`+`ochttp` (uadd shape) or `DefaultServeMux` (household-id shape)
**Key Files** (superset across both candidate shapes):
- health handler (`internal/handlers/healthcheck.go` or `internal/healthcheck/healthcheck.go`) — produces the health response
- command wiring (`cmd/uadd/main.go:394` or `server.go:166` + `reader.go:154`) — version source `gopkgver.Version` / `cliCtx.App.Version`
- `Dockerfile` (conditional) — ldflags injection target
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Thread version into health handler (field/param) | Modify | L | `NewHealthCheckHandler(version string)` or `HealthHandler(version string)` |
| Write version to health response body | Modify | L | Append preserving status 200 and any existing prefix (BR-6). Empty-body shape: version becomes first body content |
| Pass version at construction/registration site(s) | Modify | L | One site (uadd) or two sites (household-id — do NOT miss the reader site) |
| Add `healthcheck_test.go` | New | L | `httptest`: assert version present + prefix/status preserved |
| Fix Dockerfile ldflags (conditional) | Modify | L | Only if authoritative repo is the household-id checkout with the `main.Version` mismatch (DEC-001) |
| Explicit `dev` fallback in code (conditional) | Modify | L | Only if the authoritative repo's `gopkgver.Version` default is not confirmed `dev`; verified `dev` in the gopkgver source |

## Technical Context
**Language/Version**: Go (investigation-verified; `go.mod` declares Go 1.24 in the live uadd checkout — note eng-research.md states 1.22; see Corrected Assumptions below)
**Primary Dependencies**: `urfave/cli`, `gorilla/mux` + `go.opencensus.io/plugin/ochttp` (HTTP routing), `github.com/Kochava/gopkgver` (build-time version), `github.com/Kochava/kotel` (telemetry — unaffected)
**Storage**: N/A — no persistence touched (data-path packages untouched)
**Testing**: `testify` (+ `testcontainers` elsewhere in repo); new `httptest` unit test for the health handler
**Target Platform**: Kubernetes (deployed via `ko-k8s-apps` under `adornment/household-id/`; `/healthcheck` on container port 8080 at deploy layer, served by the binary on `:8787` per uadd checkout — port discrepancy noted below)
**Performance Goals**: NFR-001 — no material change to health p95 (version resolved once at startup, appended as a constant string per request)
**Constraints**: Additive/non-breaking only (BR-6/FR-006/NFR-006); no access-control changes (NFR-004); only a bare version string or `dev` exposed (NFR-005)
> **Critical repo-identity note for implementation.** Two live checkouts were investigated that both plausibly represent "the household ID adornment API": `household-id-adornment-api` (older/different state — empty subcommands structure, two call sites, broken ldflags) and `uadd` (`github.com/Kochava/uadd` — Go 1.24, single `app.Action`, one call site, **working** ldflags, `gopkgver.VersionMiddleware` already installed). The `ko-k8s-apps` investigation confirms the deployed image is `kochava/household-id-adornment-api` under `adornment/household-id/`, run with `server`/`reader` args and probing `/healthcheck:8080` — which matches the `household-id-adornment-api` shape, NOT the `uadd` single-action shape. Which source checkout is the one that actually builds the deployed `kochava/household-id-adornment-api` image is unresolved and must be confirmed before coding. [DECISION REQUIRED] (see Register — OQ for repo identity). The plan below covers both shapes; the Affected Systems table enumerates the superset of touched files.

### Confirmed Assumptions
| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| household-id-adornment-api owns the health endpoint | Investigation (uadd `main.go:403`; household-id `server.go:166`/`reader.go:154`) | Verified in both candidate checkouts. |
| Change is additive/non-breaking (BR-6/FR-006/NFR-006) | Investigation (plain-text/empty body; prefix/status preservable) | Appending text keeps status 200 and any existing prefix. |
| Version value available in-process | Investigation (`gopkgver.Version`; `app.Version = gopkgver.Version`) | Reuse existing value; no new flag/env. |
| Adding version adds no per-request work (NFR-001) | Investigation — static startup value | Resolved once at startup. |
| No programmatic consumer breaks (NFR-006) | Investigation (watson `Ping()` reads status code only, never body) | Body append is transparent to the one real programmatic consumer. |
| CI already injects real semver on release; omits on PR → `dev` (BR-3/BR-4) | Investigation (github-workflows `go_app_release.yml:130`, `go_app_pull_requests.yml` passes no `VERSION`) | Build-time `VERSION` mechanism already works at the CI layer. |
| `gopkgver.Version` default is literal `dev` (BR-4) | Investigation (gopkgver `version.go:5-6` `var Version = "dev"`) | BR-4 fallback is the built-in default; no code-level fallback required if `gopkgver` is the source. |

### Corrected Assumptions
| Spec Statement | Actual Reality | Impact |
|----------------|----------------|--------|
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
- T003 - Run a job: deploy built images locally and check connectivity with curl to /healthcheck, record DEC-003 sign-off in PR

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/household-id-adornment-api` tasks in one PR on branch `impl/prd089-backend`.

---

**Parent Task:** Product: K4A

Title: Show the service version in the household ID adornment API  — Kochava/household-id-adornment-api (4 tasks · 1 PR)
**PRD:** 089-product-k4a-title-show-the-service-version-in-the-