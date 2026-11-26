# Implementation Plan: K4A — Enhanced Error Reporting for Household ID Adornment Lookups

**PRD**: [product-spec.md](./product-spec.md)
**Date**: 2026-11-26
**Research**: [eng-research.md](./eng-research.md)
**Status**: Draft

## Summary

Enhance the `household-id-adornment-api` no-results path so every empty lookup carries exactly one distinct, named, privacy-safe diagnostic reason — 'no match found', 'invalid input format', or 'lookup dependency unavailable'. The reason is introduced from scratch as the proto contract's first `enum` plus a new field (tag 4) on `KeyedHousehold` in `household-id-adornment-proto`, consumed by the Go service which maps two of the three reasons onto existing control-flow branches, adds net-new device-path input validation for the third, and wraps Dragonfly calls in bounded retry (verified live across both repos).

---

## Technical Context

**Language/Version**: Go 1.22 (`household-id-adornment-api`); proto3 + dual Go/Python codegen (`household-id-adornment-proto`) — investigation-verified.
**Primary Dependencies**: gRPC, Dragonfly (Redis-protocol KV, sole outbound dependency), kotel (OTel), kolog, Prometheus, redisotel — all investigation-verified; no new dependency introduced.
**Storage**: No persistent-store change. Dragonfly KV schema untouched; the diagnostic reason is computed at request time, never stored (investigation-verified, api repo CA-P1 / avro-dragonfly-loader CA-1–CA-3).
**Testing**: Existing `household_test.go` testcontainers (`redis:7`) + `stretchr/testify` suite, extended with table-driven reason assertions (investigation-verified, api CA-7).
**Target Platform**: Kubernetes via external `ko-k8s-apps/adornment` — KEDA-autoscaled 8–32 replicas in prod, method-agnostic Envoy prefix routing (investigation-verified, ko-k8s-apps CA-2/CA-5).
**Performance Goals**: Hold current success-path p95 as the non-regression baseline (NFR-001); the baseline number and its monitoring source are not supplied and must be confirmed (OQ, below).
**Constraints**: Additive-only contract change (NFR-007); no raw PII in responses or logs (NFR-004/005); never collapse causes into one generic reason (FR-005/NFR-010).

---

## Standards Check

### Go Style Guide
- [x] Follows `/kochava:go-style-guide` patterns — reuses existing handler structure in `household.go`
- [x] Error handling consistent with standards — maps existing `redis.Nil`/`err != nil` branches to reasons; follows established hash-before-log pattern
- [x] Package structure follows conventions — no new packages; a bounded-retry wrapper added within existing internal layout

### Testing Standards
- [x] Test strategy covers unit/integration needs — unit (classification, validation, retry bounds) + integration (testcontainers per empty-result path)
- [x] Uses approved frameworks (testify, testcontainers) — already in use
- [x] Coverage targets defined — codecov already wired; add reason + no-PII assertions

### Dependency Standards
- [x] All dependencies on approved list — no new dependency
- [x] No new unapproved dependencies OR justification provided — none added
- [x] Dependency versions pinned — existing pins unchanged

### Documentation
- [x] CLAUDE.md updates planned (if needed) — note the new reason enum + the proto-repo BSD `sed` regen hazard
- [x] API documentation planned — proto comments document the new enum/field; released as `feat:` minor bump

**Standards Status**: PASS WITH NOTES — proto-repo regen on Linux must work around the macOS-only `sed -i ''` at `Makefile:49` (investigation-verified, proto CA-7); this is an operational hazard, not a code-logic issue.

---

## Specification Validation

*Summary of findings from eng-research.md (investigation-verified unless noted).*

### Confirmed Assumptions

| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| Proto contract is the affected surface (BR-1/BR-2) | Investigation: proto CA-2 (`household_service.proto:30-36`), CA-4 | Field 4 on `KeyedHousehold` is the free additive slot. |
| Additive, backward-compatible contract change is feasible (BR-5/FR-010/NFR-007) | Investigation: proto CA-2/CA-4; proto3 unknown-field semantics | Old clients ignore unknown fields; `_UNSPECIFIED=0` doubles as the backward-compat signal. |
| Two of three causes already distinguished in control flow | Investigation: api CA-P3 (`household.go:38` vs `42`), CA-2 IP path | no-match & dependency-unavailable already branch; invalid-input only partial (IP path). |
| Privacy pattern already established (BR-4/FR-007/FR-008) | Investigation: api CA-6 (`household.go:35,62` hash-before-log) | Follow it; do NOT resurrect the privacy-violating commented `status.Errorf` at `household.go:40`. |
| Feature self-contained; only outbound dependency is Dragonfly | Investigation: api repo + ko-k8s-apps CA-4 (`dragonfly.pubsuite.internal:6379`) | No cross-service seam for the diagnostic content. |
| No background/async tier; per-request retry is safe | Investigation: ko-k8s-apps CA-2 (KEDA 8–32, no single-replica/cron tier) | Bounded in-process retry needs no leader election. |

### Corrected Assumptions

| Spec Statement | Actual Reality | Impact |
|----------------|----------------|--------|
| API has an "empty-result indication" to extend (BR-5) | Single-lookup empty = bare `(nil, nil)` typed-nil; **no field carries a reason today** (investigation: api CA-P3, proto CA-5, `NumEnums: 0`) | Not "add a field to an error object" — must change the single-RPC empty path from bare-nil to a populated-but-empty message (or add a new RPC). Subtlest backward-compat risk. |
| Three causes are all detectable today | **"Invalid input format" is NOT detected on the device path** — `HashDeviceIDAndType` returns a plain string, no error (investigation: api CA-5) | FR-003 requires net-new device-path validation; "malformed" is undefined. |
| Bounded retry configurable on existing calls (FR-009) | Dragonfly `Get`/`SMisMember` are single-shot today (investigation: api CA-1/CA-2) | Bounded-retry wrapper is net-new middleware logic. |
| `frontend-mos` is a plausible agent surface (NFR-012) | **No household-adornment lookup surface exists in `frontend-mos`** — 0 matches for `adornment`/`household-id`/all three reason strings (investigation, Read-verified, two passes) | `frontend-mos` corrected to not-present; off the affected list. Where agents actually read reasons remains unconfirmed (OQ). |
| `GetHouseholdFromIP` is a single-lookup RPC to enhance | **It is NOT a registered RPC** — absent from the `.proto` and gRPC registration; reachable only indirectly via the batch IP path (investigation: api CA-P2) | No dedicated single-IP RPC exists; single-IP diagnostics can only surface via the batch envelope. |
| "dual Go/Python codegen + BSD sed gotcha" applies in the api repo | The **api repo** regen is Go-only with no `sed` step; the Python stubs and the `sed -i ''` hazard live in the **separate proto repo** (investigation: api "regen scope correction", proto CA-7) | Apply the sed workaround only for proto-repo regen. |

---

## Project Structure

### Documentation (PRD Folder)

