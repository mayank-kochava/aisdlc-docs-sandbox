# Research: K4A — Enhanced Error Reporting for Household ID Adornment Lookups

**Feature**: K4A / Enhanced Error Reporting for Household ID Adornment Lookups
**Date**: 2026-11-26
**Phase**: 0 - Research & Technology Decisions

## Overview

This research validates the product spec's technical assumptions against three repositories, two of which were investigated live (Read-verified) and one of which was ruled out by direct investigation:

- **`household-id-adornment-api`** (backend, Go 1.22 gRPC service) — the primary implementation surface; investigated live.
- **`household-id-adornment-proto`** (backend, proto3 + gRPC contract) — the wire-contract surface where diagnostic fields land; investigated live.
- **`frontend-mos`** (frontend, Vue 3 console) — the spec's **low-confidence** KG hint for an agent display surface; investigated live and **ruled out** as not present in this repo.

Scope of decisions made here: the contract shape for carrying a diagnostic reason, how the three named reasons map onto existing control flow, where net-new validation is required ("invalid input format"), how bounded retry is introduced, the privacy posture of diagnostic content, backward-compatibility strategy for the single-lookup RPC's bare-nil empty path, and testing approach. Several product-owned trade-offs (consumption surface, reason wording, p95 baseline, batch per-item vs envelope diagnostics) remain unresolved and are carried forward as decisions/open questions.

The headline reality discovered live: the single-lookup empty result is a bare typed-nil (`return nil, nil`) with **no field anywhere** to carry a reason (`NumEnums: 0` in the proto). So this is not "add a field to an existing error object" — it is "introduce a reason concept from scratch and change what empty looks like on the wire for the single-lookup RPC."

---

## Technology Stack Decisions

### Decision: Target stack is the existing Go 1.22 gRPC service + proto3 contract — no new technology introduced

**Chosen**: Implement entirely within the existing `household-id-adornment-api` (Go 1.22, gRPC, Dragonfly/Redis-protocol KV) and `household-id-adornment-proto` (proto3 + gRPC, dual Go/Python codegen) repositories. No new language, framework, datastore, or external dependency.

**Rationale** (investigation findings, Read-verified):
- `go.mod:3`, `README.md:8`, `internal/householdserver/household.go:12-14` confirm Go 1.22, gRPC, Dragonfly as the only outbound dependency. The feature is self-contained to this service + its proto contract + the generated client lib — no cross-service seam to trace for the diagnostic content.
- The proto repo (`household_service.proto:7-40`) exposes exactly two unary RPCs (`GetHousehold`, `GetHouseholds`) — the complete surface the enhancement must extend.
- Observability (kotel/OTel, kolog, Prometheus, redisotel, slog JSON) is already wired (`server.go:95-98,103,119-125`), so NFR-003 alerting likely sits on existing metrics rather than requiring new instrumentation.

**Alternatives Considered**:
1. **Introduce a new RPC to carry the reason (vs. extending existing messages)**
   - Pros: Leaves the single-lookup `(nil, nil)` convention untouched for existing callers.
   - Cons: Doubles the surface; doesn't satisfy "additive to existing response" intent as cleanly; forces consumers to adopt a new method to get diagnostics.
   - Rejected (provisionally): The additive proto-field approach (field 4 on `KeyedHousehold`) is backward-compatible for deserialization per proto3 semantics; a new RPC is a heavier fallback reserved only if a consumer is found to depend on bare-nil (see DEC-003 / Q1 below).

**Implementation Notes**:
- Proto regen via `make protoc-compile` (`README.md:73`, `Makefile:16-20,39-49`); commit both Go and Python stubs together. **Watch the BSD `sed -i ''` gotcha** at `Makefile:49` — macOS-specific, will bite on Linux CI.
- Land the proto change as a conventional-commit `feat:` → `semantic-release` minor bump from current **v1.0.3**.
- `avro-dragonfly-loader` (external module, surfaced by KG and by the backend investigation) provides `householdutils.HashDeviceIDAndType`/`HashIP`; it is **not vendored** in the checkout, so its validation internals are unverified (bears on FR-003 — see Open Questions register).

