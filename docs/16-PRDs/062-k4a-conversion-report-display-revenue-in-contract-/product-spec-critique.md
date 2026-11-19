# Spec Critique: K4A Conversion Report — Contract-Currency Display for EU Customers

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-19

## Summary

- **Critical Issues:** 3
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The single most important behavioral decision — the **failure posture** (fail-closed/withhold the report vs. degrade to prior USD behavior) — is left entirely as [DECISION REQUIRED] (FR-006, FR-010, NFR-007). This is not a detail; it is the core branch of the user flow for billing/revenue data under a HIGH risk tier. Engineering literally cannot implement the `FailPosture` node in the Section 2 flowchart, which means the error paths (the majority of the diagram) are unbuildable. | Critical | 1 (BR-5), 3 (FR-006/010), 5 (NFR-007) | Force a business decision before engineering starts. Given HIGH risk tier + billing data, the safe default is fail-closed (withhold), but the spec must state ONE posture, not defer it, because every error-path FR and the rollback depend on it. |
| CG-002 | The **scope-population tension** (single Lumen Retail account vs. all EU customers) is flagged [DECISION REQUIRED] but describes two materially different products with different blast radius, different data models, and different feasibility against a hard external renewal deadline. The spec picks a "buildable default" but acknowledges the deadline is "at risk until confirmed" (NFR-009). Starting engineering against an unconfirmed scope boundary on a deadline-bearing contract is a planning-level data-loss-equivalent risk (wrong build shipped). | Critical | 1 (scope tension), 5 (NFR-009) | Escalate the population boundary as a gating decision. Do not let engineering begin core data-model work until the boundary (one account vs. EU population) is confirmed. |
| CG-003 | **Which codebase owns this logic is unresolved** — the spec itself admits two "different, largely non-overlapping" candidate service sets (Set A generic names vs. Set B `sam-*` cluster) that appear to describe the same capability under two naming schemes. You cannot write an eng-plan, estimate against a deadline, or define a kill-switch's blast radius when the owning service is unknown. The independent KG cross-check provided for this review re-finds ONLY the Set B `sam-*` cluster and does NOT re-find any Set A service (reporting-api, cost-revenue-workers, analytics-query, accounts-api, reporting-dispatcher). This is corroborating evidence that Set A may be fabricated/stale. | Critical | 1 (service-ownership note), Appendix | Resolve service ownership before engineering. The fresh KG lookup only confirms `sam-*` services; Set A is unverified and should be treated as suspect, not co-equal, until a human confirms. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must not materially degrade K4A conversion-report generation latency... within the current baseline plus an agreed margin" | "Materially," "agreed margin," and "current baseline" are all undefined. Untestable — no concrete threshold. | `Given a representative K4A report-generation load, When contract-currency resolution and conversion are enabled, Then p95 report-generation time is ≤ [baseline_ms] + [margin_ms], where both values are fixed before the test is written.` |
| FR-013 | "the report presents the original amount, the converted amount, and/or conversion metadata according to the defined report-content rule" | The "and/or" plus an undefined "report-content rule" makes this untestable — there is no single assertable output. | `Given an eligible EU customer whose revenue was converted, When the downloaded report is produced, Then the report contains [exactly: converted amount only | both original and converted | converted + metadata], per the resolved BR-6 decision.` Must pick one. |
| FR-014 | "the presented contract-currency figures are consistent with the prior generation for that period" | "Consistent" is undefined given exchange rates change over time — does "consistent" mean identical (same rate frozen) or recomputed at current rate? This is a correctness landmine. | `Given a report previously generated for period P using rate R, When the report is regenerated for the same period P, Then the figures are recomputed using the SAME effective rate R (frozen at first generation), producing byte-identical revenue values.` (Assuming frozen-rate semantics — must be decided.) |
| FR-005 | "resolves the customer's contract currency from the defined authoritative source" | The source is [DECISION REQUIRED], so the central happy-path FR cannot be tested — there is no known source to stub/assert against. | Not rewritable until BR-4 is resolved. Flag as blocked on BR-4. |
| NFR-011 / NFR-013 | "state explicitly whether reports already in-flight at flip time complete under the prior behavior" [DECISION REQUIRED] | The kill-switch / rollback in-flight behavior is explicitly undefined, yet it is the HIGH-risk rollback contract. Untestable rollback = unsafe rollback. | `Given a report generation in-flight when the kill-switch is flipped, When the flip propagates, Then the in-flight report [completes under prior behavior | is aborted and regenerated under prior behavior] — pick one.` |

