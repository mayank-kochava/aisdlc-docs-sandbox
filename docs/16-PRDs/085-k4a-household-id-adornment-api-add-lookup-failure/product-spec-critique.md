# Spec Critique: Household ID Lookup Failure Diagnostics

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
| CG-001 | FR-008/NFR-003 state diagnostics degrade gracefully on a *build error*, but no requirement covers a **partial** diagnostic build — e.g. the step identifier resolves but searched-value extraction throws. The flow chart has only "built" vs "error building," no partial state. It is undefined whether partial diagnostics are emitted or suppressed wholesale. | High | 2, 3C, 5 | Add an FR: on partial diagnostic assembly failure, either emit the successfully-built subset with an explicit "partial" marker, or omit all diagnostics — pick one and make it testable. NFR-003 says "must not leak partial or malformed diagnostic data," which implies all-or-nothing, but no FR enforces it. |
| CG-002 | "No result" is treated as a single branch, but the lookup can fail for two distinct reasons: (a) resolution ran and matched nothing, vs (b) resolution could not run at all (malformed input, upstream dependency error inside the lookup). The spec never distinguishes "genuine no-match" from "lookup errored out." FR-003's "which step failed" is meaningless if the lookup never reached any step. | High | 1, 3A | Define whether an internal lookup *error* (not a clean no-match) is in scope for diagnostics, and if so what the step identifier reports for a pre-step failure. |
| CG-003 | FR-005 promises Support can determine cause "for the common failure cases" but "common failure cases" is never enumerated, and no FR defines what happens for uncommon cases beyond the implicit escalation branch (flow node K). There is no acceptance boundary for FR-005. | High | 3A | Enumerate the common failure cases the diagnostics are guaranteed to cover, or reframe FR-005 as non-testable context rather than a requirement (see Testability). |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-005 | "...the agent can determine why the lookup returned nothing without escalating to engineering **for the common failure cases**" | "Common failure cases" is unbounded and "the agent can determine" is a human-judgment outcome, not a system behavior. Cannot be automated. | `Given a no-result response for each enumerated failure case {step-A-miss, step-B-miss, input-normalization-miss}, When Support reads the diagnostic block, Then the response contains a non-null failed-step identifier AND the searched-values set for that case.` (Test the presence/shape of data; drop the human-cognition clause.) |
| FR-003 | "Then Support **can identify** which specific step/stage..." | Tests a human capability, not an API output. | `Given a no-result response, When the response is inspected, Then the diagnostic information contains exactly one failed-step identifier drawn from the service's enumerated step set.` |
| FR-004 | "Then Support **can see** the input values..." | Same human-capability framing. | `Given a no-result response, When the response is inspected, Then the diagnostic information contains the searched input values and each corresponds to a field present in the original request.` |
| NFR-002 | "must not add **material latency**" | "Material" is subjective; the +10ms p95 guardrail is given but labeled a "working default... adjust during eng-plan," so no committed threshold exists. | `Given the no-result path under representative concurrent load, When diagnostics are generated, Then added p95 overhead ≤ 10ms vs. the pre-change baseline.` (Commit the number or mark NFR-002 explicitly deferred, not both.) |
| Value section | "**shortening** the time to resolve" / "Support can **quickly** determine" | "Quickly"/"shortening" not measurable. Correctly moved to Section 6 as a future-phase target, but still appears as live prose in Section 1. | Flag as context only; no acceptance criterion should depend on it. |

Note: all nine FRs are structurally in Gherkin. The testability failures above are semantic (human-capability `Then` clauses and an uncommitted threshold), not format violations.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Proto/contract change ripples to unknown consumers | Yes — NFR-001, FR-006, FR-007 | PASS | Mitigation is concrete and has a verification condition (old-contract consumer operates unchanged). |
| Lookup may not have discrete nameable steps | Yes — FR-003 guarded, fallback to searched-values only | PASS (conditional) | Mitigation is a *deferral* ("confirm in engineering"), which is acceptable given [DECISION REQUIRED], but the fallback behavior (steps-not-discrete → searched-values only) is **not captured in any FR**. The mitigation lives only in prose. Add an FR variant for the single-stage case. |
| Diagnostic generation failure degrades core lookup | Yes — NFR-002/003, FR-008 | PASS | Well covered and testable via forced-failure injection. |
| Over-broad diagnostic payload / sensitive values | Partial — NFR-005, BR-4 | FAIL | NFR-005 states diagnostics "must not introduce raw credentials/secrets/tokens" but the actual exposure risk here is **customer address/name/identifier values** echoed into the response and logs. Masking is punted to a [DECISION REQUIRED] content call with no requirement, no default-safe behavior, and no FR. "Must not broaden exposure beyond what the lookup already handled" is unverifiable without knowing the lookup's current handling. This is the only genuine risk gap and should be closed before eng commits to echoing raw searched values. |

