# Spec Critique: Automatic Weekly S3 Report Delivery (Northwind Media)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-19

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **FR-013 (Kill-switch) has a title but NO body.** The FR header `**FR-013 — Kill-switch to disable S3 delivery**` is followed immediately by Section 4 — there is no Gherkin, no acceptance criteria, nothing. This is a referenced requirement (NFR-014, Information Requirements, Risk Mitigation all point to FR-013) with zero defined behavior. An engineer cannot build it and QA cannot test it. | Critical | §3 FR-013 | Add a full Gherkin body defining: who can flip the switch, scope (global vs per-customer), what happens to queued vs in-flight deliveries, and how the disabled state is surfaced. NFR-014 gives raw material but the FR itself must exist. |
| CG-002 | **Report-generation failure (upstream of delivery) is undefined.** Every flow and FR begins with "a report has been generated." The spec never defines what happens when the existing weekly generation step itself fails or produces an empty/corrupt report. Does delivery deliver a zero-byte zip? Is that a success? The flowchart's `Generate` node has no failure edge. | Critical | §2 flow, §3 FR-005/FR-006 | Add an FR covering generation failure / empty report: delivery must not fire, the condition must be recorded distinctly from a *delivery* failure, and support must be alerted. Distinguish "nothing to deliver" from "delivery failed." |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "should complete **well within** the weekly batch window... **typical** weekly report size... engineering sizing detail to confirm" | "Well within," "typical," and an unconfirmed size ceiling are not testable. A p95 < 10 min target has no denominator (10 min for *what size*?). | Given a weekly report of size ≤ [X] MB uncompressed, When delivery (compression + upload) runs, Then p95 end-to-end time measured over scheduled runs is < 10 minutes, and no single delivery exceeds [hard ceiling] minutes. **[X] must be set, not deferred.** |
| NFR-001 | "complete within the batch window... **without delaying** generation or delivery for other customers" | "Without delaying" is not measurable without a defined window boundary or isolation criterion. | Given N onboarded customers in one weekly run, When customer A's delivery runs, Then customer A's delivery failure or slowness adds no more than [X] to any other customer's completion time, and the run completes within the [defined] batch window. |
| NFR-014 | "New scheduled runs honor the disabled state **within 5 minutes** of the flip" | Testable number, but "a delivery already in flight completes under the prior behavior" is untestable without a defined max in-flight duration — an in-flight delivery could run arbitrarily long after the flip. | Given the kill-switch is flipped to disabled, When a delivery is in flight, Then it completes under prior behavior but no new attempt (including retries) starts, and no in-flight delivery persists beyond [max delivery duration] after the flip. |
| FR-013 | *(no text at all)* | Cannot test an empty requirement. | See CG-001. |

---

## 3. Risk Mitigation Audit

