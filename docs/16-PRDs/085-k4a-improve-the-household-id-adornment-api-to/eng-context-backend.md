# Engineering Notes: K4A Enhanced Error Reporting for Household ID Adornment Lookups - Backend

**Engineer:** _(unassigned)_
**Date:** 2026-11-26
**Based on:** product-spec.md

---

## Initial Thoughts

Did a real pass through `household-id-adornment-api` and `household-id-adornment-proto`. Good news: this is **self-contained** to these two repos + the generated client lib. No cross-service calls, no HTTP, no async workers — it's a synchronous gRPC service whose only outbound dependency is Dragonfly (Redis-protocol KV). Two unary RPCs, that's the whole surface: `GetHousehold` and `GetHouseholds`.

The spec frames this as "enhance the empty-result response with a reason." In code the empty result is literally `return nil, nil` — a bare typed-nil. There's **no field anywhere today that could carry a reason** (`NumEnums: 0`, no diagnostic/status fields in the proto). So it's not "add a field to an existing error object"; it's "introduce a reason concept from scratch, and change what empty looks like on the wire for the single-lookup RPC."

Also found a smoking gun: `household.go:40` has a **commented-out** `status.Errorf(codes.NotFound, ...)` — someone already contemplated a richer no-result signal and backed it out. AND that commented line echoes the device id into the message, which is exactly the privacy pattern BR-4/FR-007 forbids. So the tempting approach is pre-rejected for good reason.

Gut feel on complexity: **medium**. Two of three reasons are basically plumbing (control flow already distinguishes them). The third ("invalid input format") and the contract-shape decision are the real work.

---

## Concerns & Risks

- **Bare-nil → populated-empty on the single RPC.** `GetHousehold` returns `KeyedHousehold` directly. To carry a reason on empty, we have to STOP returning `(nil, nil)` and instead return a populated-but-empty message with a reason field. That changes what "empty" looks like on the wire for existing single-lookup callers. BR-5/FR-010 say additive-only and existing callers unchanged — need to make sure returning a non-nil message with an empty `household` field doesn't trip up any current consumer that checks for nil. This is the subtlest backward-compat risk in the whole thing.
- **"Invalid input format" doesn't exist yet on the device path.** `HashDeviceIDAndType` returns a plain string, no error — any input hashes to *something* and just misses. So FR-003 requires **net-new validation logic** that isn't there today. By contrast `HashIP` already returns an error on bad IPs, so the IP path is partially covered. Couldn't read `householdutils` internals (external module `avro-dragonfly-loader`, not vendored) so exact validation semantics are unverified.
- **Batch path (`GetHouseholds`) swallows per-item causes.** The aggregation loop logs errors + `continue`s and drops nils; a partial/empty batch returns `(nil, nil)` with no per-input reason. If FR-002–005 imply per-input diagnostics in batch mode, there's **nowhere to attach a reason per device/IP today** — that's a second structural change site and arguably the hairier one.
- **Two empty-result paths, not one.** `GetHousehold` (device) AND `GetHouseholdFromIP` share the `(nil, nil)` convention. Both need covering for consistency.
- **Contract-shape decision is forced but unresolved (NFR-007).** Enum vs string vs status-code; per-`KeyedHousehold` field vs batch-envelope field; message-in-proto vs client-rendered. Proto repo pushes this on us — the spec leaves it open.
- **Bounded retry doesn't exist today.** FR-009 wants up to 3 attempts w/ exponential backoff + honor `Retry-After`. The current Dragonfly `Get`/`SMisMember` calls are single-shot. That's new middleware/wrapper logic.
- **Proto repo regen gotcha:** the Python codegen Makefile step uses BSD `sed -i ''` (macOS-specific). Will bite on Linux CI. Noting now so it's not a surprise.
- **Full proto consumer list is NOT enumerated** (NFR-009). This repo only knows about its Go/Python client stubs; who else deserializes `KeyedHousehold` downstream is unknown from the checkout.

---

## Questions & Answers

### Q1: Does any current consumer of `GetHousehold` rely on receiving a nil/typed-nil `KeyedHousehold` on no-match?
- **For:** Backend (proto consumers) / PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Determines whether we can safely switch the single-lookup empty path from `(nil, nil)` to a populated-but-empty message (required to carry a reason), or whether we need a new RPC to avoid breaking existing callers. Blocks the contract-shape decision and the single-RPC implementation approach. Tied to NFR-009's "enumerate all proto consumers" gate.

### Q2: Does the no-results diagnostic need to be **per-input in batch mode** (`GetHouseholds`), or only on the single-lookup RPC?
- **For:** PM
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** If per-input, the batch aggregation loop (which currently drops errors and nils) needs restructuring so each `KeyedHousehold` carries its own reason — meaningfully more work than the single RPC. If envelope-level only is acceptable, much smaller.

### Q3: What shape should the reason take on the wire — enum only, enum + static message string, or gRPC status codes?
- **For:** PM / Backend (eng-plan decision)
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** Drives the proto change and the privacy posture. Enum-only inherently can't leak customer data (safer for BR-4/FR-007); a free-text string field could. Note gRPC status codes were already tried-and-reverted in code. This is DEC-002/NFR-007-adjacent and needs a call before finalizing the proto.

### Q4: Where do support agents actually read these reasons? (raw gRPC response, support tool, logs?)
- **For:** PM / Product
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** I ruled out `frontend-mos` as the surface (see Notes — zero matches, confirmed). If agents read raw gRPC, enum-only is fine. If there's a human-facing display, actionable wording / "customer-friendly vs agent-internal" matters. Phase 1 assumes no front-end work (DEC-003), so this mainly gates whether reason *wording* needs to be human-presentable. Still genuinely unresolved per OQ-005.

