# Spec Critique: K4A Factory Floor Operations Support

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-26

## Summary

- **Critical Issues:** 1
- **High Issues:** 5
- **Medium Issues:** 4
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The entire functional core (FR-005, the actual factory floor workflow) is undefined and explicitly "not-yet-buildable." There is no definition of task, workflow, record, or success metric. An implementation agent cannot build the product value — only an access-control shell around an empty behavior. This blocks implementation of anything beyond scaffolding. | Critical | 1, 3 (FR-005), 6 | Resolve the Phase 0 discovery decisions (DEC-001–DEC-008) before engineering starts. The spec correctly gates on this, but it remains a hard blocker, not a resolvable spec defect. |
| CG-002 | FR-004 (fail-closed on auth-source unavailability) specifies denial but does not define what "unreachable" means operationally (timeout threshold, retry count, circuit-breaker). Without a bound, a slow auth source could hang the request indefinitely rather than failing closed. | High | 3 (FR-004) | Add a timeout bound to FR-004 (e.g., "When the authoritative source does not respond within N ms"). |
| CG-003 | The user flow (Section 2) routes both validation errors (J) and dependency errors (K) back to node F (the factory floor view), but does not define state recovery — whether the user's in-progress input is preserved or discarded on return. Risk of silent data loss of user-entered input on a transient dependency failure. | High | 2 | Define whether in-progress input is retained when returning to F after error K; add an FR for input-state preservation. |
| CG-004 | FR-008 defines bounded retry with backoff, but no requirement defines idempotency. If a state-changing action times out after the write committed downstream but before K4A received the ack, a retry could double-apply the action. "No partial state" (FR-007) does not cover duplicate full application. | High | 3 (FR-007, FR-008) | Add an idempotency-key / dedupe requirement for retried state-changing actions. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-005 | "K4A applies the defined result / confirms the outcome to the user" | "Defined result" and "the defined factory floor action" are placeholders — not testable; no concrete pass/fail condition exists. (Acknowledged by the spec as a blocker, cited here for completeness.) | Cannot be made testable until the workflow decision (BR-01) lands. Until then, assert only the guard: `Given the workflow is undefined, When an agent attempts to implement FR-005, Then it blocks on DEC-002 and does not infer behavior.` |
| FR-004 | "...is shown a try-again-later message distinct from the not-authorized message" | "Distinct" is testable, but the trigger ("is unreachable") lacks a measurable boundary. | `Given the auth source does not respond within 2000ms, When an authenticated user requests the factory floor view, Then K4A returns a 503-class try-again response distinct from the 403 not-authorized response.` |
| NFR-006 | "alert when error rate exceeds 1% for 5 minutes sustained" | This one IS measurable (good), but the latency target is explicitly deferred ("cannot be set until... defined"). No testable latency SLA currently exists. | Flag as blocked on DEC-001; no rewrite possible until the success metric lands. |
| NFR-007 | "its own separate rate limit rather than one shared value" | "Separate rate limit" has no numeric value — "engineering default to be set during planning" means it is currently untestable. | `Given the read path receives >R requests/min, Then requests beyond R are rejected with 429; And the write path limit W is enforced independently of R.` (R, W pending BR-01.) |

Note: The FRs are commendably structural and mostly measurable at the shell level. The vagueness is inherent to the undefined product, not sloppy wording — but it remains untestable all the same.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Undefined requirement (primary) | Partial — gated, not resolved | FAIL | Mitigation = "block and wait for decisions." This defers the risk rather than mitigating it; the spec cannot proceed to value until DEC-001–DEC-008 resolve. Honest, but not a mitigation. |
| Wrong/shallow codebase match (blast-radius mismatch) | Yes — discovery step named | PASS | Correctly flags name-only match and requires engineering confirmation of the owning codebase. |
| Multi-service blast radius vs single-service evidence | Yes — contradiction flagged | PASS | Spec explicitly surfaces that the "multi-service" rationale is unsupported by KG evidence (now two name-matched services, still no behavior evidence). |
| Domain complexity (regulatory/safety/compliance) | Deferred to DEC-008 | FAIL | No requirement exists to handle regulatory/safety constraints; it is entirely contingent on an unanswered decision. For a factory floor domain, safety/compliance is a plausible hard constraint that could invalidate the architecture — needs at least a placeholder requirement to confirm/deny applicability before Phase 1. |
| Restricted data class in records (NFR-003) | Deferred to DEC-006 | FAIL | Whether records contain a restricted data class is undefined; no storage/masking requirement is committed. Acceptable given unknown records, but remains an open risk gap. |
| Audit-write failure | Yes (NFR-005) | PASS | Async write, immediate alert on failure, no silent failure, 12-month retention — well specified. |
| Duplicate application on retry | No | FAIL | No idempotency mitigation for retried state-changing actions (see CG-004). |

