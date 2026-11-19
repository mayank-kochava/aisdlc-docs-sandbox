# Spec Critique: K4A Conversion Report — Contract-Currency Revenue Display

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
| CG-001 | **Conversion method / FX rate is undefined but is a correctness gate.** BR-5 flags the FX rate source and as-of date as `[DECISION REQUIRED]`, yet BR-2 makes correctness a release gate and FR-005 depends entirely on this. Engineering cannot implement conversion, and QA cannot assert any expected value, until the FX source and as-of date are pinned. This is a dead end for the core feature, not a parallel decision. | Critical | 1 (BR-5), 3 (FR-005) | Resolve BR-5 before build: name the FX rate source, the as-of date rule (report-period vs. contract date), and the handling of weekends/holidays/missing-rate days. Convert BR-5 into a concrete, testable FR with a worked numeric example. |
| CG-002 | **Service ownership is unresolved and internally contradictory.** Section 1 and the Appendix concede two "partly contradictory clusters" (reporting-api/cost-v2/k4a-mcp-server vs. the SAM-family cluster) could own the K4A conversion report. No FR or NFR can be validated against a system that has not been identified. The entire spec rests on an unknown codebase. | Critical | 1 (affected-system note), Appendix | Complete the ownership-reconciliation investigation and name the single authoritative generating service BEFORE this spec is accepted for estimation, not during planning. Until then, every downstream FR (esp. FR-008/FR-012 "cached USD value") is unverifiable. |
| CG-003 | **"Authoritative contract-currency source" is asserted to exist but is unverified.** FR-001/FR-003/BR-4 assume a per-customer authoritative contract-currency configuration exists and is retrievable. The spec itself lists this as a "prerequisite investigation" (Section 6) — i.e., it is NOT yet confirmed to exist. If the field does not exist, FR-002's "no currency resolvable" path becomes the *default* for most customers, silently turning the happy path into mass download failure. | Critical | 3 (FR-001/003), 6 | Confirm the contract-currency field exists and is populated per customer/subscription before build. Define expected behavior and alerting if coverage is partial (e.g., 40% of customers have no configured currency). |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must not *materially* degrade... baseline plus a *small margin*... engineering performance call" | "Materially," "small margin," and an undefined baseline are not testable. No pass/fail threshold exists. | Given a report of representative size under representative concurrent load, When the user requests a download, Then p95 generation time does not exceed [BASELINE_MS] × 1.15. (Pin BASELINE_MS and the multiplier.) |
| NFR-002 | "respect a *rate cap* protecting it so report generation cannot *starve* other consumers" | No numeric cap, no definition of "starve." Untestable. | Given N concurrent report requests, When read load hits the revenue/contract source, Then requests from the report path are capped at [X] req/s and other consumers retain ≥ [Y]% of throughput. |
| NFR-003 | "defined decimal precision and rounding rule" (the rule itself is deferred) | The convention is named but not specified; no test can assert the rounded output. | Given a raw value of 100.005 EUR equivalent, When converted and rounded, Then the file shows exactly [2] decimals using [HALF_UP] rounding → 100.01. (Pin decimals + mode.) |
| FR-006 | "applied in a *defined, consistent order* so that the aggregated total equals the *intended* converted total" | "Intended" is circular; the actual order (convert-then-sum vs sum-then-convert) is not stated here (only NFR-003 hints "convert-then-sum"). | Given 3 revenue records, When the report aggregates, Then each record is converted individually and summed (convert-then-sum), and the file total equals Σ(round(convert(recordᵢ))). State the order explicitly in the FR. |
| FR-010 | "the file *indicates* which currency" | "Indicates" is unspecified (BR-8 defers the extent). Not testable until scope is fixed. | Given the file contains contract-currency revenue, When opened, Then [the currency ISO code appears in the column header for each revenue column]. (Resolve BR-8 to make concrete.) |
| NFR-011 | "New requests pick up the change *within 5 minutes*" | 5 min is testable, but "without separate approval" and "revert to prior behavior" need a verifiable assertion. | Given the kill-switch is flipped off at T, When a new report is requested at T+5min, Then it uses prior (USD) behavior; And in-flight reports started before T complete under the new behavior. (Good — keep, just add an in-flight assertion test.) |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Multi-service blast radius | Partial — NFR-011, NFR-012, ownership-reconciliation prerequisite | FAIL | Mitigations reference a system set that is *itself unresolved* (CG-002). You cannot assert backward compatibility (NFR-012) against downstream consumers you haven't identified. Mitigation is nominal, not actionable. |
| Billing/revenue data handling | Yes — BR-2, NFR-003, NFR-007, NFR-008 | PASS | Fail-closed + audit + rounding are concrete. Weakened only by NFR-003's deferred precision rule. |
| Tight end-of-month deadline | Partial — NFR-011, NFR-014, "resolve decisions early" | FAIL | The spec acknowledges the deadline is at risk but provides no de-scope plan if BR-5/BR-6/CG-002 aren't resolved in time. "Resolve early" is a hope, not a mitigation. Define the minimal shippable slice (e.g., EUR-only, single-currency, no multi-contract). |
| Legacy/shared revenue pipeline dependency | Yes — NFR-002, NFR-010 (bounded retry), NFR-012 | PASS (conditional) | Retry + rate cap are specified. Conditional on NFR-002 getting a numeric cap. |
| FX rate source unavailability | Yes — FR-007, NFR-008, NFR-010 | PASS | Fail-closed + named error + bounded retry cover this well. |
| Partial contract-currency coverage (some customers unconfigured) | **No** | FAIL | Spec treats "no currency resolvable" (FR-002) as an exception, but never estimates how common it is or defines a bulk-failure threshold/alert. If this is the common case, the feature fails silently at scale (see CG-003). |
| Audit retention / compliance | Flagged `[DECISION REQUIRED]`, provisional 12-month floor | FAIL (open) | Retention is a provisional guess pending compliance. A revenue/billing audit artifact with an unconfirmed retention period is a compliance risk, not a resolved mitigation. |