Note: The FR prose bodies ARE in Gherkin and are generally well-formed. The testability failures above are concentrated in the NFRs (prose) and in FRs whose assertions dissolve into [DECISION REQUIRED] placeholders.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard renewal deadline (end of next month) | Partial (NFR-009) | **FAIL** | NFR-009 does not mitigate the deadline; it correctly states the deadline is "at risk until" two unresolved [DECISION REQUIRED] items are answered. Flagging a risk as unmitigated-pending-decision is honest but is NOT a mitigation. This is a real, open blocker. |
| Billing/revenue data correctness (HIGH tier) | Yes (NFR-006, NFR-010, FR-009, FR-010) | PASS | Reproducibility, no double-conversion, no mislabeled USD figure, append-only trace. Well-covered — the strongest part of the spec. |
| Multi-service / high blast radius (6+ services) | Yes (NFR-011 kill-switch) | PASS (with caveat) | Kill-switch + no-deploy revert is a real mitigation. BUT its blast radius is unknowable while service ownership (CG-003) is unresolved — a kill-switch across the wrong service set does not mitigate anything. Downgrade to conditional PASS. |
| Backward compatibility for existing consumers | Yes (NFR-012, FR-003) | PASS | Byte-for-byte unchanged for non-EU; eligibility gate required before any consumer sees a different currency. Specific and testable. |
| Contract-currency source unknown (BR-4) | No (deferred) | **FAIL** | BR-4/FR-005 acknowledge the authoritative source is unknown but define no requirement to handle the unknown — the entire capability's data foundation is a [DECISION REQUIRED]. |
| Exchange-rate mechanism may not exist (FR-008) | No (deferred) | **FAIL** | FR-008 acknowledges "whether a conversion mechanism / rate source already exists or must be built" is undecided. A HIGH-risk conversion feature with an unknown rate source is an unmitigated feasibility risk against the deadline. |
| Secure report delivery standard (NFR-005) | Partial | FAIL | NFR-005 states the report "must be delivered through the existing secure path" but admits it is "not stated" whether that path satisfies a time-limited secure-link standard. Assumed-but-unverified security control. |
| Conversion-trace retention (NFR-006) | Partial | FAIL | Append-only is specified (good), but minimum retention is [DECISION REQUIRED]. For HIGH-risk billing audit data, "we don't know how long we keep it" is an unmitigated audit/compliance gap. |

---

## 4. Missed Edge Cases