---

## Architectural Patterns

### Decision: Carry the diagnostic reason as a new proto enum + reason field on `KeyedHousehold` (field 4), covering both RPCs

**Chosen**: Add a new proto3 `enum` enumerating the three named reasons, plus a reason field at **tag 4** on `KeyedHousehold` (`household_service.proto:30-36`) — the single placement that serves both `GetHousehold` and `GetHouseholds`. Optionally a companion actionable-message field, gated on the message-shape decision below.

**Rationale** (Read-verified, proto investigation CA-2/CA-4):
- Fields 1–3 are used on `KeyedHousehold`; **field 4 is the free, backward-compatible slot** — old clients ignore unknown proto3 fields (satisfies BR-5 / FR-010 / NFR-007).
- The contract has **zero** existing enums/oneofs/diagnostic fields today (`NumEnums: 0`, `household_service.pb.go:522`), so the named-reason vocabulary (BR-2: 'no match found', 'lookup dependency unavailable', 'invalid input format') is introduced from scratch as a new enum.
- Per-`KeyedHousehold` placement is the only placement that also serves the single-lookup RPC; an envelope-level field on `GetHouseholdsResponse` (field 2 is free) would cover batch only.

**Alternatives Considered**:
1. **gRPC status codes (`codes.NotFound` / `InvalidArgument` / `Unavailable`)**
   - Pros: Native gRPC mechanism; no proto message change.
   - Cons: Status codes are not currently used in live handlers (CA-6: all `status.`/`codes.` usages are generated code or the one comment); the commented-out `status.Errorf` at `household.go:40` shows this approach was **tried and deliberately reverted**, and its form echoed the raw device id (a BR-4 privacy violation).
   - Rejected: Pre-rejected in-code for good reason; status codes also map awkwardly to a per-item batch reason.
2. **Envelope-level reason only (`GetHouseholdsResponse` field 2)**
   - Pros: Smaller change for batch.
   - Cons: Does not serve the single `GetHousehold` RPC; cannot express per-input causes in a batch.
   - Rejected: Per-`KeyedHousehold` placement is strictly more general.

**Implementation Notes**:
- Proto change sites: new `enum` + `KeyedHousehold` field 4 (`household_service.proto:30-36`).
- The message-shape sub-decision — **enum-only vs. enum + static message string vs. gRPC status code** — is a forced, unresolved contract decision (NFR-007; backend Q3). An enum-only field inherently cannot leak customer data (safest for BR-4/FR-007); a free-text `string` field could. `[DECISION REQUIRED: DEC-008]`

### Decision: Map the three named reasons onto existing control-flow branches; add net-new validation only where it is genuinely absent

**Chosen**: Reuse the control-flow distinctions that already exist for *no-match* vs *dependency-unavailable*, and add net-new input validation for *invalid input format* on the device-id path.

**Rationale** (Read-verified, backend CA-1/CA-2/CA-5):
- `GetHousehold` (`household.go:32-53`): `redis.Nil` branch (line 38) = **no match**; `err != nil` branch (line 42) = **dependency unavailable**. Both already distinguished in control flow, collapsed only at the `(nil, nil)` return.
- `GetHouseholdFromIP` (`household.go:56-82`): same two causes already structurally distinguished (lines 68 vs 72); `HashIP` already returns an error on malformed IPs (partial invalid-input coverage already present).
- `HashDeviceIDAndType` (`household.go:33`, CA-5) returns a **plain string, no error** — any device input hashes to *something* and simply misses. So FR-003 "invalid input format" on the device path requires **net-new validation logic that does not exist today**; without it, "no match" and "invalid input" are indistinguishable on that path.

**Alternatives Considered**:
1. **Collapse all empty outcomes into a single generic reason**
   - Pros: Trivial.
   - Cons: Violates FR-005 / NFR-010 (distinct named reasons must never be collapsed).
   - Rejected outright by the spec.