---

## 4. Missed Edge Cases

1. **Stale / missing FX rate for the as-of date:** No rule for weekends, holidays, or a date where the FX source has no published rate. Does it fail (FR-007) or use the nearest prior rate? Undefined — directly undermines BR-2.
2. **Partial contract-currency coverage across a customer base:** FR-002 handles one unresolvable customer; nothing defines behavior or alerting when a large fraction of customers lack a configured currency (ties to CG-003).
3. **Zero / negative / refund revenue values:** Conversion and rounding behavior for credits, refunds, or zero revenue is unspecified. Rounding a negative near .005 interacts with the (deferred) rounding mode.
4. **Currency that equals USD:** If a customer's contract currency *is* USD, is a conversion (rate 1.0) still "applied," or is it a no-op? FR-008 (no double conversion) and FR-005 need to state the identity case.
5. **FX rate = contract currency but historical rate drift:** A report regenerated later for the same period — does it reproduce the identical figure (pinned as-of rate) or drift with a live rate? Reproducibility of a billing artifact is unaddressed.
6. **Kill-switch flipped mid-aggregation:** NFR-011 says in-flight reports complete under the behavior at start, but a multi-service generation (per Appendix, spanning several backends) may read the switch at different times across services. Cross-service switch consistency is undefined.
7. **Double conversion across the two candidate clusters:** FR-008 forbids double conversion, but if BOTH cost-v2 *and* a SAM-family service touch revenue (ownership unresolved), the convert-once guarantee cannot be proven. Edge case is unverifiable until CG-002 is closed.
8. **Concurrent download of the same report during a config flip:** Two users downloading the same customer's report seconds apart could receive different currencies (one pre-flip USD, one post-flip EUR) — no guarantee of consistency within a tight window beyond the 5-minute note.
9. **File-format rendering of currency symbols / locale:** CSV vs. XLSX handling of currency symbols, thousands separators, and decimal separators (EUR locales use comma decimals) is unspecified — a "1.234,56" vs "1,234.56" ambiguity on a revenue file.
10. **Multi-currency report with a currency that has zero revenue:** FR-004 handles mixing, but not a segmented report where one currency segment has no rows — empty-segment labeling is undefined.
11. **Audit write fails while report succeeds:** NFR-007 says a failed audit write raises an alert but report generation proceeds async — so a HIGH-risk billing artifact can be delivered with no audit record. Is that acceptable for a release gate? Policy unstated.
12. **Backward-compat consumer reading the file programmatically:** NFR-012 protects USD consumers, but if any integration *parses the downloaded file itself* (not an upstream API), changing the in-file currency silently breaks it. The spec only considers API/data consumers, not file-parsing consumers.
13. **Retry exhaustion vs. partial success:** NFR-010 allows 3 retries; if attempt 3 to the contract-currency source succeeds but the FX source then fails, which named error surfaces (FR-002 vs FR-007)? Ordering of the two failure checks is undefined.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-012 are all in proper Gherkin blocks. No prose FRs found. |
| Document contains only Sections 0–6 | **FAIL** | The document ends with `## Appendix: Engineering Reference`, a multi-paragraph narrative analysis section (two service clusters with prose role descriptions, a "SAM-family cluster" cross-check narrative, and investigation recommendations). This is a stray narrative/appendix section beyond Sections 0–6 — exactly the "multi-paragraph KG cross-check appendix with its own findings" shape that is NOT exempt. It is not a Revision Summary and not a one-line `> **Label:**` metadata trailer. Move the ownership-reconciliation evidence out of the spec (into an engineering ticket) or collapse it into the Section 1 note. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list of *what-must-be-known* items with Source annotations in prose; no attribute tables or type definitions present. |

---

## 6. Recommendations

1. **(CG-002) Resolve service ownership before acceptance.** Reconcile the reporting-api/cost-v2 cluster vs. the SAM-family cluster and name the single authoritative generating service. Everything else is unverifiable until this is closed — including FR-008/FR-012's "no cached USD / single conversion" guarantees, which cannot hold if two clusters both touch revenue.
2. **(CG-001) Pin the FX rate source and as-of date (BR-5) with a worked example.** Turn BR-5 from `[DECISION REQUIRED]` into a concrete, testable FR before build; specify weekend/holiday/missing-rate handling.
3. **(CG-003) Confirm the authoritative contract-currency field exists and measure coverage.** Define bulk-failure alerting and expected behavior for partially-configured customer bases.
4. **(Format) Remove the `## Appendix: Engineering Reference` section** from the spec so the document contains only Sections 0–6; relocate the KG ownership evidence to an engineering investigation ticket.
5. **(Testability) Replace vague NFR thresholds with numbers:** pin NFR-001 baseline + margin, NFR-002 rate cap, and NFR-003 precision/rounding mode with a worked example.
6. **Define the convert-then-sum order explicitly in FR-006** (not just by reference in NFR-003) and add zero/negative/refund and USD-identity cases.
7. **Resolve the deferred decisions (BR-6, BR-7, BR-8, NFR-007 retention, NFR-012 consumers)** or explicitly define the minimal Phase-1 slice (EUR-only, single-currency-per-report) so the end-of-month deadline has a de-scope fallback.
8. **Add edge-case FRs** for stale FX rates, locale-specific number formatting in the file, cross-service kill-switch consistency, and audit-write-failure policy on this HIGH-risk artifact.