---

## 4. Missed Edge Cases

1. **Partial diagnostic build:** Step identifier succeeds but searched-value capture fails (or vice versa). Undefined — see CG-001.
2. **Lookup internal error vs. clean no-match:** No distinction between "matched nothing" and "couldn't run." See CG-002.
3. **Empty / malformed request input:** If the request itself has no searchable values, FR-004's "searched values" set is empty — is an empty diagnostic echo valid, or is that itself a diagnosable condition? Not specified.
4. **Multiple steps fail / ambiguous failure point:** FR-003 assumes exactly one identifiable failed step. If resolution fails across several stages, which step is reported? First, last, or all? Information Requirements say "which discrete step" (singular) — unaddressed.
5. **Diagnostics size / unbounded searched-value payload:** No cap on the size of echoed searched values. A request with a very large input set could bloat the no-result response. No limit specified.
6. **Concurrency / idempotency of diagnostic generation:** NFR-002 mentions "representative concurrent load" but no FR covers whether diagnostic generation is stateless/thread-safe. Likely fine, but unstated.
7. **Old-contract consumer encountering the new optional field on a wire format that is position-sensitive (e.g. non-tag-based):** FR-007 assumes tolerant deserialization. For a strict/positional contract this assumption can break. The proto path is only medium-confidence; the deserialization-tolerance claim should be verified against the actual contract encoding, not assumed.
8. **Masked-value correlation loss:** If NFR-005 masking is later applied, masked searched-values may no longer let Support "correlate with customer-reported input" (FR-004's stated purpose). The masking decision and FR-004 are in tension; neither references the other.
9. **Caching / intermediary layers:** If any CDN, gateway, or response cache sits in front of the API, a cached no-result response could carry stale diagnostics for a different input. No caching posture stated.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-009 all use valid Gherkin blocks. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present. The trailing "Appendix: Engineering Reference" is a KG-derived engineering-reference section with service bullets — it is a short grounded-context block, not a multi-paragraph narrative findings/recommendations appendix. It borders the "stray KG appendix" concern; recommend relocating its content into Section 1's existing Service Scope Note to be unambiguous, but it is not treated as a blocking format failure. No Revision Summary or metadata-blockquote trailer present. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list of business concepts; no attribute tables, no types. The "e.g. the address/name/identifier values" is illustrative prose, not a schema. |

---

## KG Cross-Check Findings

The independent KG lookup re-confirmed exactly the two services the spec claims: `household-id-adornment-api` (high confidence, owns lookup) and `household-id-adornment-proto` (medium confidence, contract change). **No discrepancy** between the Context Seed's claimed services and the fresh lookup — the spec's Service Scope Note (Section 1) and Appendix correctly represent the grounded reality, including the medium-confidence caveat on the proto. No service was claimed by the spec that the fresh lookup failed to re-find, and no additional service surfaced. The spec's implementation-reality claims (single-service, additive contract change, no cross-service fan-out, no new datastores) are consistent with the evidence. One caveat: the spec asserts prior-contract consumers "continue to deserialize without error" (FR-007/NFR-001) as a design guarantee — the evidence does not confirm the proto's encoding is tag-based/tolerant, so that guarantee should be verified against the actual proto rather than asserted (see Missed Edge Case #7).

---

## 6. Recommendations

1. **Close the sensitive-value handling gap (NFR-005 / FR-004).** Define a default-safe behavior for echoing customer identifier/address values into the response and logs — mask-by-default until the BR-4 content decision lands, rather than leaving raw echo as the implicit default. This is the one real risk gap.
2. **Resolve the no-match vs. lookup-error distinction (CG-002)** and define what the failed-step identifier reports when resolution never reached a step.
3. **Specify partial-diagnostic behavior (CG-001)** — all-or-nothing vs. marked-partial — to back NFR-003's "no partial/malformed data" claim with a testable FR.
4. **Make FR-005 testable** by enumerating the guaranteed-covered "common failure cases" or demoting it to context.
5. **Add an FR for the single-stage fallback** currently living only in the FR-003 risk prose, so the "no discrete steps" path has acceptance criteria.
6. **Commit or explicitly defer the NFR-002 +10ms threshold** — don't do both.
7. **Verify the proto's deserialization tolerance** against the actual contract encoding before relying on FR-007's additive-optional guarantee; address multi-step/empty-input and payload-size edge cases (#3, #4, #5).
8. **Consider relocating the Engineering Reference appendix** content into the Section 1 Service Scope Note to keep the document strictly within Sections 0–6.