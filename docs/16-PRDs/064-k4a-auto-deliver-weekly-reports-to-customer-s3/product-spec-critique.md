# Spec Critique: Automatic Weekly S3 Report Delivery (Northwind Media)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-19

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **FR-013 (kill-switch) has a header but NO requirement body.** The requirement is entirely empty — no Given/When/Then, no acceptance criteria. Engineering has nothing to implement against, yet the kill-switch is cited as the deadline safety net (Risk Mitigation) and referenced by NFR-014 and the Information Requirements. A load-bearing control is undefined. | Critical | §3 FR-013 | Write the full Gherkin body. See CG-001 rewrite in §2. |
| CG-002 | **No defined behavior for report generation failing or producing an empty/partial artifact.** FR-005 assumes the report "is generated as it does today," then FR-006 zips it and FR-008 delivers it. If generation fails, times out, or yields a zero-byte/partial file, the flow has no branch — the system may zip nothing and deliver a corrupt/empty object to the customer's bucket, silently overwriting a prior good report (FR-008 overwrites same object). This is customer-facing data corruption. | Critical | §2 flow, §3 FR-005/006/008 | Add an FR: if generation fails or yields an empty/invalid artifact, do NOT deliver, do NOT overwrite the prior object, record a distinct failure reason, and alert per FR-011. Validate artifact integrity before upload. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-013 | *(empty — header only)* | Not written at all; untestable by definition. | **Given** S3 delivery is globally disabled via the config kill-switch, **When** the weekly schedule fires for an onboarded customer, **Then** no new delivery job starts, **And** the disabled state takes effect within 5 minutes of the flip, **And** a delivery already in flight completes under prior behavior, **And** the flip requires no code deploy and no separate approval step, **And** the flip is recorded in the audit trail. |
| NFR-002 | "should complete **well within** the weekly batch window... for a **typical** weekly report size... engineering sizing detail to confirm" | "Well within" and "typical size" are not testable; the p95<10min target is stated but the input size is explicitly left open, so the test has no fixed precondition. | **Given** a weekly report of ≤ [DEFINE] MB, **When** delivery (compression + upload) runs, **Then** p95 end-to-end time per customer is < 10 minutes measured over ≥ [N] scheduled runs. Author must fix the size ceiling before this is testable. |
| NFR-001 | "must complete within the batch window... **without delaying** generation or delivery for other customers" | "Without delaying" has no measurable threshold. | **Given** N onboarded customers in one batch run, **When** one customer's delivery is slow or retrying, **Then** no other customer's delivery start time is delayed by more than [DEFINE] seconds (isolation bound). |
| FR-009 / NFR-011 | "exponential backoff" with no base interval or cap stated | Max attempts (3) is testable; the backoff schedule is not — "exponential" without a base/cap yields non-deterministic test timing. | Define: **Then** retries occur at [base]s, [base×2]s, [base×4]s intervals, capped at [max]s, for a maximum of 3 attempts. |

---

## 3. Risk Mitigation Audit

