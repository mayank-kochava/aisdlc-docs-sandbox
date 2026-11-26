# Spec Critique: K4A — Enhanced Error Reporting for Household ID Adornment Lookups

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-26

## Summary

- **Critical Issues:** 0
- **High Issues:** 5
- **Medium Issues:** 6
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | FR-003 (invalid input) and FR-002/FR-004 ordering: the flowchart checks "input well-formed" BEFORE "dependency available", but no FR covers the case where input is malformed AND the dependency is simultaneously down. The precedence rule exists only in the Mermaid diagram, not in any testable requirement — an ambiguity about which reason wins when multiple causes co-occur. | High | 2, 3 | Add an FR making cause precedence explicit: invalid-input is evaluated and reported first, dependency-unavailable second, no-match last. FR-005 says "exactly one" reason but never states which one wins on overlap. |
| CG-002 | FR-004 covers dependency "unavailable or fails to respond within bounded retry," but no requirement defines behavior for a PARTIAL/degraded dependency response (e.g. dependency returns HTTP 200 with a malformed or empty body). This falls between "available" and "unavailable" and hits no defined branch. | High | 3 | Add an FR classifying a successful-status-but-unparseable dependency response — treat it as dependency-unavailable (transient) rather than silently as no-match. |
| CG-003 | No FR defines behavior when input-format validation itself cannot run or the request is structurally un-parseable at the transport layer (e.g. missing required field entirely, not merely malformed value). FR-003 assumes the request reaches format validation. | Medium | 3 | Clarify that a structurally un-parseable / missing-field request maps to the invalid-input-format reason, or define a separate rejection behavior. |

No issue rises to **Critical** (implementation-blocking / data-loss / security). The privacy and access-control requirements (FR-007, FR-008, NFR-004–006) are present and testable, which removes the usual security-blocker candidates.

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-006 | "the reason explains the cause ... in terms the agent can act on" / "enables the agent to determine an appropriate next step" | Not objectively measurable — "actionable" and "can act on" have no pass/fail criterion. An automated test cannot assert "actionability." | Given a no-results response with reason R, When R is inspected, Then R maps to exactly one entry in a defined, versioned reason→agent-action catalog, And every supported reason code has a non-empty catalog entry. (Requires defining that catalog — currently undefined.) |
| NFR-001 | "must not measurably degrade successful-lookup latency ... remains within its current p95 ... baseline p95 target is not supplied" | Untestable: no numeric baseline exists, so "within current p95" cannot be verified. Flagged [DECISION REQUIRED] but still unmeasurable as written. | Given 10k representative concurrent happy-path lookups, When measured, Then success-path p95 ≤ [BASELINE_MS] (baseline must be supplied before this is a test). |
| NFR-003 | "error rate exceeds 1% for 5 minutes, sustained" | Mostly measurable, but "error rate" is undefined — numerator/denominator unspecified (dependency errors ÷ total lookups? ÷ dependency calls?). Risk of ambiguous alerting. | Define error-rate = (dependency calls returning error or timeout) ÷ (total dependency calls) over a rolling 5-min window; alert when > 1%. |
| FR-009 | "exponential backoff (default: up to 3 attempts)" | "default" implies configurability but no FR defines the configurable bounds or min/max backoff interval — partially testable. | Specify initial backoff, multiplier, and max cap (e.g. 100ms base, ×2, cap 2s, max 3 attempts) so the retry envelope is deterministically testable. |

All ten FRs are written in Gherkin — no prose-FR violations. The issues above are vagueness within otherwise-conformant Gherkin, not format failures.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Response shape designed before agent needs known (NFR-008) | Yes — deferred to named-owner decisions; behavioral contract in interim | PASS | Correctly gated, not just listed. |
| Proto/contract change ripples to unknown consumers (NFR-009) | Partial — mitigation is "enumerate all consumers before rollout" | **FAIL** | Mitigation is a process promise with no owner, no deadline, and no FR. NFR-007 enforces additive-only shape, but nothing verifies the enumeration actually happened. This is the spec's own highest-rated risk and its mitigation is unenforceable. |
| Collapsing distinct failure causes (NFR-010) | Yes — FR-005 | PASS | Directly backed by a testable FR. |
| Privacy leakage via diagnostic detail (NFR-011) | Yes — FR-007, FR-008, NFR-004/005 + release gate | PASS | Well covered. |
| Undefined front-end involvement (NFR-012) | Yes — out-of-scope for Phase 1, gated on decision | PASS | Cleanly scoped out. |
| No added latency on success path (NFR-001) | Partial — target stated but no baseline | **FAIL** | Risk acknowledged, mitigation not verifiable (no baseline number). Flagged [DECISION REQUIRED] but ships without an acceptance threshold. |

