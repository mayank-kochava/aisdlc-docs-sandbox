# Spec Critique: K4A MCP — Date Format Standardization (YYYY-MM-DD)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-18

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The entire spec is gated on discovery that has not happened: the current date format, the exact date field locations, AND the owning backend repository are all unknown. BR-2, the reconciliation note, and the KG cross-check all confirm engineering cannot even identify the surface to change. This blocks implementation outright — there is no concrete transformation to build until discovery lands. The spec acknowledges this but ships FRs that assume discovery outputs exist. | Critical | 1, 3, 4 | Gate the requirement behind a mandatory discovery deliverable (signed Date Field Inventory + confirmed owning repo) as an explicit precondition/exit-criteria before any FR is considered implementable. Treat discovery as a formal Phase 0 with its own acceptance. |
| CG-002 | Owning-repository ambiguity is unresolved: the Context Seed names one backend, but the independent KG lookup surfaces TWO candidates — `k4a-mcp-server` (high confidence) and `kochava-k4a-mcp-server` (medium, flagged "may own date format"). DEC-004 defers this. Implementing the serialization change against the wrong repo = change invisible to the customer or applied to the wrong service. This is a correctness/data-flow risk that blocks a code change. | Critical | 1 | Resolve DEC-004 before engineering starts. Confirm which of `k4a-mcp-server` / `kochava-k4a-mcp-server` emits the customer-facing date serialization (e.g., by tracing the actual response path for the gaming customer), and record the single confirmed repo in the spec. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "MUST NOT materially degrade… within the pre-change p95 baseline plus a negligible margin" | "Materially," "negligible margin," and an unsupplied baseline are untestable. No concrete number exists (DEC-005). | Given a representative concurrent load of [X] req/s, When the date-formatting step is enabled, Then response-time p95 does not exceed the pre-change p95 baseline of [Y]ms by more than [Z]ms. |
| NFR-004 | "an elevated rate of such failures SHOULD alert when the failure rate exceeds a defined threshold for a sustained window" | "Elevated," "defined threshold," "sustained window" are placeholders (DEC-006). Not testable. | Given formatting-failure indications are recorded, When the failure rate exceeds [N]% over a [M]-minute window, Then an alert fires to [owner]. |
| FR-005 | "rendered according to that decision" | The decision (BR-4/DEC-002) does not exist yet, so there is no concrete expected output to assert. The Then clause is a pointer to an unmade decision. | Once DEC-002 resolves: Given a date field source value carries a time/timezone component, When formatted, Then the value is rendered as YYYY-MM-DD [date-only | truncated per rule X], and time-of-day is [dropped | …] uniformly across fields sharing that semantic. |
| FR-010 | "does not fail silently… insofar as discovery coverage is documented" | "Insofar as discovery coverage is documented" is an escape hatch, not an assertion. A consumer missed by discovery still breaks silently; the clause only requires documenting that you might have missed it. Not a testable guarantee. | Given discovery has produced a Downstream Consumer Registry, When the format change is released, Then every registered consumer has a recorded compatibility verdict, and the registry's coverage boundary (what was and was not searched) is documented and signed off. |
| FR-008 / NFR-006 | "in a way that could diverge from or silently override" | "Could diverge" is not a crisp pass/fail. | Given consumption path P has one authoritative formatting layer L, When any other layer processes the date value, Then it passes the value through byte-for-byte without re-parsing or re-formatting. |

All FRs are structurally in Gherkin (see Section 5), but the ones above assert against undecided inputs or vague NFR thresholds and are not yet testable.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Unknown current format & field locations | Yes — BR-2, FR-001/002, Date Field Inventory | PASS (process) | Mitigation is "do discovery first." Sound, but nothing in the FRs is executable until it completes — see CG-001. Mitigation is a precondition, not an in-spec requirement with acceptance criteria. |
| Silent breakage of downstream consumers | Partial — FR-009, FR-010, NFR-005 | FAIL | Mitigation reduces to "document that an unidentified consumer might exist." FR-010's "insofar as discovery coverage is documented" does NOT prevent silent breakage of a missed consumer; it only records the gap. No versioning, no dual-format transition window, no consumer opt-in. For a change to an existing output contract this is insufficient. |
| Fix lands in wrong layer (backend vs console) | Yes — BR-3, FR-008, NFR-006 | PASS | Adequately addressed pending DEC-001. |
| Wrong repository touched (backend name collision) | Named only — see reconciliation note | FAIL | The risk is acknowledged and a decision is flagged (DEC-004), but there is NO requirement that determines *how* the correct repo is confirmed. "Mitigated when… resolved" restates the risk as its own solution. Needs a concrete verification step (trace the live response path). This is CG-002. |
| Ambiguous time/timezone handling | Yes — BR-4, FR-005, NFR-003 | PASS (pending decision) | Mitigation exists but depends on DEC-002 being made. |
| Cross-layer coordination / release sequencing | Partial — final bullet of Risk Mitigation | FAIL | Says release "MUST be sequenced… in a compatible order" but defines no sequence, no owner, no gate. No FR enforces ordering. "Must be sequenced" with no defined sequence is not a mitigation. |