---

## 4. Missed Edge Cases

1. **Retry non-idempotency:** A state-changing action times out after the downstream commit but before ack; FR-008 retry re-applies it. No dedupe/idempotency key defined.
2. **In-progress input loss:** User enters data, hits dependency error (flow node K), is returned to view F — unclear whether input is preserved (CG-003).
3. **User navigates "Back" / abandons mid-action:** No requirement covers browser back, tab close, or session timeout during a state-changing action. Does the action commit, roll back, or leave an orphan?
4. **Concurrent eligibility revocation:** A user's eligibility is revoked mid-session between FR-003 (view granted) and FR-005 (action). FR-002 re-derives per request, which helps, but no requirement defines what happens to an action already in flight.
5. **Auth-source slow (not down):** FR-004 covers "unreachable" but not "degraded/slow." A high-latency auth source is a distinct failure mode with no defined bound (CG-002).
6. **Partial audit-trail loss:** NFR-005 alerts on audit-write failure but does not define whether the primary action is reversed, flagged, or proceeds un-audited. Compliance gap if audit is mandatory.
7. **Rate-After header malformed/absent:** FR-008 honors `Retry-After` "when carried," but no behavior is defined for a malformed or missing `Retry-After` on a 429 beyond the fixed backoff default — acceptable but should be explicit.
8. **Two owning codebases:** KG evidence now shows TWO backend services (`factory-floor`, `kochava-factory-floor`), both high-confidence name matches. No requirement or discovery step picks the authoritative one; work could land in the wrong repo.
9. **Ineligible user data leakage via error timing/shape:** FR-002 returns no data, but no requirement prevents inferring record existence via differential error responses or timing. Minor, but relevant for a role-gated capability.
10. **Audit trail for failed/denied actions:** NFR-005 logs state-changing actions; it does not require logging denied-access attempts (403s), which is often a security/compliance expectation for a gated capability.
11. **Downstream returns success but malformed/unexpected payload:** FR-007 covers timeout/unavailable/rate-limited, not a 200 with a semantically invalid body. No validation-of-response requirement.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-008 are all authored in valid Gherkin blocks. |
| Document contains only Sections 0–6 | FAIL | The document contains an **"## Appendix: Engineering Reference"** section after Section 6. This is a multi-subsection narrative appendix ("Affected Services" with prose role descriptions and discovery recommendations) — not a sanctioned trailing Revision Summary and not a one-line `> **Label:**` metadata blockquote. It is exactly the kind of stray narrative appendix that must be flagged. Fold the owning-codebase question into Section 1 / Phase 0 discovery and remove the standalone appendix. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list of business concepts with prose source/rationale; no attribute tables or type declarations. (Note: the trailing blockquote after the list is explanatory and does not introduce a table/type.) |

---

## 6. Recommendations

1. **Resolve the Phase 0 discovery gate (CG-001).** Answer DEC-001–DEC-008 — workflow, user population, record set, integration need, success metric, compliance constraints — before any Phase 2 behavior work. Nothing of product value is buildable until then.
2. **Confirm the owning codebase.** Two high-confidence, name-matched backend services (`factory-floor`, `kochava-factory-floor`) exist with zero behavior evidence (no endpoints/tables/symbols). Reconcile against the unsupported "multi-service blast radius" claim and pick the authoritative repo before Phase 1.
3. **Add an idempotency requirement (CG-004)** for retried state-changing actions to prevent double-application on timeout-then-retry.
4. **Bound the auth-source "unreachable" condition (CG-002)** with an explicit timeout in FR-004 so fail-closed cannot hang.
5. **Define input-state recovery (CG-003)** for the flow's error-return paths (nodes J and K back to F) so user input is not silently lost.
6. **Promote the compliance/safety risk (DEC-008) to a blocking confirm/deny** before Phase 1, given factory-floor regulatory exposure could invalidate the architecture.
7. **Remove the "Appendix: Engineering Reference" section** to restore Sections 0–6 compliance; relocate its content into Section 1's service-grounding note and the Phase 0 discovery list.
8. **Add requirements for denied-action audit logging (NFR-005 gap)** and response-payload validation (FR-007 gap).