1. **Rounding and precision on conversion:** No FR defines rounding rules (half-up, banker's rounding, decimal places) when converting USD→EUR. Two reports summing line-items vs. converting a total can disagree by cents — a reconciliation defect in a feature whose entire purpose is reconciliation accuracy. Critical for billing data.
2. **Sum-vs-convert order of operations:** Related to above — does the system convert each line item then sum, or sum in USD then convert? These produce different totals. Undefined.
3. **Zero / negative revenue (refunds, credits, chargebacks):** No FR addresses negative or zero revenue figures. Conversion and formatting of negative amounts in locale conventions (e.g., `-1.234,56 €` vs `(1.234,56 €)`) is unspecified.
4. **Mid-period currency change:** What happens if Lumen Retail's contract currency changes (or the account's EU status changes) partway through a reporting period? FR-014's "consistency" doesn't cover a currency that was different earlier in the period.
5. **EU status changes between report request and generation:** FR-004 re-derives eligibility "on that request," but a long-running async report (reporting-dispatcher / sam-report-scheduler) could have eligibility flip between enqueue and generation. Which wins is undefined.
6. **Stale exchange rate / rate freshness boundary:** FR-008 defers rate refresh frequency. There is no FR defining the behavior when a rate exists but is stale beyond an acceptable age — "rate available" is treated as binary, but a 6-month-old rate on billing data is a correctness risk distinct from "rate-unavailable."
7. **Currency pair where original ≠ USD:** FR-008 says "original currency," FR-003/FR-009 assume USD is the baseline. If a revenue figure's original currency is neither USD nor the contract currency (e.g., GBP revenue, EUR contract), is a rate pair guaranteed? Triangulation (GBP→USD→EUR) is unaddressed.
8. **Partial conversion failure within one report:** FR-010 treats rate-unavailable as a report-level failure, but what if ONE currency pair in a multi-line report is unavailable while others succeed? Does the whole report fail, or is it partial? Undefined — a partial-success hole in the HIGH-risk path.
9. **Kill-switch flip mid-report (in-flight):** Explicitly flagged [DECISION REQUIRED] in NFR-011/NFR-013 but listed here because it is an unhandled edge case, not just a decision — the rollback contract is incomplete without it.
10. **Multi-account report where accounts span EU and non-EU:** FR-007 covers multi-currency EU accounts, but a parent org with both EU and non-EU accounts in one report (mixed eligibility within a single file) is not addressed.
11. **Downstream reader breakage (sam-agent):** The KG cross-check explicitly flags `sam-agent` as a downstream reader of SAM revenue fields that "needs currency-contract verification." No FR or NFR addresses downstream consumers that read revenue fields and assume USD semantics — NFR-012 covers report *consumers* but not service-level field readers like sam-agent. This is a silent-corruption risk.
12. **Locale vs. currency mismatch:** FR-012 formats "according to the contract currency's locale conventions," but currency (EUR) and locale (de-DE vs. fr-FR vs. ga-IE) are not the same thing. EUR is used across many EU locales with different formatting. Which locale drives formatting is undefined.
13. **Report generated for a brand-new EU customer with no revenue history:** Empty-report / no-data case under the new currency path is unspecified.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-014 are all in well-formed `gherkin` blocks. NFRs are intentionally prose per the template (NFRs are not FRs), so they are not counted here. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"## Appendix: Engineering Reference"** after Section 6. This is a multi-paragraph narrative appendix with its own two candidate-service lists and prose findings (Set A / Set B service ownership analysis) — exactly the kind of stray KG cross-check / analysis appendix that is NOT one of the sanctioned trailer exceptions (it is not a Revision Summary, and it is not a short one-line `> **Label:**` metadata blockquote). It belongs in a separate eng-plan artifact, not the product spec. Also note the two stray orphan lines "Now reporting the decisions I flagged in Sections 0–3." and "Now reporting the decisions I flagged in Sections 4–6." appear mid-document as leftover generation artifacts and should be removed. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 (Information Requirements) is bullet-list only. Each item uses "Source:" and "Matters because" prose with [DECISION REQUIRED] annotations — no attribute tables and no data types (types are correctly deferred to eng-plan). Compliant. |

---

## 6. Recommendations

1. **Resolve the failure posture (CG-001) before any engineering begins.** Pick ONE posture for both named error states (fail-closed/withhold strongly recommended for HIGH-risk billing data). Every error-path FR, NFR-007, and the rollback depend on this single decision.
2. **Escalate the scope-population boundary (CG-002) as a hard gate.** Confirm single-account vs. EU-population before data-model work starts; the renewal deadline cannot be honestly estimated until this is fixed.
3. **Resolve service ownership (CG-003).** The fresh KG cross-check re-finds ONLY the `sam-*` (Set B) cluster and finds NONE of Set A's services — treat Set A as suspect/stale, confirm the real owning services with a human, then scope the kill-switch blast radius against the confirmed set.
4. **Define conversion arithmetic precisely:** rounding rules, decimal precision, and sum-vs-convert order of operations (edge cases 1, 2). These are correctness defects waiting to happen in a reconciliation feature.
5. **Add a downstream-reader requirement for sam-agent and any service reading revenue fields** (edge case 11) so a currency-semantics change cannot silently corrupt a downstream consumer that assumes USD.
6. **Decide frozen-rate vs. recomputed-rate for regeneration (FR-014)** and rewrite the FR with a testable, deterministic assertion.
7. **Pin down NFR latency numbers, conversion-trace retention, and secure-delivery standard** so the NFRs become testable rather than "agreed margin" / "not stated" placeholders.
8. **Cover partial-failure, stale-rate, negative-revenue, mixed-eligibility, and locale-vs-currency edge cases** (edge cases 3, 6, 8, 10, 12) with explicit FRs.
9. **Remove the Engineering Reference appendix** from the product spec (move to eng-plan) and delete the two orphan "Now reporting the decisions…" artifact lines to restore Section 0–6 compliance.