---

## 4. Missed Edge Cases

1. **Partial/ambiguous dates (missing day or month):** FR-001 assumes every date resolves to Y+M+D. What about a value that is only a year (`2024`) or year-month (`2024-06`)? Is that "unparseable" (FR-004) or padded? Undefined.
2. **Out-of-range / sentinel dates:** Values like `0000-00-00`, `9999-12-31`, Unix epoch 0, or placeholder "no date" sentinels. Are these valid calendar dates to reformat, or formatting failures?
3. **Date ranges and arrays of dates:** A field holding a list of dates or a `start–end` range. FR-002 says "every enumerated date field" but doesn't address multi-value or composite date fields.
4. **Dates embedded in free-text / nested structures:** A date appearing inside a string field, message, or nested JSON object — discovery may enumerate it or miss it. Behavior undefined.
5. **BCE / pre-1000 years:** Four-digit year assumption breaks for years < 1000 (would `0500-01-01` be zero-padded?) and has no BCE handling. Low likelihood but undefined.
6. **Timezone date-boundary shift:** If a source value is `2024-06-30T23:00:00Z` and the consumer is UTC+2, the calendar date differs (July 1 locally). FR-005 defers this to DEC-002, but the edge case — that reducing to date-only can change the date depending on timezone rule — is not called out as a correctness hazard.
7. **Console re-formatting from a cached/old backend response:** FR-007 assumes console reads live backend output. If the console caches a pre-change response, it renders the old format after cutover. Cache invalidation not addressed.
8. **Concurrent deploy window (backend changed, console not yet):** The cross-layer sequencing risk is named but no FR covers the transient window where backend emits YYYY-MM-DD and console still expects the old format.
9. **Formatting-failure flood / recording backpressure:** FR-004 records a failure indication per field. A response with many malformed date fields (or high-volume traffic) could flood the recording channel. NFR-002 bars external calls in-line, but the recording mechanism's cost is unspecified.
10. **Idempotency on already-correct values:** If a source value is ALREADY `YYYY-MM-DD`, is it passed through or re-parsed? Re-parsing an already-correct value risks the timezone shift in edge case #6. Not stated.
11. **Rollback of in-flight / already-returned responses:** Rollback (Section 6) only affects new responses — correctly stated — but there is no guidance for a consumer that already stored mixed-format data during the brief post-cutover window.
12. **Non-Gregorian / locale calendar source values:** If any source date is stored in a non-Gregorian representation, conversion to ISO 8601 Gregorian is a semantic change, which Section 6 says is out of scope — creating a gap if such a value exists.
13. **Discovery produces an empty inventory (no date fields found):** The flow handles "no date fields present" per-response, but there is no defined outcome if discovery concludes there are *no* date fields at all — does the requirement become a no-op, and is that an acceptable close-out?

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-011 are all in Gherkin blocks. Several assert against undecided inputs (see Section 2), but structurally they comply. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present (Section 0 not emitted, but no extra/appendix sections beyond 6). The "Affected-Services Reconciliation" block is a subsection of Section 1, not a stray appendix. No trailing Revision Summary or metadata trailer present. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list with no attribute tables or type declarations. (Note: tables appearing in Section 6 Out-of-Scope are permitted there; the constraint applies only to Section 4.) |

---

## 6. Recommendations

1. **Resolve DEC-004 / CG-002 first:** Confirm which single backend repo (`k4a-mcp-server` vs `kochava-k4a-mcp-server`) owns the customer-facing date serialization by tracing the live response path. Record the confirmed repo in the spec. Nothing else is safe to start.
2. **Formalize discovery as a gated Phase 0 (CG-001):** Make the signed Date Field Inventory + current-format-per-field a hard exit criterion before any FR is implementable. Define what "discovery complete" means and who signs off.
3. **Make DEC-002 (time/timezone rule) a blocker for FR-005:** Without it, FR-005 and the date-boundary edge case (#6) have no deterministic answer. Resolve before implementation.
4. **Strengthen downstream-breakage mitigation (Risk Audit FAIL):** Replace FR-010's "document the coverage gap" posture with a real transition strategy — e.g., versioned contract, dual-format window, or explicit consumer sign-off — so a missed consumer cannot break silently.
5. **Define the release sequencing concretely:** Convert "MUST be sequenced in a compatible order" into an ordered, owned release plan (backend-then-console or gated), and cover the transient cross-layer window (edge cases #7, #8).
6. **Supply the deferred thresholds (DEC-005, DEC-006):** Provide the p95 baseline + margin (NFR-001) and the failure-rate threshold + window (NFR-004) so these become testable.
7. **Specify behavior for partial/sentinel/range dates (edge cases #1–#3, #10):** Decide whether year-only, year-month, sentinels, multi-value fields, and already-ISO values are reformatted, passed through, or treated as failures.
8. **Assign DEC-007 rollback ownership and timing** so rollback is actionable under an elevated-failure trigger.