**Implementation Notes**:
- Two empty-result sites must be covered consistently: `GetHousehold` (CA-1) and `GetHouseholdFromIP` (CA-2).
- What counts as "invalid input format" on the device path (empty `device_id`? bad `device_id_type`? format rules?) is **undefined** and must be specified before validation can be written (backend Q6). `householdutils` internals are unverified (external, not vendored). `[DECISION REQUIRED: DEC-009]`

### Decision: Batch-path per-item diagnostics is a second structural change, gated on a product decision

**Chosen**: Treat whether batch (`GetHouseholds`) needs per-input reasons as an explicit decision; default to per-`KeyedHousehold` reason placement (which naturally supports per-item) but confirm with product before restructuring the aggregation loop.

**Rationale** (Read-verified, backend CA-3):
- `GetHouseholds` (`household.go:84-116`) currently **discards** both errors (logged + `continue`) and nil results; a partial/empty batch returns `(nil, nil)` with nowhere to attach a per-input reason. If FR-002–005 imply per-input diagnostics, this loop is a second, hairier structural change site.

**Implementation Notes**:
- The per-`KeyedHousehold` field-4 placement chosen above already supports per-item reasons structurally; the open question is whether product requires per-input diagnostics or whether envelope-level is acceptable (backend Q2). `[DECISION REQUIRED: DEC-010]`

---

## Performance Considerations

### Decision: Diagnostics touch only the no-results path; hold current success-path p95 as the non-regression baseline

**Chosen**: Confine all new logic (reason derivation, validation, retry) to the no-results/error paths so the happy path (FR-001) is structurally unchanged. Adopt the current production p95 as the regression gate (DEC-005), to be confirmed from existing monitoring.

**Rationale**:
- Backend investigation confirms the happy path returns the matching `KeyedHousehold` directly and is not on the new code path; low regression risk (backend Q5 notes this).
- Observability is already in place (kotel/Prometheus) to measure p95.

**Alternatives Considered**:
1. **Define a fresh absolute latency SLO**
   - Pros: Explicit target.
   - Cons: No baseline number was supplied in the input; inventing one would be a fabrication.
   - Rejected: Use the existing measured p95 as the baseline instead.

**Implementation Notes**:
- The actual p95 number and its monitoring source (kotel/Prometheus dashboard) are **not supplied** and must be confirmed to set the regression gate (NFR-001 / DEC-005 / backend Q5). `[DECISION REQUIRED: DEC-011]`

### Decision: Bounded retry with exponential backoff around Dragonfly calls, honoring Retry-After

**Chosen**: Introduce a new bounded-retry wrapper around the Dragonfly `Get`/`SMisMember` calls — default up to 3 attempts, exponential backoff, honoring any `Retry-After`; only after exhaustion return "lookup dependency unavailable" (FR-004/FR-009/NFR-002).

**Rationale** (Read-verified, backend CA-1/CA-2):
- Current Dragonfly calls are **single-shot** (`dfly.Get` at CA-1, `dfly.SMisMember` at CA-2) — bounded retry does not exist today and is net-new wrapper/middleware logic.
- NFR-002 requires returning the unavailable reason within the retry-bounded time rather than blocking indefinitely.

**Implementation Notes**:
- Wrap both Dragonfly call sites consistently. The "lookup dependency unavailable" reason must only surface after bounded attempts are exhausted (FR-009), and the response must indicate the condition is transient rather than a definitive no-match (FR-004).

---

## Error Handling Strategy

### Decision: Enum-classified named reasons, privacy-safe by construction, following existing hash-before-log conventions

**Chosen**: Classify every empty result into exactly one of three named reasons (FR-005). Describe only the *category* of failure; never echo query values. Follow the existing hash-before-log pattern for any telemetry.

**Rationale** (Read-verified, backend CA-6; proto CA-4):
- Live code already **hashes id/ip before logging** (`household.go:35,62`) — the privacy-safe pattern BR-4/FR-008/NFR-005 require is already established and should be followed.
- The commented-out `status.Errorf` at `household.go:40` echoes the raw device id — the **pre-rejected, privacy-violating** counter-example; do not resurrect it (DEC-002/DEC-007/FR-007/NFR-004).
- An **enum-only** reason inherently cannot carry customer data, making the contract shape itself a privacy control.