```text
docs/16-PRDs/K4A-enhanced-error-reporting/
├── product-spec.md
├── eng-context-backend.md
├── eng-research.md
├── eng-plan.md          # This document
├── data-model.md        # Generated by this skill
└── contracts/           # Generated by this skill (proto IDL excerpt)
```

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
├── qa/api.yaml                                             image tag bump (+ ConfigMap only if retry is runtime-configurable)
└── prod/api.yaml                                           image tag bump
```

**Structure Notes**: The reason concept is defined once in the proto repo and consumed in the api repo. No persistent-store migration exists. ko-k8s-apps changes are deploy-only (image-tag bumps); routing is message-shape-agnostic so Envoy/Istio/Service need no change (investigation: ko-k8s-apps CA-5).

---

## Affected Systems

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

### ko-k8s-apps (deploy/rollout only)

**Technology**: FluxCD/Kustomize GitOps (investigation-verified).
**Key Files**:
- `adornment/household-id/api/qa/api.yaml`, `prod/api.yaml` — image/version bumps (CA-6)

**Changes**:

| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Bump image tag + version label in qa then prod | Modify | L | Routing/HPA/PDB need no change (CA-5/CA-2) |
| Add retry-tuning env var to ConfigMap | Modify | L | ONLY if retry is made runtime-configurable; none exists today (CA-3) |

### Related systems (context, NOT affected — no work lands)

Confirmed via live investigation/rule-out: **avro-dragonfly-loader** (write/loader side; its 8-day TTL + bad-IP-skip semantics inform "no match found" wording so it stays actionable per BR-3), **kolog** and **kotel** (unchanged telemetry/logging dependencies; existing helpers suffice; BR-4 redaction is the api caller's responsibility), **constants** and **control** (KG name-collisions — `constants` is a generic config API, `control`'s "adornment" is uadd device-dimensions; both ruled out with whole-repo `grep -i household` → zero matches), and **frontend-mos** (ruled out, two passes).

---

## Implementation Phases

### Phase 0: Test Coverage Foundation

**Goal**: Pin current behavior before changing the empty-result contract
**Prerequisites**: None

- [ ] Verify existing `household_test.go` suite passes against `redis:7` testcontainers
- [ ] Add a characterization test asserting the CURRENT single-RPC empty behavior (so the bare-nil → populated-empty change is deliberate and visible)
- [ ] Confirm existing hash-before-log behavior is exercised

**Deliverables**: Green baseline + explicit test on the empty-result contract being changed

---

### Phase 1: Contract (proto repo)

**Goal**: Introduce the reason vocabulary additively
**Prerequisites**: Phase 0; on-wire message-shape decision (see Register) — provisional default lets Phase 1 proceed

- [ ] Add `enum NoResultReason` with `NO_RESULT_REASON_UNSPECIFIED = 0` + three named values
- [ ] Add `reason` field at tag 4 on `KeyedHousehold`
- [ ] Add envelope reason at `GetHouseholdsResponse` tag 2 (if batch granularity decision requires it)
- [ ] Regenerate Go + Python stubs; **apply Linux `sed -i` workaround for `Makefile:49`**; commit all generated trees
- [ ] Release `feat:` minor bump (v1.0.3 → v1.1.0)

**Deliverables**: Backward-compatible contract carrying the reason vocabulary

---

### Phase 2: Core Implementation (api repo)

**Goal**: Populate the correct distinct reason on every empty-result path
**Prerequisites**: Phase 1 (consume new stubs)

- [ ] Change `GetHousehold` empty path to populated-empty `KeyedHousehold` + reason (backward-compat verified per Register item)
- [ ] Map `redis.Nil`→NO_MATCH_FOUND; post-retry failure→LOOKUP_DEPENDENCY_UNAVAILABLE
- [ ] Add net-new device-path validation → INVALID_INPUT_FORMAT (per the "malformed" definition, see Register)
- [ ] Add the bounded-retry wrapper (3 attempts, exp backoff, honor `Retry-After`) around both Dragonfly call sites
- [ ] Ensure FR-005 — exactly one named reason, never a generic fallback
- [ ] Privacy pass: no raw id/ip in reason or logs; follow hash-before-log

**Deliverables**: All three distinct reasons returned on the single-lookup path

---

### Phase 3: Batch Path & Integration

**Goal**: Consistent diagnostics in batch mode + end-to-end verification
**Prerequisites**: Phase 2; batch-granularity decision (see Register)

- [ ] Restructure `GetHouseholds` loop to carry per-item reasons (if per-input required) or envelope reason (if envelope-level accepted)
- [ ] Integration tests: each empty-result path returns the correct distinct reason
- [ ] Backward-compat test: existing bare-nil-expecting consumers still function against the changed empty shape
- [ ] Explicit no-raw-PII assertion in reasons and logs

**Deliverables**: Feature working end-to-end across single and batch RPCs

---

### Phase 4: Polish, Observability & Rollout

**Goal**: Finalize for release
**Prerequisites**: Phase 3; proto-consumer enumeration (see Register); p95 baseline source (see Register)

- [ ] Add reason-labeled metric (counter) via existing OTel global meter; no kotel change
- [ ] Wire NFR-003 alert rule (dependency error rate >1% for 5 min sustained) on existing Prometheus metrics
- [ ] Privacy review of diagnostic content as a release gate (NFR-011)
- [ ] Confirm success-path p95 unchanged vs baseline
- [ ] Bump image tags in ko-k8s-apps qa → prod
- [ ] Update CLAUDE.md (new enum + proto-repo sed hazard)

**Deliverables**: Feature released, alerted, and privacy-reviewed

---

## Data Model & Contracts

- **Data Model**: [data-model.md](./data-model.md)
- **API Contracts**: [contracts/](./contracts/)

---

## Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Changing single-RPC empty from bare-nil to populated-empty breaks a consumer doing `if resp == nil` (investigation: api CA-P3) | M | H | Enumerate proto consumers (NFR-009); explicit backward-compat test; fall back to a new RPC only if a bare-nil dependence is found |
| Full proto consumer list not enumerated (NFR-009) | H | M | Enumerate and confirm each unaffected before rollout — blocking gate for Phase 4 |
| "Invalid input format" undefined on device path (investigation: api CA-5) | H | M | Define "malformed" with product before writing validation; `householdutils` internals unverified (external, not vendored) |
| Proto-repo Linux regen fails on `sed -i ''` (investigation: proto CA-7) | M | L | Use `sed -i` (no empty-string arg) on Linux CI/local |
| Batch loop restructure is hairier than single-RPC (investigation: api CA-P4) | M | M | Scope depends on batch-granularity decision; default per-`KeyedHousehold` placement supports per-item |
| Privacy leak via free-text reason message (BR-4) | M | H | Prefer enum-only (cannot leak by construction); privacy review as release gate (NFR-011) |

---

## Dependencies & Blockers

| Dependency | Owner | Status | Needed By |
|------------|-------|--------|-----------|
| Full `KeyedHousehold`/proto consumer enumeration (NFR-009) | Backend / proto-consumer teams | Open | Phase 4 rollout |
| "Invalid input format" definition on device path | PM / Backend | Open | Phase 2 |
| Success-path p95 baseline + monitoring source | Infra / Backend | Open | Phase 4 regression gate |
| NFR-003 alert-rule wiring (observability repo) | Infra | Open | Phase 4 |

---

## Appendix

### Related Documents

- [Product Spec](./product-spec.md)
- [Engineering Research](./eng-research.md)
- [Backend Context](./eng-context-backend.md)

### Decision Log

| Date | Decision | Rationale | Decided By |
|------|----------|-----------|------------|
| 2026-11-26 | Carry reason as proto enum + field 4 on `KeyedHousehold` | Only placement serving both RPCs; additive/backward-compatible | eng-research |
| 2026-11-26 | Reuse control-flow for 2 reasons; net-new validation for invalid-input | Two causes already branch; device path has no validation today | eng-research |
| 2026-11-26 | Enum-only preferred over free-text for privacy | Enum cannot leak customer data by construction | eng-research |

### References

- Investigation findings (Read-verified): `household-id-adornment-api`, `household-id-adornment-proto`, `frontend-mos`, `avro-dragonfly-loader`, `ko-k8s-apps`, `kolog`, `kotel`, `constants`, `control`

---
## Open Questions & Decisions Register

_Durable, ID'd reference for the next HITL gate — inline `[DECISION REQUIRED: DEC-00N]` tags in the sections above point back here. Reflects the latest round only._

### Decisions Required (business/technical trade-offs — needs a human call)

_Each carries an **Owner** and a **Provisional Value** engineering may build against until the decision lands._

| ID | Decision | Owner | Provisional Value (build against until confirmed) |
|----|----------|-------|-----------------------------------------------------|
| **DEC-008** _(Code-only)_ | On-wire shape of the diagnostic reason: enum-only vs enum + static message string vs gRPC status codes. Enum-only cannot leak customer data by construction (safest for BR-4/FR-007); gRPC status codes were already tried and reverted in-code (household.go:40, with a privacy-violating device-id echo). This forced NFR-007 contract decision gates the final proto field set. | Backend lead / Architect (with PM for actionability wording) | Enum-only (NoResultReason field 4), no free-text message field — proceed with this for Phase 1; add a companion static message string only if a confirmed human-facing surface requires presentable wording. |
| **DEC-009** _(Code-only)_ | Single-lookup empty-path backward compatibility: GetHousehold returns bare (nil, nil) today with no message to carry a reason. Switching to a populated-but-empty KeyedHousehold changes what 'empty' looks like on the wire; a consumer checking `if resp == nil` behaves differently than one checking `if resp.Household == nil`. Must confirm no current consumer depends on bare-nil before adopting populated-empty vs. adding a new RPC. | Backend lead (requires NFR-009 proto-consumer enumeration) | Return populated-but-empty KeyedHousehold with reason field set, pending consumer enumeration; reserve a new RPC as the fallback only if a bare-nil dependence is found. |
| **DEC-010** _(Code-only)_ | Batch (GetHouseholds) diagnostic granularity: per-input reasons (restructure the aggregation loop that currently swallows errors/nils) vs envelope-level reason only. Per-item is more work but preserves which input failed; envelope-only cannot express per-input causes but suffices for request-wide failures like dependency-unavailable. | PM (product requirement) / Backend lead | Support per-item reasons via KeyedHousehold.reason (the chosen field-4 placement already enables this) AND add the envelope-level field for request-wide dependency-unavailable; default to this richer shape until product confirms envelope-only is acceptable. |
| **DEC-011** _(Code-only)_ | Definition of 'invalid input format' on the device path: HashDeviceIDAndType accepts any string with no error today, so FR-003 requires net-new validation. What counts as malformed (empty device_id? invalid device_id_type? format rules?) is undefined and householdutils internals are unverified (external module, not vendored). | PM / Backend lead | Treat empty device_id OR empty/unrecognized device_id_type as INVALID_INPUT_FORMAT for Phase 1; refine once householdutils validation semantics are confirmed. |
| **DEC-012** _(Code-only)_ | Success-path p95 baseline and its monitoring source: NFR-001/DEC-005 require 'no measurable regression vs current p95' but no baseline number or source is supplied. Needed to set the Phase 4 regression gate. | Infra / Backend lead | Capture the current prod p95 from the existing kotel/Prometheus (Datadog) dashboard as the baseline immediately before Phase 2; gate on no measurable regression against that captured number. |

### Open Questions (evidence gaps — needs more input)

- **OQ-008** _(Code-only)_: Where do support agents actually consume the diagnostic reasons (raw gRPC response, a support tool, or logs)? frontend-mos is ruled out (two Read-verified passes, zero matches) and no hosting repo for any agent display/support surface has been identified by KG or investigation. This must be confirmed before any display/parsing work can enter Affected Systems; until then it stays out of scope (DEC-003).
- **OQ-009** _(Code-only)_: What is the complete set of consumers that deserialize KeyedHousehold / depend on GetHousehold's response shape (NFR-009)? The full proto-consumer list is not enumerated in any checkout and is a required rollout gate — especially to confirm no consumer relies on the single-RPC bare-nil empty result before the populated-empty change ships.
- **OQ-010** _(Code-only)_: Should the NFR-003 dependency-health alert rule (Dragonfly error rate >1% for 5 min sustained) live in ko-k8s-apps/observability, and does a household-id Grafana dashboard already exist there? Investigation did not exhaustively search observability/, so the alert may be net-new; its hosting must be confirmed during Phase 4.
