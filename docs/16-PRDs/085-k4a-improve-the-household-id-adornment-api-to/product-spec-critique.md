# Spec Critique: K4A — Enhanced Error Reporting for Household ID Adornment Lookups

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-26

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Ordering collision is undefined.** The flow (Section 2) implies strict precedence: input-format → dependency availability → match. But FR-002 through FR-005 do not define behavior when two conditions are simultaneously true (e.g. malformed input AND the dependency is down, or an input that passes surface validation but is semantically unresolvable). FR-005 mandates "exactly one" reason yet never states the resolution rule the API uses to pick which one. Implementers will each choose differently. | Critical | §2, FR-002–FR-005 | Add an explicit precedence requirement: "Given multiple empty-result conditions apply simultaneously, Then the API returns the reason according to the fixed precedence: invalid input format > lookup dependency unavailable > no match found." Ground this in the Mermaid ordering so flow and FRs agree. |
| CG-002 | **"Dependency unavailable" vs. "slow-but-eventually-empty" is a dead end.** FR-009 bounds retries, but there is no defined behavior for the case where the dependency *responds successfully after retry* with a genuine no-match, versus partial/degraded responses. FR-004 fires only when attempts are "exhausted," but a dependency that returns a 200 with a malformed/empty payload (not a timeout, not a clean no-match) has no defined reason classification and would silently fall through to "no match found" — misleading the agent into telling the customer no record exists when the truth is a data-integrity failure. | Critical | FR-004, FR-009 | Add an FR for degraded/ambiguous dependency responses (malformed payload, partial response, unexpected status): classify as "lookup dependency unavailable" (transient), never as "no match found." |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-006 | "the reason explains the cause ... in terms the agent can act on ... enables the agent to determine an appropriate next step" | "Actionable" and "can act on" are subjective and untestable. No automated test can assert "actionability." | `Given an empty-result response with a diagnostic reason, When the response is validated, Then the reason field equals one of the approved enumerated reason codes AND carries a non-empty human-readable message AND the (reason code → message) mapping matches the approved content table defined by the content-decision owner.` (Requires the content table to exist — see RM audit.) |
| NFR-001 | "must not measurably degrade successful-lookup latency ... remains within its current p95 ... baseline p95 target is not supplied" | Untestable: the baseline p95 is admitted missing and marked `[DECISION REQUIRED]`. "Measurably degrade" has no threshold. | `Given representative concurrent load at the documented baseline, When the happy path (FR-001) is exercised, Then p95 latency does not exceed <BASELINE_P95_MS> by more than <N>%.` Blocked until baseline is supplied. |
| FR-003 | "whose input is malformed or fails input-format validation" | "Malformed" / "fails input-format validation" is undefined. The actual validation rules (length, charset, allowed formats) are never specified, so the trigger for FR-003 is untestable. | Define the validation ruleset, then: `Given an input violating rule <X> of the documented input-format spec, When the API validates it, Then it returns empty-result with reason "invalid input format".` |
| FR-004 | "the lookup dependency is unavailable or fails to respond within its bounded retry attempts" | "Unavailable" conflates timeout, connection refused, 5xx, and rate-limit. Each needs a distinct testable trigger. | Split into discrete Given clauses per failure mode (timeout / 5xx / connection error / rate-limit-exhausted) each asserting the same reason. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| NFR-008 — Response shape designed before agent needs known | Deferred to a `[DECISION REQUIRED]` with no named owner | **FAIL** | Mitigation is conditional on decisions that have *no assigned human owner and no due date*. "Mitigated when … have named human owners" — but no owner is named anywhere. The mitigation is a promissory note, not a requirement. |
| NFR-009 — Proto change ripples to unknown consumers | Partial — states "mitigated when complete consumer set enumerated" but no FR enumerates them | **FAIL** | The consumer list is explicitly *not enumerated* ("not enumerated in available evidence"). FR-010/NFR-007 assert additive-compatibility but no requirement forces the enumeration-and-verification step before rollout. This is a Critical-adjacent gap for a contract change. |
| NFR-010 — Collapsing distinct causes | Yes — FR-005 directly enforces | **PASS** | Well-handled and testable. |
| NFR-011 — Privacy leakage via diagnostic detail | Yes — FR-007, FR-008, NFR-004, NFR-005, release gate | **PASS** | Strong. Release-gate privacy review is a concrete mitigation. |
| NFR-012 — Undefined front-end involvement | Yes — scoped out of Phase 1 pending decision | **PASS** | Deferral is explicit and gated. Acceptable. |
| BR-4 Privacy boundary marked `[DECISION REQUIRED]` | Enforced by FRs but still flagged undecided | **FAIL** | A privacy boundary that is both enforced by FR-007/008 *and* marked `[DECISION REQUIRED]` is contradictory — either the boundary is settled (drop the flag) or it is open (then FR-007/008 are built on sand). Resolve the contradiction. |

---

## 4. Missed Edge Cases