### Q5: What's the actual current p95 baseline for the happy path, and from which monitoring source?
- **For:** PM / Infra / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** NFR-001/DEC-005 say "no measurable regression vs current p95" but the baseline number isn't supplied. Need the real number from existing kotel/Prometheus dashboards to set a regression gate. Low risk (diagnostics only touch the no-results path, success path shouldn't change) but need the number to prove it.

### Q6: Confirm "invalid input format" is expected for the **device-id** path given there's no validation there today — what counts as invalid?
- **For:** PM / Backend
- **Status:** 🔴 Open
- **Answer:** _Pending response_
- **Impact:** FR-003 requires detecting malformed input, but `HashDeviceIDAndType` currently accepts anything. Need to know what "malformed" means (empty device_id? bad device_id_type? format rules?) before I can write validation. Without a definition, "no match" and "invalid input" are indistinguishable on the device path.

---

## Rough Scope

What I think we'll need to do:

**Proto repo (`household-id-adornment-proto`):**
- Add a new `enum` for the reason vocabulary (no match found / invalid input format / lookup dependency unavailable) — `NumEnums` goes 0→1
- Add reason field (field #4 is free) to `KeyedHousehold` — placement that covers BOTH RPCs
- Maybe companion `string message` field for actionability (pending Q3) — weigh against BR-4 privacy
- Regenerate Go **and** Python stubs via `make protoc-compile`, commit both (watch the BSD `sed` gotcha)
- Land with conventional-commit `feat:` → semantic-release minor bump from v1.0.3

**API repo (`household-id-adornment-api`):**
- `GetHousehold` (`household.go:32-53`): stop returning `(nil, nil)` on `redis.Nil`; return populated-empty message w/ reason. Map: `redis.Nil` → no match; `err != nil` → dependency unavailable; new pre-hash check → invalid input
- `GetHouseholdFromIP` (`household.go:56-82`): same treatment — dependency vs no-match already distinguished in control flow
- **New input validation** for the device path (doesn't exist today) to support "invalid input format"
- `GetHouseholds` batch loop (`household.go:84-116`): restructure to attach per-item reasons IF Q2 says per-input
- **New bounded-retry wrapper** around Dragonfly calls (3 attempts, exp backoff, honor `Retry-After`) for FR-009/NFR-002
- Privacy: ensure no raw id/ip in reason content or logs — follow the existing hash-before-log pattern (`household.go:35,62`); do NOT resurrect the commented echo at line 40
- Dependency health alerting (NFR-003): error rate >1% for 5 min — kotel/Prometheus already wired, so likely an alerting rule not new instrumentation
- Extend testcontainer tests (`household_test.go`) with reason assertions — table-driven, framework already there

---

## Dependencies on Other Roles

- **PM / Product** — need the content & consumption-surface decisions (Q3, Q4), the "invalid input format" definition (Q6), and the p95 baseline source (Q5). Spec itself flags these as the gating risks (NFR-008).
- **Proto consumers (other backend teams)** — need the full consumer list enumerated and each confirmed unaffected by the additive change (NFR-009) before rollout. I can't produce this list from the checkout.
- **Frontend** — **none for Phase 1.** I investigated `frontend-mos` directly and ruled it out as the agent surface (see Notes). No front-end work in scope until the consumption-surface decision reopens it (DEC-003, NFR-012).
- **Infra** — for the p95 baseline dashboard (Q5) and wiring the NFR-003 dependency-health alert rule; k8s deploy config lives in external `ko-k8s-apps/adornment` repo, not this checkout.

---

## Notes

- **frontend-mos ruled OUT (high confidence despite the KG low-confidence hint).** Investigated the repo directly: `adornment` = 0 matches, `household-id` / `adornment-api` = 0 matches, all three BR-2 reason strings = 0 matches. The "household" hits that do exist are unrelated (Atlas household graph CRM, attribution-waterfall glossary copy). It's the customer-facing marketing console — no support-agent/internal-ops lookup tool among its views/routes. So the spec's low-confidence front-end link is corrected to **not present in this repo**. Where agents DO read results is still unknown (Q4).
- **Key code anchors for whoever picks this up:**
  - `household.go:32-53` — `GetHousehold`, the primary empty-result site; `redis.Nil` branch = no-match, `err != nil` = dependency-down
  - `household.go:40` — commented-out `status.Errorf` with privacy-violating device-id echo; the pre-rejected approach, don't resurrect
  - `household.go:56-82` — `GetHouseholdFromIP`, second empty path; `HashIP` already errors on bad input
  - `household.go:84-116` — `GetHouseholds` batch loop that swallows per-item causes
  - `household_service.proto:30-36` — `KeyedHousehold`, field 4 is the free slot
- Observability already solid: kotel (OTel) + kolog + Prometheus + redisotel + slog JSON. NFR-003 alerting likely sits on top of existing metrics.
- Deploy is k8s via external `ko-k8s-apps/adornment`; local dev is docker-compose only. No background/async dispatch anywhere — purely synchronous request/response.
- Couldn't verify `householdutils.HashDeviceIDAndType`/`HashIP` internals (external module, not vendored). Validation semantics beyond observed signatures are unconfirmed — ties into Q6.

---

*Captured by `/kochava:eng-context backend` on 2026-11-26*