**Error Categories** (BR-2 / DEC-001 / DEC-006 named reasons):
- **No match found**: well-formed, valid input, dependency available, no matching record (`redis.Nil` branch, `len(results)==0`). Definitive.
- **Invalid input format**: input malformed / fails format validation (net-new on device path; partial on IP path via `HashIP` error). Must not echo the offending value (FR-007).
- **Lookup dependency unavailable**: Dragonfly call failed/timed out after bounded retry (`err != nil` branch after exhaustion). Transient, not a definitive no-match (FR-004).

**Implementation Notes**:
- Never collapse to a single generic empty/error indication (FR-005 / NFR-010).
- No raw customer-identifying data in returned reasons (NFR-004) or in logs/telemetry (NFR-005) — any troubleshooting identifier must be masked/hashed (the existing pattern). A deliberate privacy review of diagnostic content is a release gate (NFR-011).
- Dependency health alerting (NFR-003): error rate >1% for 5 min sustained — likely an alerting rule on existing kotel/Prometheus metrics, not new instrumentation.

---

## Testing Strategy

### Decision: Extend the existing testcontainers + testify integration tests with reason assertions

**Chosen**: Add table-driven reason assertions to the existing `household_test.go` suite, which already spins up real `redis:7` containers and exercises every empty-result case.

**Rationale** (Read-verified, backend CA-7):
- `household_test.go` (558 lines) uses **testcontainers** (`:29-31,13`) with `stretchr/testify` (`go.mod:17`); every empty-result case is already exercised via `ShouldBeNil` assertions (`:93,216,348-349`). Adding reason assertions extends these directly.
- The IP edge-case tests (`:364-420`) already cover invalid/empty IP → error — a template for "invalid input format" assertions.
- Coverage tracked via codecov (`README.md:15`).

**Test Types**:
| Test Type | Scope | Tools |
|-----------|-------|-------|
| Unit | Reason classification mapping; new device-path validation logic; retry/backoff bounds | Go stdlib testing + testify |
| Integration | Each empty-result path (`GetHousehold`, `GetHouseholdFromIP`, batch) returns the correct distinct reason; backward-compat of the changed single-RPC empty shape; no-raw-PII in reasons/logs | testcontainers (`redis:7`) + testify |
| E2E / Contract | Regenerated Go + Python stubs deserialize; existing bare-nil-expecting consumers still function | proto codegen (`make protoc-compile`) + consumer verification (see NFR-009) |

**Implementation Notes**:
- Add an explicit assertion that no raw device_id/ip appears in reason content or logs (FR-007/FR-008/NFR-004/NFR-005).
- The backward-compat test for the single-RPC empty path (populated-but-empty message vs. former typed-nil) is the subtlest risk and must be covered explicitly (see Corrected Assumptions).

---

## Specification Validation

### Confirmed Assumptions

| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| Proto/contract surface is a confirmed affected area (BR-1/BR-2) | proto CA-2 (`household_service.proto:30-36`), CA-4 (`household_service.pb.go:522`) | Confirmed; field 4 on `KeyedHousehold` is the free additive slot. |
| Additive, backward-compatible contract change is feasible (BR-5/FR-010/NFR-007) | proto CA-2/CA-4; proto3 unknown-field semantics | Structurally satisfiable — old clients ignore unknown fields. |
| Three distinct causes exist (no-match / dependency-unavailable / invalid-input) | backend CA-1 (lines 38 vs 42), CA-2 (lines 68 vs 72) | no-match & dependency-unavailable already distinguished in control flow; invalid-input only partially (IP path). |
| Privacy boundary pattern (BR-4/FR-007/FR-008) | backend CA-6 (`household.go:35,62` hash-before-log) | Existing hash-before-log pattern to follow; commented line 40 is the pre-rejected violation. |
| Feature is self-contained (not multi-service for the diagnostic content itself) | backend "multi-service flow" row; only outbound call is Dragonfly | No cross-service seam to trace for diagnostics; blast radius is this service + proto + generated client. |
| No background/async dispatch; synchronous request/response | backend repo-at-a-glance (`cmd/`, `internal/commands/` only `server`/`reader`) | Deploy-topology / background-dispatch decisions do not apply. |