| Risk (from spec §1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------|---------------------|---------|-------|
| PII / customer-data delivery to external destination | Yes — NFR-009 (scoped single destination, encrypted transport), FR-003 (validate before data leaves), FR-012 (audit) | **PASS** | Coherent, but see CG-002: no integrity check means a corrupt artifact can still be delivered to the validated destination. |
| Customer credential management (headline risk) | Yes — NFR-004/005/007/008, FR-002/004 | **PARTIAL PASS** | Storage/logging/masking/scoping/offboarding all covered. BUT: no requirement for behavior when the **secured credential store itself is unreachable** at delivery time (fail-closed is only specified for *invalid* credentials in NFR-010, not for store outage). And BR-006 credential model is still [DECISION REQUIRED], which the spec itself flags as must-resolve-before-build. |
| Multi-service integration / KG disagreement | Acknowledged (§1 note, NFR Risk Mitigation) but mitigation = "confirm in discovery" | **FAIL** | A risk is not mitigated by deferring it to discovery. The affected-service set is genuinely unsettled (confirmed by the independent cross-check — see §4). There is no FR owning the report-config vs. dispatch split, so work can be duplicated or dropped. This is a real planning risk left unmitigated in the spec. |
| Hard 6-week deadline | Partially — relies on resolving BR-006/BR-007 early + kill-switch fallback | **FAIL** | The named fallback (FR-013) is **empty** (CG-001). The deadline's safety net does not exist in the document. Two [DECISION REQUIRED] items (BR-006, BR-007) remain open and are explicitly on the critical path. |
| Audit durability / retention | Yes — NFR-013, FR-012 | **PARTIAL PASS** | Immutability and alert-on-write-failure are covered. Retention is 12 months but marked [DECISION REQUIRED] — untestable until fixed. |

---

## 4. Missed Edge Cases

1. **Report generation failure / empty artifact:** (CG-002) No branch for generation failing, timing out, or producing a zero-byte/partial file before zip. Overwrite semantics (FR-008) mean a bad artifact can destroy the prior good object.
2. **Zip/compression step failure:** FR-006 assumes compression always succeeds. No handling for compression errors or out-of-disk during zip.
3. **Credential store unreachable at delivery time:** NFR-010 fails closed on *invalid* credentials; nothing specifies behavior when the secured credential store (NFR-005) is down and credentials can't be *retrieved* at delivery time. Transient? Permanent? Alert?
4. **Bucket permissions silently narrowed after enablement:** FR-004 revalidates "on credential or permission *change*," but how is a permission change detected if the customer edits their bucket policy without telling K4A? Likely only discovered at delivery time as a permission-denied failure — the revalidation trigger is underspecified.
5. **Bucket exists at onboarding but is deleted/renamed later:** FR-010 names "missing/unreachable bucket" as a failure reason at delivery, but there is no re-validation or customer re-notification loop beyond the per-cycle alert.
6. **S3 object-lock / versioning / existing object conflict:** FR-008 relies on overwrite for idempotency. If the customer's bucket has object-lock or write-once policies, overwrite will fail. No handling defined for a destination whose policy forbids overwrite.
7. **Partial / interrupted upload:** If the upload is cut off mid-transfer, S3 may contain a truncated object (for non-multipart) or an incomplete multipart upload. No requirement to clean up incomplete uploads or verify object integrity post-upload (e.g., ETag/checksum check).
8. **Clock/timezone definition of "report week":** FR-007/FR-008/FR-012 key idempotency and dedup on "customer and report week," but "week" boundary (timezone, week-start) is never defined. Two runs near a week boundary could be treated as different weeks.
9. **Kill-switch flipped mid-cycle with retries pending:** NFR-014 says in-flight deliveries complete under prior behavior. Undefined: does a *queued-but-not-started* retry honor the disable, and does an exhausted-retry failure still alert while the feature is disabled?
10. **Concurrent credential rotation during active delivery:** FR-004 revalidates on rotation; FR-007 prevents overlapping delivery. Undefined what happens if credentials are rotated *while* a delivery for that customer is in flight.
11. **Audit write succeeds but delivery record write fails (or vice versa):** FR-012 handles audit-write failure, but the consistency between the delivery-attempt record (Info Req) and the audit record under partial failure is undefined.
12. **Alert storm / notification throttling:** NFR-003 alerts on sustained failure; FR-011 alerts per failure. For a single customer failing every cycle, no de-duplication/throttling is defined — risk of alert fatigue.
13. **Offboarding during an in-flight delivery:** NFR-008 revokes credentials on offboarding. Undefined what happens if offboarding occurs while a delivery using those credentials is active.
14. **Entitlement source unreachable mid-delivery:** NFR-006 fails closed if entitlement source is unreachable — but this would halt delivery for a legitimately-entitled customer every time the entitlement service blips. Acceptable? No requirement weighs availability vs. strict enforcement here.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | **FAIL** | **FR-013** is a header with no body at all — not Gherkin, not even prose. All other FRs (FR-001–FR-012) are correctly in Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | A trailing **"## Appendix: Engineering Reference"** exists — a multi-paragraph narrative analysis section with its own curated/cross-check service inventory and prose findings/recommendations. This is NOT a sanctioned trailer (not a Revision Summary, not a one-line `> **Label:**` metadata blockquote) and is exactly the "stray KG cross-check appendix with its own prose findings" the format rule targets. It must be flagged and either removed or relocated out of the spec. |
| Section 4 is bullet-list only (no tables/types) | **PASS** | Section 4 is bullet-list only; no attribute tables or type declarations present. |

---

## 6. Recommendations

1. **Write FR-013 (kill-switch) in full.** It is empty and is the named deadline fallback. (CG-001)
2. **Add a report-generation / artifact-integrity failure requirement** so a failed, empty, or partial report is never zipped and never overwrites a prior good object. (CG-002)
3. **Remove or relocate the "Appendix: Engineering Reference"** multi-paragraph service-inventory section — it breaks the Sections 0–6 constraint and injects implementation narrative into a product spec.
4. **Resolve the two [DECISION REQUIRED] blockers on the critical path — BR-006 (credential model) and BR-007 (customer-facing surface) — now**, since the spec itself names them as uncertainty-collapsing and the 6-week deadline has little slack.
5. **Add fail-closed behavior for credential-store and entitlement-source *unavailability*** (distinct from invalid credentials), including whether such outages are transient-retried or permanently failed, and weigh strict fail-closed enforcement against availability for a legitimately-entitled customer.
6. **Define "report week"** (timezone and week-start boundary) since idempotency, dedup, and audit all key on it.
7. **Specify concrete, testable values** for NFR-002 (report-size ceiling) and FR-009/NFR-011 (backoff base/cap), replacing "well within," "typical," and bare "exponential."
8. **Add post-upload integrity verification** (checksum/ETag) and incomplete-multipart cleanup, and define behavior for buckets with object-lock/versioning that forbid overwrite.
9. **Reconcile the affected-service set as a spec-visible task, not just "discovery."** The independent KG cross-check **confirms the disagreement**: the fresh lookup re-found `sam-report-scheduler` (high), `reporting-dispatcher`, `reporting-api`, `reporting-subscriptions-worker`, `sam-performance-reporter`, `k4a-sam-backend` (medium), and `distribute-api`/`sam-mono`/`sam-agent`/`frontend-mos` (low) — but did **NOT** re-surface `credentials-api` or `accounts-api`, the two services the spec leans on for its headline credential-storage (NFR-005) and entitlement-enforcement (NFR-006) requirements. The spec's core security claims rest on two services the independent evidence could not confirm exist in the expected form; flag this credential-store/entitlement-service assumption as explicitly unverified before build.