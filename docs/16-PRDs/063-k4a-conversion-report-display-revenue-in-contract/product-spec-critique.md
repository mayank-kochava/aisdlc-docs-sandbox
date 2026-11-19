# Spec Critique: K4A Conversion Report — Contract-Currency Revenue Display

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
| CG-001 | The entire conversion-accuracy requirement (FR-006, BR-4, NFR-009) rests on "currency-resolution inputs" (conversion rate or pre-converted source) whose nature, source, and timing are explicitly undefined and deferred to engineering. "Revenue reflects contract terms accurately" is undefined until it is known whether the figure is a FX-converted USD amount (which drifts daily and will NOT match a contract-billed figure) or a natively contract-currency amount (which would). The spec's own Value proposition — "match what the customer is billed" — is unachievable with a market-rate conversion, yet the spec permits that path. This is a correctness/data-integrity gap on a billing document. | Critical | 1 (BR-4), 3 (FR-006/007), 4 (Currency-resolution inputs), 5 (NFR-009) | Decide and state whether contract-currency revenue is sourced natively or FX-converted. If FX-converted, define which rate, as-of what date, from what source, and add an FR asserting the converted figure equals the contracted/billed figure within a stated tolerance — otherwise the stated Value is not met. |
| CG-002 | Scope boundary ("download-only, no UI change", FR-010) directly contradicts the independent KG cross-check, which surfaces `frontend-mos` (console UI displaying conversion report revenue) and `sam-report-scheduler` (scheduled/batch report runs). If a console view or scheduled-report path exists, a file-only change silently produces an on-screen or scheduled report whose revenue currency disagrees with the download — a split-brain correctness defect on a revenue document. The spec acknowledges this but defers it as an "engineering reconciliation item" rather than blocking on it, while simultaneously asserting FR-010 as a confirmed requirement. | Critical | 1 (Service-map uncertainty), 3 (FR-010), 6, Appendix | Resolve whether the conversion report has an on-screen console view (`frontend-mos`) and a scheduled path (`sam-report-scheduler`) BEFORE committing FR-010. If either exists, FR-010's "download-only" claim is false and the scope/blast-radius must be rewritten. Do not treat as parallelizable. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-006 | "reflects the contract currency accurately and consistently with the customer's contract terms" | "Accurately" is not measurable without a defined source-of-truth figure and tolerance. No test can assert this. | Given a report whose display currency is a non-USD contract currency, When revenue values are resolved, Then each displayed value equals the corresponding contract-billed figure from the authoritative source within a tolerance of ±0.01 of the smallest currency unit. |
| NFR-001 | "must not materially degrade... within the current baseline plus an acceptable margin" | "Material", "acceptable margin", and baseline are all undefined (spec admits [DECISION REQUIRED]). Not testable. | Given a representative report set of N generations, When generated with contract-currency resolution, Then p95 generation time must not exceed [baseline]ms + [X]% / +[Y]ms. (Fill numbers — currently untestable.) |
| FR-009 | "unambiguously identified as being in the actual display currency used" | "Unambiguously" is subjective; exact label location/format deferred to BR-5 [DECISION REQUIRED]. | Given a downloaded report with non-USD revenue, When opened, Then the revenue column header (or [named metadata field]) displays the ISO-4217 code of the display currency, and no field displays "USD" for a non-USD report. |
| NFR-002 | "bounded and, where feasible, resolved once per report" | "Where feasible" is an escape hatch that makes the requirement unenforceable; "bounded" has no value. | Given report generation, When contract currency is retrieved, Then the authoritative source is queried exactly once per report and the call completes within [X]ms or triggers NFR-004. |
| FR-013 / NFR-005 | "without a code deploy... within 5 minutes" | The 5-minute propagation is testable; "without a separate approval or escalation step" is a process claim, not a system behavior — untestable as written. | Given the kill-switch config is flipped to OFF, When a new report is generated ≥5 minutes later, Then revenue falls back to USD behavior. (Keep. Drop the approval-process clause or move it to runbook.) |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Multi-service blast radius (HIGH) | Partial — kill-switch, staged rollout, fail-closed defined | FAIL | Blast radius cannot be trusted while the service-map is unresolved (two conflicting KG sets). The mitigations are generic; none addresses the specific risk that the wrong service set gets instrumented. The staged-rollout percentages (NFR-006) assume a known deployment surface that does not yet exist. |
| Billing/revenue data handling (HIGH) | Yes — NFR-009/010/011, FR-011 | PASS (conditional) | Well-covered for labeling/consistency, BUT the underlying accuracy definition is unresolved (see CG-001). Fail-closed is sound; the accuracy bar it protects is undefined. |
| Tight end-of-month deadline (HIGH) | Acknowledged, scope-controlled | FAIL | Mitigation is "the business must decide" — this is a deferral, not a mitigation. Two hard blockers (customer identity, service-map) are unresolved and the spec admits the deadline cannot be validated. Listing a risk and saying someone else will handle it is not a mitigation. |
| Legacy/unknown contract-currency source (`accounts-api`, medium confidence, "presumed source of truth") | Partial — NFR-004 retries, NFR-003 fail-closed | FAIL | BR-3 asserts an "authoritative customer-contract record" but the source is only *presumed* to be `accounts-api` (medium confidence) and unconfirmed. No requirement validates that this source actually holds contract currency per customer. Fallback is "fail closed," which is safe but means an unconfirmed source could fail-close EVERY non-USD report. |
| Downstream USD-dependent consumers break silently | Conditional — FR-012/NFR-008 | PASS (conditional) | Requirement correctly forbids silent breakage, but the inventory of such consumers does not exist yet. The requirement is enforceable only after the open reconciliation item is closed. |
| Scheduled/batch report path (`sam-report-scheduler`, cross-check) | No | FAIL | No FR or NFR addresses whether scheduled report generations pick up the currency change or the kill-switch flip. NFR-005 only covers "new report generations" generically. |

