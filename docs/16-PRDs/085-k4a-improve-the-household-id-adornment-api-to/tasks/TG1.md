# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/household-id-adornment-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/household-id-adornment-api |
| Phase | Phase 5 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/household-id-adornment-api with every task above merged. Do not edit source.

This task is part of K4A: Improve the household-id-adornment-api to provide.

### household-id-adornment-api
**Technology**: Go 1.22 gRPC service; sole outbound dependency Dragonfly (investigation-verified).
**Key Files**:
- `internal/householdserver/household.go` — all three empty-result sites + batch loop (CA-P3/CA-P4)
- `internal/householdserver/household_test.go` — existing testcontainers suite (CA-7)
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Change `GetHousehold` empty path from `(nil, nil)` to populated-empty message + reason | Modify | M | Subtlest backward-compat risk — a caller doing `if resp == nil` behaves differently (see Register) |
| Map `redis.Nil`→no-match, post-retry `err != nil`→dependency-unavailable at both cause sites | Modify | L | Control flow already distinguishes these |
| Net-new device-path input validation for "invalid input format" | New | M | `HashDeviceIDAndType` returns no error today; "malformed" undefined (see Register) |
| Restructure batch loop to stop swallowing per-item causes (if per-input required) | Modify | M/H | Currently logs + `continue`s, drops nils; gated on batch-granularity decision |
| New bounded-retry wrapper (3 attempts, exp backoff, honor `Retry-After`) around Dragonfly calls | New | M | FR-009/NFR-002; single-shot today |
| Ensure no raw id/ip in reason content or logs; follow hash-before-log; do not resurrect `household.go:40` | Modify | L | FR-007/FR-008/NFR-004/005 |
| Add reason + no-PII assertions to testcontainers tests | Modify | M | Table-driven; IP edge-case tests are a template |

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
- T005 - Carry per-item reasons through the GetHouseholds batch loop

**Blocks:**
- none

## Testing Notes

```bash
go test ./...
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/household-id-adornment-api` tasks in one PR on branch `impl/prd085-backend`.

---

**Parent Task:** K4A: Improve the household-id-adornment-api to provide — Kochava/household-id-adornment-api (5 tasks · 1 PR)
**PRD:** 085-k4a-improve-the-household-id-adornment-api-to