# Spec Critique: Northwind Media — Automatic Weekly S3 Report Delivery

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-19

## Summary

- **Critical Issues:** 1
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **FR-013 (Kill-switch) has a title but NO Gherkin body.** The requirement is named in Section 3 and referenced by NFR-014 and Information Requirements, but its actual behavior (Given/When/Then) is entirely absent. Engineering cannot implement a requirement with no acceptance criteria. This is a hard blocker. | Critical | §3 FR-013 | Supply a full Gherkin body: `Given S3 delivery is globally enabled` / `When an on-call engineer flips the kill-switch to disabled` / `Then no new scheduled delivery jobs start` / `And an in-flight delivery completes under prior behavior` / `And the state change is recorded in the audit trail`. |
| CG-002 | **No defined behavior for a report that fails to GENERATE (vs. fails to deliver).** The flow assumes `Generate → Zip → Deliver` always produces a file. There is no FR for "report generation itself fails or produces an empty/zero-byte artifact." The delivery path would zip and ship a corrupt/empty object, or fail with no classified reason. | High | §2, §3 FR-005/FR-006 | Add an FR covering generation failure and empty-artifact detection before delivery, with a distinct failure reason. |
| CG-003 | **Compression failure is undefined.** BR-003/FR-006 mandate zipping, but no scenario covers the zip step failing (disk, memory, library error). Flow goes straight from Zip to Deliver with no failure branch for Zip. | High | §3 FR-006 | Add a failure branch: if compression fails, record a distinct "packaging failed" reason, do not deliver, alert per FR-011. |
| CG-004 | **Onboarding validation (FR-003) has no timeout / hang handling.** If the customer's bucket is unreachable during validation, the flow loops `Validate → SetupFail → Onboard` but never bounds how long validation waits or what happens on an indefinite hang. | Medium | §2, §3 FR-003 | Define a validation timeout and its failure classification. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "should complete **well within** the weekly batch window... **typical** weekly report size... engineering sizing detail to confirm" | "Well within" and "typical" are not testable; the size assumption is explicitly deferred, so the p95<10min target has no defined input. | `Given a weekly report of size <= [DEFINED_CEILING] MB` / `When delivery (compression + upload) runs` / `Then p95 end-to-end delivery time per customer is under 10 minutes measured over scheduled runs`. Define DEFINED_CEILING. |
| NFR-001 | "complete within the batch window... **without delaying** generation or delivery for other customers" | "Without delaying" is unquantified — no measurable threshold for acceptable impact on other customers. | `Given N customers share the weekly batch window` / `When S3 delivery runs for one customer` / `Then the batch completion time for other customers increases by no more than [X]%`. |
| FR-009 / NFR-011 | "exponential backoff, up to a maximum of 3 attempts" | Backoff **base interval and multiplier are undefined** — "exponential backoff" alone is not testable. | Specify initial interval and multiplier, e.g. `Then the system retries after 30s, 60s, 120s` so retry timing is assertable. |
| NFR-014 | "honor the disabled state **within 5 minutes**" | 5 minutes is testable, but "a delivery already in flight completes under the prior behavior" needs a bound — what if in-flight delivery itself hangs? | Add: `And an in-flight delivery that does not complete within [its own timeout] is force-terminated and recorded as failed`. |
| NFR-003 | "delivery failure rate exceeds 1% of attempts for 5 minutes, sustained" | For a **weekly batch** feature with a single Phase-1 customer, "1% of attempts over 5 minutes" is nonsensical — attempt volume per week is tiny. Threshold is untestable against actual cadence. | Rewrite the alert threshold in terms of per-cycle failures for the batch cadence, not a per-minute rate. |

*Note:* FR-001 through FR-012 are otherwise correctly expressed in Gherkin.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| PII / customer-data delivery to external destination | Yes — NFR-009 (scoped, encrypted), FR-003 (validate before data leaves), FR-012 (audit) | PASS | Coherent chain. |
| Customer credential management (headline risk) | Yes — NFR-004/005/007/008, FR-002/004 | PASS *with gap* | BR-006 credential-model decision is still open; several mitigations (what is stored, rotation) cannot be finalized until resolved. Mitigation framework exists but is decision-blocked. |
| Multi-service integration / disagreeing KG lookups | Partial — "confirm affected-service inventory in discovery" | **FAIL (as a requirement)** | This is a *planning action*, not a functional requirement. No FR disambiguates the reporting-dispatcher vs. reporting-subscriptions-worker vs. scheduler delivery ownership. Risk is acknowledged but not mitigated by any testable behavior. |
| Hard 6-week deadline | Partial — relies on resolving BR-006/BR-007 "early" and kill-switch fallback | FAIL (as a requirement) | A deadline is a project risk, not mitigable by an FR. Acceptable to note, but three [DECISION REQUIRED] items (BR-001, BR-006, BR-007) plus NFR-013 remain unresolved — these are the actual schedule risk and are not yet closed. |
| Orphaned credentials after offboarding | Yes — NFR-008 | PASS | But no FR operationalizes offboarding as a flow (see Edge Cases). |

---

## 4. Missed Edge Cases