---

## 4. Missed Edge Cases

1. **Multiple co-occurring causes:** Malformed input while dependency is down — which reason is returned? Precedence lives only in the flowchart (CG-001).
2. **Dependency returns 200 with empty/garbage body:** Classified as no-match or dependency-unavailable? Undefined (CG-002).
3. **Structurally un-parseable request / missing required field:** Not covered by FR-003's "malformed value" framing (CG-003).
4. **Rate-limit with no Retry-After header:** FR-009 says "honor the Retry-After indication" but doesn't define fallback when a 429 arrives WITHOUT a Retry-After value.
5. **Retry budget exhausted mid-backoff vs. a hard failure:** No requirement distinguishes "all 3 attempts failed fast" from "3 attempts timed out slowly" for NFR-002's bounded-wait guarantee — the worst-case total wait time is unstated.
6. **Concurrent/duplicate lookups for the same household:** No idempotency or de-dup consideration; irrelevant if stateless, but unconfirmed.
7. **Partial match / ambiguous multiple matches:** FR-001 assumes a single matching record. Behavior for >1 match or a low-confidence match is undefined (though matching logic is out of scope, the RESPONSE shape for it is not).
8. **Masking failure in FR-008:** What happens if the masking/hashing step itself fails — does the system fail-closed (drop the log) or fail-open (risk raw PII)? Privacy-critical gap with no stated default.
9. **Backward-compat negative test:** FR-010 asserts existing callers "continue to function unchanged" but no FR specifies the regression test against a real legacy consumer — and the consumer list is admittedly un-enumerated (NFR-009).
10. **Reason-code localization/versioning:** If reasons are structured codes (NFR-007), no requirement covers how a new reason code is introduced without breaking consumers that switch on a closed enum.
11. **Telemetry sink unavailable:** If the diagnostic telemetry write (FR-008) fails, does the lookup response still return? No requirement decouples the user-facing response from the operational logging path.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-010 all use Given/When/Then. |
| Document contains only Sections 0–6 | **FAIL** | Document ends with an **"Appendix: Engineering Reference"** containing an "Affected Services" narrative beyond Sections 0–6. This is NOT a sanctioned Revision Summary nor a one-line `> **Label:**` metadata trailer — it is a multi-paragraph narrative appendix with its own per-service findings, exactly the kind of stray content the format rule is meant to catch. Flag and remove (or relocate the KG grounding into Section 1). |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list items with prose descriptions; no attribute tables or type declarations. (Note: each bullet carries a "Source:" label in prose, which is acceptable as bullet content, not a type table.) |

---

## 6. Recommendations

1. **Remove or relocate the "Appendix: Engineering Reference."** It violates the Sections 0–6 constraint (Format Compliance FAIL). The grounded affected-service context belongs inside Section 1's KG Cross-Check Note or an eng-plan artifact, not as a trailing appendix on the spec.
2. **Add a cause-precedence FR (CG-001)** so the "exactly one reason" rule in FR-005 is deterministic when invalid-input, dependency-down, and no-match overlap. Precedence must not live only in the Mermaid diagram.
3. **Make NFR-009's consumer-enumeration mitigation enforceable** — assign a named owner and a pre-rollout gate; it is the spec's highest-rated risk with the weakest mitigation. (Note: the independent KG cross-check re-found only `household-id-adornment-api` (high) and `household-id-adornment-proto` (medium) and did NOT re-find `frontend-mos`; this corroborates keeping front-end out of Phase 1 and reinforces that the proto consumer list is genuinely unverified.)
4. **Supply the NFR-001 baseline p95 number** or explicitly convert it to a decision with an owner and a deadline — otherwise the success-path latency guard is untestable at release.
5. **Define behavior for degraded/partial dependency responses (CG-002)** and rate-limit-without-Retry-After (edge case 4).
6. **Specify the privacy fail-mode (edge case 8)** — masking failure must fail-closed by default; this is privacy-critical and currently silent.
7. **Decouple telemetry-write failure from the user response (edge case 11)** so a logging-sink outage never blocks a lookup response.
8. **Tighten FR-006 actionability into a testable reason→action catalog** so QA can assert it.