| Risk (from spec §1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| PII / customer-data delivery | Yes | PASS | NFR-009 (scoped single destination, encrypted transport) + FR-003 pre-validation + FR-012 audit. Adequately mitigated. |
| Customer credential management | Partial | **FAIL** | Storage/logging/masking/scoping/offboarding all covered (NFR-004/005/007/008). **But the core BR-006 credential-model decision is unresolved**, and NFR-008 says credentials are "revoked/removed" on offboarding without defining revocation semantics for a *long-lived IAM key* (K4A cannot revoke the customer's own IAM key — it can only delete its stored copy). The mitigation is incomplete until BR-006 resolves. |
| Multi-service integration | Partial | **FAIL** | "Mitigated when the affected-service inventory is confirmed in engineering discovery" — this defers the mitigation rather than defining it. The two KG lookups still disagree (see §4/CG). No requirement forces the config/execution split (report-config vs dispatcher) to be disambiguated before build; it is only aspirationally noted. |
| Hard 6-week deadline | Partial | **FAIL** | Relies on BR-006 and BR-007 being "resolved early" — but both are still `[DECISION REQUIRED]` in this very document. The mitigation is contingent on open decisions the spec does not close. Kill-switch fallback (FR-013) is cited as mitigation but FR-013 has no body (CG-001). |
| Audit-write failure | Yes | PASS | FR-012 + NFR-013 raise an operational alert rather than failing silently. Well handled. |
| Entitlement bypass | Yes | PASS | NFR-006 server-side enforcement, fail-closed. Good. |

---

## 4. Missed Edge Cases

1. **Empty / zero-row report:** If a given week produces no attribution data, is an empty zip delivered and recorded as success? Undefined (also CG-002).
2. **Report larger than expected / S3 single-PUT limit:** No multipart-upload behavior defined for a report exceeding S3's 5 GB single-PUT ceiling. NFR-002's size assumption is deferred, so the upper bound is unknown.
3. **Partial / interrupted upload:** If the upload fails mid-stream, does a truncated object land in the customer bucket? FR-008 idempotent-overwrite assumes a clean overwrite, but a partial object followed by exhausted retries could leave corrupt data in the customer's bucket with no cleanup requirement.
4. **Customer revokes access mid-flight:** Permissions valid at FR-003 validation but revoked before the weekly run. Covered by FR-010 permanent-failure path conceptually, but the *detection timing* (stale validation state) is not specified — nothing forces re-validation at delivery time.
5. **Bucket lifecycle / object-lock conflict:** If the customer's bucket has versioning or object-lock enabled, the idempotent "overwrite same object" in FR-008/NFR-012 may create a new version or be rejected. Not addressed.
6. **Clock skew / "same report week" definition:** FR-007 and FR-008 key on "same customer and report week," but "report week" is never defined (timezone? ISO week? week-ending date?). Idempotency and overlap-prevention correctness depend on this definition.
7. **Multiple weekly runs missed during an outage:** If the kill-switch (or a failure) keeps delivery off across several cycles, is there backfill, or are missed weeks silently skipped? No catch-up behavior defined.
8. **Credential rotation race:** FR-004 revalidates on rotation, but if rotation occurs *while a delivery is in flight* (FR-007), which credential set does the in-flight attempt use? Undefined.
9. **Validation passes but delivery path differs:** FR-003 validates write to "the configured bucket and path," but FR-008 writes a *zipped* object possibly at a different key pattern. Validation should write-test the *actual* delivery key, not just the prefix, to avoid a validation-passes/delivery-fails gap.
10. **Audit-write failure during kill-switch:** NFR-013 alerts on audit-write failure, but if the system is already in a degraded/killed state, the alerting path's own availability is unaddressed.
11. **Offboarding of a long-lived IAM key:** NFR-008 "revoke/remove" — K4A can only delete its stored copy; it cannot revoke the customer's IAM key. No requirement to notify the customer to deactivate the key on their side. Residual access risk.
12. **Concurrent onboarding re-submission:** If support re-submits credentials while a validation (FR-003) is already running, overlap behavior is undefined (FR-007 only covers delivery overlap, not validation overlap).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | **FAIL** | **FR-013 has no body at all** — not prose, not Gherkin, nothing (CG-001). All other FRs (FR-001 through FR-012) are correctly in Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | Document contains an `## Appendix: Engineering Reference` after Section 6 — a multi-paragraph narrative "Affected Services" analysis with its own curated/cross-check findings and planning recommendations. This is exactly the stray multi-paragraph KG-analysis appendix the format rule is meant to catch (it is NOT a Revision Summary and NOT a one-line `> **Label:**` metadata trailer). Additionally, §1 contains a "### KG cross-check note" subsection; this is borderline but acceptable as context within Section 1. The trailing Appendix is the violation. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 contains only bullet-list items with prose descriptions and inline Source/FR references; no attribute tables or type definitions. Compliant. |

**KG cross-check (evidence vs. spec claims):** The spec's Appendix honestly reports that two lookups disagreed. My independent fresh lookup **confirms the discrepancy and sharpens it**: the fresh lookup did **not** re-surface `credentials-api` or `accounts-api` at all (both claimed by the spec's first lookup as the credential-store and entitlement services). Since the entire credential-security risk tier (NFR-004/005/007/008) and entitlement enforcement (NFR-006) implicitly rely on a credentials service and an accounts/entitlement service existing, the fact that the independent lookup cannot confirm either is a **material grounding gap**: the spec asserts a "secured credentials-handling path" (NFR-005) and an "authoritative account/entitlement source" (NFR-006, Information Requirements) as if they exist, but the independent evidence does not confirm the services that would provide them. This should be flagged as a scope/grounding risk in §1, not resolved by picking one lookup. (Correctly, the spec already treats service scope as discovery-required — but it should explicitly note that the *credential store itself* is unconfirmed, since that is the headline risk.)

---

## 6. Recommendations

1. **Write the FR-013 body (CG-001).** A referenced, load-bearing kill-switch FR cannot be empty. This is a Critical format + logic failure blocking engineering.
2. **Define report-generation / empty-report failure handling (CG-002).** Add an FR distinguishing "nothing to deliver" and "generation failed" from delivery failures.
3. **Remove or relocate the Engineering Reference Appendix** to satisfy format compliance — fold the confirmed service-scope-is-discovery-required point into Section 1's existing KG cross-check note; the appendix's narrative analysis cannot remain as a post-Section-6 section.
4. **Resolve BR-006 (credential model) before build.** It gates NFR-008 revocation semantics, FR-002 storage, and every credential edge case. Clarify that for long-lived IAM keys, K4A revocation means deleting its stored copy only, and add a customer-side deactivation step.
5. **Set NFR-002's report-size assumption to a concrete number** and add multipart-upload behavior for large/partial uploads (edge cases 2 & 3), including cleanup of truncated objects.
6. **Define "report week" precisely** (timezone/week boundary) — FR-007 overlap prevention and FR-008 idempotency correctness depend on it.
7. **Require delivery-time (not just onboarding-time) revalidation or stale-state detection** so a mid-period permission revocation is caught before an attempt (edge cases 4 & 9).
8. **Confirm the credentials and accounts/entitlement services exist** in engineering discovery, since the independent KG lookup could not re-find either; flag this explicitly as a grounding risk for the headline credential risk tier.