### Corrected Assumptions

| Spec Statement | Actual Reality | Impact on Implementation |
|----------------|----------------|-------------------------|
| API returns an "empty-result indication" that can be extended (BR-5) | Single-lookup empty = bare **`(nil, nil)`** typed-nil; **no field carries a reason** today (backend CA-1, proto CA-2/CA-4) | Not "add a field to an existing error object" — must change the single-RPC empty path from bare-nil to a populated-but-empty message with a reason (or add a new RPC). Genuine contract-design decision; subtlest backward-compat risk (a current consumer checking for nil could break). |
| Three distinct causes are all detectable today | **"Invalid input format" is NOT detected on the device path** — `HashDeviceIDAndType` returns a plain string, no error (backend CA-5) | FR-003 requires net-new validation logic on the device path; "malformed" is undefined and must be specified (backend Q6). |
| Bounded retry behavior can be configured on existing calls (FR-009) | Dragonfly `Get`/`SMisMember` calls are **single-shot** today (backend CA-1/CA-2) | Bounded-retry wrapper (3 attempts, exp backoff, Retry-After) is net-new middleware logic. |
| Low-confidence front-end display surface (`frontend-mos`) for agents (NFR-012) | **No household-ID-adornment lookup surface exists in `frontend-mos`** — 0 matches for `adornment`, `household-id`, and all three BR-2 reason strings (frontend investigation, Read-verified) | `frontend-mos` is corrected to **not present in this repo**; it is the customer-facing marketing console, not a support-agent ops tool. Front-end display work stays out of scope for Phase 1 (DEC-003/NFR-012). **Where agents actually read reasons remains unconfirmed** (OQ-005). |

### Role-note coverage

- **Backend** (`eng-context-backend.md`): all key concerns reflected above — bare-nil→populated-empty contract risk, net-new device-path validation, batch per-item structural site, contract-shape decision, bounded-retry wrapper, privacy pattern, BSD `sed` gotcha, unenumerated proto consumers, and the five open backend questions (Q1–Q6). The `householdutils` internals and the `ko-k8s-apps/adornment` deploy config are both **external/uninvestigated** and surface as Open Questions in the register (p95 baseline source, "invalid input" definition).
- **Frontend**: the backend engineer and a dedicated live investigation both ruled out `frontend-mos`. No frontend role-notes file was provided and no in-scope frontend surface exists for Phase 1. **Where agents actually consume the diagnostics is unidentified by KG/investigation and is carried as an Open Question (OQ-005)** — if a real support/display surface exists, its hosting repo is not yet named and must be confirmed before Affected Systems can include it.
- **Infra**: no infra role-notes file was provided. Two infra-adjacent items the backend engineer flagged are carried as Open Questions: (1) the p95 baseline dashboard source for NFR-001/DEC-005, and (2) wiring the NFR-003 dependency-health alert rule. The production k8s deploy config lives in the **external `ko-k8s-apps/adornment` repo** (surfaced by KG at medium confidence), **not in the investigated checkout** — its hosting is noted but uninvestigated.

---

## Summary

The two primary repositories were investigated live and the diagnostic enhancement is well-scoped to them: a new proto enum + reason field on `KeyedHousehold` (field 4) covering both RPCs, control-flow mapping for two of three reasons, net-new device-path validation for the third, a bounded-retry wrapper around Dragonfly, and privacy-safe reason/telemetry content following the existing hash-before-log pattern. The spec's low-confidence `frontend-mos` link was investigated and corrected to "not present," keeping an unrelated console off the change surface.

Several decisions remain product/architect-owned and are carried forward: the on-wire message shape (enum-only vs enum+string vs status code), whether batch needs per-input reasons, the definition of "invalid input format" on the device path, the p95 baseline source, and — most consequentially — where support agents actually consume the diagnostics (the consumption surface, still unidentified) and whether any current consumer relies on the single-RPC bare-nil empty result. These gate the final contract shape (NFR-008) and rollout (NFR-009).

