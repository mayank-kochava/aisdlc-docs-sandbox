# Spec Critique: Household ID Lookup Diagnostic Enrichment

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-26

## Summary

- **Critical Issues:** 0
- **High Issues:** 4
- **Medium Issues:** 6
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | FR-005 is unfalsifiable. "The agent can determine why the lookup returned nothing... for the common failure cases" has no definition of "common failure cases" and no way to pass/fail a test. It smuggles an untestable success claim into a Gherkin block. | High | §3 FR-005 | Either drop the resolution claim from the FR (it's a value statement, not a behavior) or enumerate the specific failure categories the diagnostics must cover and assert presence of each. |
| CG-002 | FR-003/FR-004 Then-clauses are written from the point of view of what "Support can" do ("Support can identify which specific step"), which is a human cognitive outcome, not an observable system behavior. These cannot be automated. | High | §3 FR-003, FR-004 | Rewrite so the assertion is on the response payload (e.g. "the diagnostic information contains a non-empty step identifier"), not on what a human infers. |
| CG-003 | No requirement defines behavior when the lookup returns no result **because the input itself was empty/malformed** (e.g. blank household ID, missing address). FR-004 says "the input values that were used/searched" are echoed — but an empty-input no-result has no meaningful searched values, and the flow diagram has no branch for it. Dead end. | High | §2, §3 | Add an FR covering the empty/invalid-input no-result case: what the step identifier and searched-values fields contain when nothing was validly searched. |
| CG-004 | NFR-005 flags that searched values may contain customer identifiers/address fields, but **no FR enforces masking or exposure limits.** The risk is acknowledged and the decision deferred, yet FR-004/FR-010 unconditionally require echoing raw lookup data and "the raw lookup data underlying that summary." If this ships before the [DECISION REQUIRED] is resolved, raw customer input lands in the diagnostic response and any logs. | High | §3 FR-004/FR-010, §5 NFR-005 | Add an FR making the raw-value echo conditional on the masking decision, OR gate FR-004/FR-010 behind resolution of the NFR-005 decision. Do not let an unresolved data-exposure decision coexist with an unconditional "echo raw data" requirement. |

No Critical issues found — the change is genuinely additive, single-service, and error-branch-only, and the resilience path (FR-008/NFR-003) is well specified. The four items above are High: real testability and data-exposure gaps that should be closed before engineering, but none block implementation or cause guaranteed data loss on their own.

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-005 | "the agent can determine why the lookup returned nothing without escalating to engineering for the common failure cases" | "Common failure cases" undefined; "can determine" is a human outcome, not a measurable response property. | `Given a no-result response, Then the diagnostic information contains a non-empty failure-reason summary and a non-empty failed-step identifier and a non-empty searched-values set` (assert on payload, not on the human). |
| FR-003 | "Support can identify which specific step/stage of the resolution process failed to produce a match" | Asserts human identification; not automatable. | `Given a no-result response, Then the diagnostic information contains a step/stage identifier field, And that field value is one of the defined resolution stage names, And it is not empty.` |
| FR-004 | "Support can see the input values that were used/searched during the lookup" | Human-centric; silent on empty-input case. | `Given a no-result response for a non-empty input, Then the diagnostic information contains the searched input values, And each value present in the request appears in the searched-values set.` |
| NFR-002 | "must not add material latency" (prose standalone sentence); the +10ms p95 figure is tagged "Working default... adjust during eng-plan" | "Material latency" is vague; the one concrete number is explicitly provisional, so there is no committed threshold to test against. | `Given the no-result path under representative concurrent load, When diagnostics are generated, Then added p95 overhead ≤ 10ms` — and remove "adjust during eng-plan" or move it to a decision note so a firm number exists. |

Note: all FR-001 through FR-010 are in valid Gherkin structure. The testability defects are in the *content* of the Then-clauses (human outcomes / undefined terms), not in the format.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Proto/contract change ripples to unknown consumers | Yes — NFR-001, FR-006, FR-007 | PASS | Additive/optional required and verifiably assertable. |
| Lookup may not have discrete, nameable steps | Yes — FR-003 flagged as assumption; fallback to searched-values-only | PASS | Fallback behavior is stated; still carries an open [DECISION REQUIRED] in §4. |
| Diagnostic generation failure degrades core lookup | Yes — NFR-002, NFR-003, FR-008 | PASS | Forced-failure test is specified. |
| Over-broad / sensitive diagnostic payload | Partially — NFR-005 names it, but no FR enforces masking | **FAIL** | The risk is acknowledged and then deferred, while FR-004/FR-010 unconditionally mandate echoing raw data. Acknowledgment without an enforcing requirement = unmitigated. See CG-004. |
| Empty/invalid-input no-result | No | **FAIL** | Not listed as a risk and not covered by any FR. See CG-003. |

---

## 4. Missed Edge Cases

1. **Empty / malformed input no-result:** What do the step-identifier and searched-values fields contain when the lookup had nothing valid to search? Undefined.
2. **Partial / concurrent multi-step failure:** If resolution fans through several steps and more than one "fails," FR-003 assumes a single failed step. No requirement defines reporting when the no-result is the cumulative outcome of multiple stages.
3. **Diagnostic generation partially succeeds:** NFR-003 says degrade "gracefully" and must not "leak partial or malformed diagnostic data," but no FR defines the partial-success case — is a half-built diagnostic omitted entirely, or emitted partial? FR-008 only covers total build failure.
4. **Sensitive-value masking before echo:** Raw searched values echoed unmasked (CG-004) — no edge handling for identifier/address fields that should be redacted.
5. **Oversized diagnostic payload:** "Raw lookup data" (FR-010) could be large. No bound on diagnostic size; could inflate the no-result response materially.
6. **Backward-compat for a consumer that strictly validates/rejects unknown fields:** FR-006/FR-007 assume tolerant deserializers. A consumer with strict schema validation (reject-on-unknown) would break even on an "additive optional" field. Not addressed.
7. **Logging of diagnostics:** NFR-005 mentions "any associated logging" but no FR governs whether/where diagnostics are logged, or retention — the same raw-value exposure risk applies to logs, not just the response.
8. **Localization / human-readable summary language:** FR-010 requires a "human-readable summary" but doesn't define language, format, or whether it's a free-text string vs. a coded reason — affects both testability and i18n.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-010 all use valid Given/When/Then blocks. Content-level testability issues are logged in §2, not here. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"## Appendix: Engineering Reference"** section with a multi-subsection narrative (Affected Services list with role/confidence prose). This is a stray appendix beyond Sections 0–6 and is NOT one of the sanctioned trailers (it is not a Revision Summary, nor a one-line labeled blockquote metadata line). The trailing "## Revision Summary" itself is EXPECTED and is NOT flagged. |
| Section 4 is bullet-list only (no tables/types) | PASS | §4 is a bullet list of business concepts; no attribute tables or type declarations present. |

Note on the Appendix: the base rule explicitly cites "a multi-paragraph narrative KG cross-check analysis appendix with its own prose findings" as the kind of stray content to flag. The "Appendix: Engineering Reference" here is exactly that shape — a narrative, multi-item engineering-reference section appended after Section 6 — so it is flagged. (The §1 "Service Scope Note (KG cross-check)" is in-section and NOT a violation.)

**KG cross-check reconciliation:** The spec's claimed affected services (`household-id-adornment-api` high confidence; `household-id-adornment-proto` medium confidence) **match the independent KG cross-check exactly** — same two services, same roles, same confidence levels. No discrepancy; no contradiction with implementation reality. The spec correctly treats the change as additive to the proto contract.

---

## 6. Recommendations

1. **Resolve the NFR-005 sensitive-value decision before engineering (CG-004 / Risk FAIL).** Do not ship unconditional "echo raw lookup data" (FR-004/FR-010) alongside an unresolved masking decision. Add an FR that gates raw-value echo on the masking determination, covering both the response payload and logs.
2. **Add an FR for the empty/invalid-input no-result case (CG-003 / Edge Case 1).** Define what step-identifier and searched-values contain when nothing valid was searched.
3. **Rewrite FR-003, FR-004, FR-005 Then-clauses to assert on the response payload, not on human cognition (CG-001/CG-002).** Remove "common failure cases" and "Support can determine" phrasings; assert presence/non-emptiness of concrete diagnostic fields.
4. **Remove the "## Appendix: Engineering Reference" section (Format FAIL)** or fold its grounded-context content into Section 1's Service Scope Note / Section 6, so the document stays within Sections 0–6 plus the sanctioned Revision Summary.
5. **Commit a firm latency threshold in NFR-002** or relocate the "adjust during eng-plan" caveat to a decision note, so there is one testable number.
6. **Add requirements for partial diagnostic build success (Edge Case 3), strict-validation consumers (Edge Case 6), payload-size bounds (Edge Case 5), and summary format/localization (Edge Case 8).**