1. **Simultaneous condition precedence:** No rule when input-invalid + dependency-down + no-match all apply (see CG-001).
2. **Degraded dependency 200 with garbage payload:** Not classified; risks false "no match found" (see CG-002).
3. **Concurrent / duplicate lookups:** No statement on idempotency or whether repeated identical lookups during a dependency outage each consume full retry budgets (retry-storm amplification against an already-failing dependency).
4. **Retry-After larger than caller timeout:** FR-009 honors `Retry-After`, but if the indicated wait exceeds the overall request budget, behavior is undefined — does the API wait, or immediately return "dependency unavailable"? Unspecified.
5. **Partial match / ambiguous multi-match:** The flow treats match as binary (yes/no). What if the lookup returns *multiple* candidate households? Neither "no match" nor "match" cleanly applies; no reason exists for ambiguity.
6. **Empty vs. null vs. whitespace input:** FR-003 covers "malformed," but a truly empty/absent input parameter may short-circuit before validation. Is an empty query "invalid input format" or a different error? Undefined.
7. **Reason-set versioning:** If the reason taxonomy later expands (explicitly anticipated in §6), existing consumers may receive a reason code they don't recognize. No requirement defines forward-compatible handling of unknown reason codes by consumers.
8. **Telemetry write failure:** FR-008 governs *what* is logged, but not what happens if the privacy-safe telemetry write itself fails — does the lookup response still succeed? (It should; confirm the telemetry path is non-blocking.)
9. **Masking collision / reversibility:** NFR-005 requires "masked" identifiers for troubleshooting. No requirement forbids a masking scheme that is reversible or that allows correlation back to a customer across log entries — a weak mask defeats the privacy intent.
10. **Access-control on the new fields specifically:** NFR-006 preserves existing gating, but doesn't state whether the *new diagnostic fields* could leak information to a caller authorized for the lookup but not for diagnostic detail (e.g. tiered agent roles). Diagnostic detail may warrant its own gate.
11. **Rate-limit on the enhanced path itself:** No requirement caps how often a caller can trigger diagnostic generation; a hostile caller could probe the "invalid input format" path to fingerprint validation rules.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-010 are all in valid Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **`## Appendix: Engineering Reference`** section between Section 6 and the Revision Summary. This is a multi-section narrative appendix ("Affected Services" with prose role/kind/confidence descriptions) — exactly the kind of extra content the Sections 0–6 rule exists to catch. It is NOT a sanctioned trailing Revision Summary nor a one-line `> **Label:**` metadata blockquote, so the exceptions do not apply. The trailing `## Revision Summary` section itself is EXPECTED and is **not** flagged. Recommend moving the Engineering Reference content into eng-plan, or demoting it so Sections 0–6 + Revision Summary are the only top-level structure. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list; each item carries a Source/Matters narrative but no attribute tables or type declarations. Compliant. |

---

## KG Cross-Check Findings (grounding reconciliation — not document content)

- The spec's own "KG Cross-Check Note" (§1) and Appendix claim three affected services: `household-id-adornment-api` (high), `household-id-adornment-proto` (contract), and `frontend-mos` (low confidence). The independent cross-check re-found **only** `household-id-adornment-api` (high) and `household-id-adornment-proto` (medium). **`frontend-mos` was NOT re-found by the independent lookup.** This is consistent with the spec's own characterization of `frontend-mos` as low-confidence and out of scope for Phase 1, so it is not a contradiction — but it *strengthens* the case to keep front-end work out of scope and confirms the consumption-surface decision (§1 `[DECISION REQUIRED]`) remains genuinely unresolved. No spec claim about data flow, joins, config, or existing endpoints contradicts the KG evidence.

---

## 6. Recommendations

1. **Resolve CG-001 (precedence rule).** Add an explicit, testable precedence ordering for simultaneous empty-result conditions; align the Mermaid flow and FR-002–FR-005 to it. **Blocking.**
2. **Resolve CG-002 (degraded dependency response).** Add an FR classifying malformed/partial/degraded dependency responses as transient "lookup dependency unavailable," never "no match found." **Blocking.**
3. **Fix Format Compliance FAIL:** remove or relocate the `## Appendix: Engineering Reference` so the document is Sections 0–6 + Revision Summary only.
4. **Assign named human owners and due dates** to every `[DECISION REQUIRED]` (content, consumption-surface, reason-set, baseline p95, BR-4 privacy boundary). NFR-008 and NFR-009 mitigations are inert until owners exist.
5. **Force the proto-consumer enumeration (NFR-009) as a pre-rollout gate FR**, not a prose aspiration — a contract change with an unenumerated consumer list is the single highest-risk item here.
6. **Make FR-006 testable** by replacing "actionable" with a concrete reason-code→approved-message mapping table (dependent on the content-decision owner).
7. **Supply the baseline p95** so NFR-001 becomes measurable.
8. **Specify input-validation rules** so FR-003's trigger is concrete and testable.
9. **Address the remaining edge cases** (ambiguous multi-match, unknown-reason-code forward compatibility, telemetry non-blocking, masking non-reversibility, tiered diagnostic access-control, probe rate-limiting) before engineering, in roughly that priority order.