1. **Report generation produces an empty / zero-byte report:** No FR detects or blocks delivery of an empty artifact. Customer's pipeline ingests garbage silently.
2. **Compression step failure:** Covered in CG-003 — no failure branch for the mandatory zip step.
3. **Credentials expire *between* validation and the weekly run:** FR-004 covers rotation "when credentials are rotated," but silent server-side expiry of a long-lived key (if BR-006 chooses access keys) between cycles is not explicitly handled — it would surface only as an FR-010 permanent failure at delivery time, with no proactive re-validation.
4. **Bucket deleted or region changed by customer after onboarding:** "Missing/unreachable bucket" is a named failure reason (FR-010), but there is no re-validation trigger comparable to FR-004 for destination (not credential) changes.
5. **Partial / interrupted upload:** FR-008 covers overwrite idempotency, but a multipart upload interrupted mid-stream could leave a partial object. No requirement mandates atomic/complete-or-nothing write to the customer bucket.
6. **Customer bucket is full / quota exceeded / object-lock or retention policy on bucket rejects overwrite:** FR-008 assumes overwrite always succeeds. S3 Object Lock or a deny-on-overwrite bucket policy would break idempotency. Unclassified failure.
7. **Offboarding flow not defined as a sequence:** NFR-008 asserts credentials are revoked on offboarding, but no FR describes *who* triggers offboarding, *when* delivery stops, or the ordering (stop deliveries → revoke credentials → confirm). Race condition risk: a scheduled run firing during offboarding.
8. **Kill-switch flipped mid-cycle for a customer whose delivery already failed:** Interaction between FR-013/NFR-014 and in-flight retry (FR-009) is undefined — does a pending retry honor the kill-switch?
9. **Audit-write failure during a successful delivery:** FR-012 says an audit-write failure raises an alert, but does the *delivery* still count as success if its audit record was never written? Traceability gap.
10. **Clock skew / report "week" boundary ambiguity:** FR-007/FR-008 key idempotency on "customer and report week." The definition of "report week" (timezone, boundary) is never specified — two runs near a week boundary could be treated as different weeks and double-deliver.
11. **Entitlement source unreachable at delivery time (not just config time):** NFR-006 says the check fails closed if the entitlement source is unreachable — but this means a transient entitlement-service outage would *silently skip the weekly delivery* for an otherwise-valid customer. Is a skipped-due-to-fail-closed delivery alerted? Not specified.
12. **Concurrent credential rotation during an active delivery:** FR-004 revalidation and FR-008 delivery could race — a rotation mid-delivery could invalidate the credentials the in-flight job is using.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | **FAIL** | **FR-013** has a title only and **no Gherkin body at all** (see CG-001). All other FRs (FR-001–FR-012) are correctly in Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"Appendix: Engineering Reference"** after Section 6 — a multi-paragraph narrative section with its own curated/cross-check service analysis and recommendations. This is exactly the kind of stray narrative appendix the format rule is written to catch (it is NOT a Revision Summary and NOT a short one-line labeled metadata trailer). Additionally, the inline **"### KG cross-check note"** sub-block in Section 1 embeds discovery-analysis prose into the requirements. Recommend moving both out of the review document. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet-list only; no attribute tables or type declarations present. |

---

## KG Cross-Check Finding (grounding against independent lookup)

The spec's own Appendix concedes the two lookups disagreed, which is honest. Checking the spec's claims against the independent fresh KG lookup:

- The spec's first-lookup services **`credentials-api`** and **`accounts-api`** did **NOT re-surface** in the independent cross-check at all. The spec builds its entire credential-security mitigation chain (NFR-005, NFR-007) and entitlement enforcement (NFR-006) around the *existence* of a dedicated credentials service and an authoritative entitlement/accounts service. **The fresh lookup found no such services.** This is a material discrepancy: the spec's "secured credentials-handling path" (FR-002/NFR-005) may not map to an existing `credentials-api` as assumed — it should be flagged as an unverified assumption, not treated as grounded reality.
- The independent lookup ranks **`sam-report-scheduler`** as the only high-confidence service and surfaces **`reporting-subscriptions-worker`**, **`reporting-dispatcher`**, and **`reporting-api`** at only *medium* confidence with overlapping delivery roles. The spec treats `reporting-dispatcher` as "most load-bearing" — this is not supported by the independent evidence, which does not single it out above the subscriptions-worker or scheduler. The config/execution split (FR-005 delivery attachment) is genuinely unsettled; the spec correctly flags this but should not assert a load-bearing owner.

This contradiction between the spec's claimed implementation reality (dedicated credentials/accounts services) and the independent evidence is the strongest reason the "Multi-service integration" risk must be resolved in discovery before build.

---

## 6. Recommendations

1. **Write the FR-013 Gherkin body (CG-001).** This is the single Critical blocker — the kill-switch is referenced everywhere but has no acceptance criteria. Cannot go to engineering as-is.
2. **Add FRs for generation failure, empty-artifact detection, and compression failure (CG-002, CG-003).** The entire `Generate → Zip` half of the flow currently has no failure handling.
3. **Resolve the credential-service assumption against the KG reality.** The independent lookup did not find `credentials-api` or `accounts-api`. Confirm the secured-credential-store and entitlement-source actually exist before committing NFR-005/006/007 as implementable.
4. **Close the three open [DECISION REQUIRED] items (BR-001, BR-006, BR-007) and NFR-013 retention.** Per the spec's own deadline-risk analysis, these collapse the most uncertainty and are the real schedule risk.
5. **Make NFR-002, NFR-001, NFR-003, and FR-009 backoff testable** — define the report-size ceiling, the acceptable other-customer delay, a cadence-appropriate alert threshold, and concrete backoff intervals.
6. **Define "report week" precisely** (timezone + boundary) since idempotency (FR-007/FR-008) keys on it.
7. **Add an offboarding flow FR** operationalizing NFR-008 (ordering: stop deliveries → revoke → confirm), and define whether a fail-closed skipped delivery (NFR-006) is alerted.
8. **Move the Appendix and the inline "KG cross-check note" out of the review document** to restore Sections 0–6 format compliance.