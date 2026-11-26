# TG2: Run tests & validate build

## In short

Run the full build and tests for Kochava/household-id-adornment-proto with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG2 |
| Repo | Kochava/household-id-adornment-proto |
| Phase | Phase 2 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/household-id-adornment-proto with every task above merged. Do not edit source.

This task is part of K4A: Improve the household-id-adornment-api to provide.

### household-id-adornment-proto
**Technology**: proto3 + dual Go/Python codegen; `semantic-release` from v1.0.3 (investigation-verified).
**Key Files**:
- `pkg/householdservice/household_service.proto` — the one hand-edited schema file (CA-2)
- `go/` + `python/` generated stubs — regenerated, never hand-edited (CA-4/CA-6)
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Add `enum NoResultReason` (repo's first enum, `_UNSPECIFIED=0` + three named values) | New | M | `NumEnums` 0→1; `_UNSPECIFIED` doubles as backward-compat signal |
| Add `reason` field at tag 4 on `KeyedHousehold` | New | L | Serves both RPCs; field 4 confirmed free |
| Optional envelope reason at `GetHouseholdsResponse` tag 2 | New | L | Likely required for request-wide dependency-unavailable in batch — gated on batch-granularity decision (see Register) |
| Regenerate Go + Python stubs via `make protoc-compile`; commit both | Modify | L | **Linux regen must work around `sed -i ''` at `Makefile:49`** |
| Release as `feat:` → minor bump (v1.0.3 → v1.1.0) | New | L | Tags both `go/` and `python/` |

## Summary
Enhance the `household-id-adornment-api` no-results path so every empty lookup carries exactly one distinct, named, privacy-safe diagnostic reason — 'no match found', 'invalid input format', or 'lookup dependency unavailable'. The reason is introduced from scratch as the proto contract's first `enum` plus a new field (tag 4) on `KeyedHousehold` in `household-id-adornment-proto`, consumed by the Go service which maps two of the three reasons onto existing control-flow branches, adds net-new device-path input validation for the third, and wraps Dragonfly calls in bounded retry (verified live across both repos).

## Technical Context
**Language/Version**: Go 1.22 (`household-id-adornment-api`); proto3 + dual Go/Python codegen (`household-id-adornment-proto`) — investigation-verified.
**Primary Dependencies**: gRPC, Dragonfly (Redis-protocol KV, sole outbound dependency), kotel (OTel), kolog, Prometheus, redisotel — all investigation-verified; no new dependency introduced.
**Storage**: No persistent-store change. Dragonfly KV schema untouched; the diagnostic reason is computed at request time, never stored (investigation-verified, api repo CA-P1 / avro-dragonfly-loader CA-1–CA-3).
**Testing**: Existing `household_test.go` testcontainers (`redis:7`) + `stretchr/testify` suite, extended with table-driven reason assertions (investigation-verified, api CA-7).
**Target Platform**: Kubernetes via external `ko-k8s-apps/adornment` — KEDA-autoscaled 8–32 replicas in prod, method-agnostic Envoy prefix routing (investigation-verified, ko-k8s-apps CA-2/CA-5).
**Performance Goals**: Hold current success-path p95 as the non-regression baseline (NFR-001); the baseline number and its monitoring source are not supplied and must be confirmed (OQ, below).
**Constraints**: Additive-only contract change (NFR-007); no raw PII in responses or logs (NFR-004/005); never collapse causes into one generic reason (FR-005/NFR-010).

### Source Code Changes
```text
household-id-adornment-proto/
├── pkg/householdservice/household_service.proto            [MODIFY] new enum + field 4
├── go/householdservice/household_service.pb.go             [REGEN]
├── go/householdservice/household_service_grpc.pb.go        [REGEN, no functional change]
├── python/householdservice/household_service_pb2.py        [REGEN]
├── python/householdservice/household_service_pb2.pyi       [REGEN]
└── python/householdservice/household_service_pb2_grpc.py   [REGEN, no functional change]
household-id-adornment-api/
├── internal/householdserver/household.go                   [MODIFY] reason at 3 cause sites + batch + net-new validation + retry
├── internal/householdserver/<retry wrapper>.go             [NEW] bounded-retry around Dragonfly calls
└── internal/householdserver/household_test.go              [MODIFY] reason + no-PII assertions
ko-k8s-apps/adornment/household-id/api/                      [MODIFY, rollout only]
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
- T001 - Add NoResultReason enum and KeyedHousehold.reason (tag 4); regenerate Go and Python stubs

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/household-id-adornment-proto` tasks in one PR on branch `impl/prd085-backend`.

---

**Parent Task:** K4A: Improve the household-id-adornment-api to provide — Kochava/household-id-adornment-proto (2 tasks · 1 PR)
**PRD:** 085-k4a-improve-the-household-id-adornment-api-to