---

## 4. Missed Edge Cases

1. **FX conversion as-of date ambiguity:** If revenue is FX-converted, no requirement defines the rate date (report-run date, billing-period date, contract-signing date). Different choices yield different figures for the same revenue — directly undermines BR-4.
2. **Contract currency changes mid-period:** A customer's contract currency changing (renewal, renegotiation) between the revenue period and the report-generation time is undefined. Which currency applies to historical revenue in the report?
3. **Scheduled/batch reports (`sam-report-scheduler`):** Does a kill-switch flip (NFR-005) affect an in-flight scheduled batch? Reports generated by a scheduler rather than an interactive download are not addressed by the "new report generations" language.
4. **On-screen console view (`frontend-mos`):** If revenue is also shown in a console, the file and the screen will disagree during/after rollout. Not covered (see CG-002).
5. **Partial report scope spanning a currency change:** FR-005 covers *multiple concurrent contracts* but not a *single contract whose currency changed over the report's time range*.
6. **Zero-revenue vs. failed-generation distinction:** FR-011 requires these be distinguishable, but no requirement defines a legitimately zero-revenue non-USD report (e.g., a valid €0.00 report) — risk of a real zero being treated as a failure or vice versa.
7. **Rounding / minor-unit handling per currency:** Currencies with different minor-unit conventions (e.g., JPY has 0 decimals) are not addressed. FR-008 consistency could break on rounding alone. BR-7 promises "any non-USD currency" generality, which makes this a real exposure.
8. **Totals computed in USD then displayed in contract currency:** FR-008 requires internal consistency, but if subtotals are summed in USD and converted, versus converting each row then summing, totals may not reconcile due to rounding. No requirement pins the computation order.
9. **Kill-switch flip mid-generation:** A flag flipped while a report is being generated — does the report use USD or contract currency? No atomicity requirement.
10. **Caller requests a report for a customer that doesn't exist / identity unresolvable:** FR-001 assumes the report-subject customer is reliably derivable; the failure path when customer identity itself is ambiguous (vs. currency ambiguous) is not defined.
11. **Contract currency is a deprecated/invalid ISO code:** No validation requirement that the retrieved contract currency is a real, supported currency before it's used as a display label.
12. **Download retry after a fail-closed generation:** If generation fails closed (FR-011), is the failure cached, or does every retry re-hit `accounts-api`? Interacts with NFR-002/NFR-004 but is unspecified.
13. **Multiple KG service sets implementing the same report differently:** If both the cost-pipeline set AND the SAM set generate a conversion report, the change may need to land in both or produce two divergent reports. Not addressed as a behavioral requirement.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-013 are all in Gherkin code blocks. Explanatory `>` notes accompany some FRs but the requirement bodies themselves are Gherkin. |
| Document contains only Sections 0–6 | FAIL | The document contains an **"Appendix: Engineering Reference"** after Section 6. This is a multi-paragraph narrative analysis section (Affected Services, two service sets, a Reconciliation note with prose findings) — exactly the kind of stray analytical appendix the format rule is written to catch. It is NOT a sanctioned Revision Summary and NOT a short one-line `> **Label:**` metadata trailer, so it is flagged. (Note: content after the "END OF DOCUMENT UNDER REVIEW" line is excluded from this check per the document's own marker.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list describing *what* information and *why*, with Source annotations in prose. No attribute tables, no field types, no schema. Compliant. |

---

## 6. Recommendations

1. **Resolve the conversion-accuracy definition (CG-001) before engineering.** State whether revenue is natively contract-currency or FX-converted; if converted, define rate, source, and as-of date, and add a testable equality-to-billed-figure requirement. Without this, the feature's stated Value is unverifiable.
2. **Close the scope/blast-radius contradiction (CG-002) before committing FR-010.** Confirm or rule out the console view (`frontend-mos`) and scheduled path (`sam-report-scheduler`). The "download-only" claim and the staged-rollout plan both depend on this.
3. **Reconcile the two conflicting KG service sets** and confirm `accounts-api` (currently only *presumed*, medium confidence) actually holds per-customer contract currency. The multi-service risk mitigation cannot be trusted until this is done.
4. **Make FR-006, NFR-001, NFR-002, and FR-009 testable** — supply the missing numbers (tolerance, baseline, margin, label location) and remove escape-hatch phrasing ("where feasible").
5. **Add explicit requirements for scheduled/batch report generation** (kill-switch applicability, in-flight behavior) and for the on-screen console view if it exists.
6. **Add edge-case FRs** for mid-period contract-currency changes, per-currency minor-unit/rounding handling (especially given BR-7's "any currency" promise), totals computation order, and kill-switch atomicity mid-generation.
7. **Move the Engineering Reference appendix** out of the product spec (or into eng-plan) to restore Section 0–6 compliance, or fold its open items into Section 1 where the service-map uncertainty is already recorded.
8. **Treat the deadline risk honestly** — it currently has no mitigation, only a deferral. Surface the two hard blockers (customer identity, service-map) as explicit go/no-go gates.