**Decisions Made**: 9 (stack reuse; proto enum + field-4 placement; reason↔control-flow mapping; batch default + gate; success-path isolation + p95 baseline policy; bounded retry; enum-classified privacy-safe error handling; testing via existing testcontainers suite)
**Open Questions**: 7 (consumption surface; single-RPC bare-nil dependence; batch per-item requirement; on-wire message shape; "invalid input format" definition; p95 baseline source; full proto consumer enumeration)
**Status**: Ready for Implementation Planning on the non-gated backend/proto mechanics; the final contract shape and rollout are Blocked on the product/architect decisions above.

---
## Open Questions & Decisions Register

_Durable, ID'd reference for the next HITL gate — inline `[DECISION REQUIRED: DEC-00N]` tags in the sections above point back here. Reflects the latest round only._

### Decisions Required (business/technical trade-offs — needs a human call)

_Each carries an **Owner** and a **Provisional Value** engineering may build against until the decision lands._

| ID | Decision | Owner | Provisional Value (build against until confirmed) |
|----|----------|-------|-----------------------------------------------------|
| **DEC-008** _(Code-only)_ | On-wire shape for the diagnostic reason: enum-only vs enum + static message string vs gRPC status codes. Enum-only is privacy-safe by construction (cannot echo customer data); a free-text string field could leak. gRPC status codes were tried and reverted in-code (household.go:40). Must be fixed before the proto is finalized (NFR-007). | Backend lead + PM (eng-plan contract decision) | New proto enum (NoMatchFound / InvalidInputFormat / LookupDependencyUnavailable) at KeyedHousehold field 4, enum-only, with agent-facing wording rendered client-side rather than carried as free text — the privacy-safest shape that still satisfies BR-2/BR-3. |
| **DEC-009** _(Code-only)_ | Does batch mode (GetHouseholds) require per-input diagnostic reasons, or is envelope-level acceptable? Per-input requires restructuring the aggregation loop (household.go:84-116) which currently drops errors and nils. | PM | Per-input: place the reason on each KeyedHousehold (field 4), which also serves the single-lookup RPC and avoids a second placement. Default to building per-input unless product confirms envelope-level suffices. |
| **DEC-010** _(Code-only)_ | Definition of 'invalid input format' on the device-id path. HashDeviceIDAndType returns a plain string with no error (household.go:33), so no validation exists today; without a definition, 'no match' and 'invalid input' are indistinguishable on that path (FR-003). | PM + Backend | Reject empty device_id and empty/unrecognized device_id_type as invalid input format before hashing; treat everything else as potentially matchable (no match) until householdutils validation semantics are confirmed. |
| **DEC-011** _(Code-only)_ | Actual current p95 baseline for the success path and its monitoring source, to set the NFR-001/DEC-005 non-regression gate (not supplied in the input). | Infra + Backend | Capture the current success-path p95 from the existing kotel/Prometheus dashboard at the start of the work and freeze that observed number as the regression gate; diagnostics touch only the no-results path so no regression is expected. |

### Open Questions (evidence gaps — needs more input)

- **OQ-008** _(Code-only)_: Where do support agents actually consume the diagnostic reasons (raw gRPC response, a support tool, or logs)? frontend-mos is ruled out and no hosting repo for an agent display surface has been identified by KG or investigation — needs confirmation before any display surface can be added to Affected Systems (OQ-005, NFR-012).
- **OQ-009** _(Code-only)_: Does any current consumer of GetHousehold rely on receiving a nil/typed-nil KeyedHousehold on no-match? This determines whether the single-RPC empty path can switch from (nil,nil) to a populated-but-empty message or whether a new RPC is required; tied to the full proto-consumer enumeration (NFR-009). The complete consumer list is not enumerable from the investigated checkout.
- **OQ-010** _(Code-only)_: Which repository hosts the production k8s deploy config and the NFR-003 dependency-health alert rule? The backend investigation locates these in the external ko-k8s-apps/adornment repo (KG medium confidence), which was not investigated — needs confirmation before the alerting rule and any deploy changes can be planned.
