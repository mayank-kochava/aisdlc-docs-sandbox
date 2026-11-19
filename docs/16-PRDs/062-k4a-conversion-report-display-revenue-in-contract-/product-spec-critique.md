# Spec Critique: K4A Conversion Report — Contract-Currency Display for EU Customers

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-19

## Summary

- **Critical Issues:** 2
- **High Issues:** 7
- **Medium Issues:** 6
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Service ownership is unresolved and the two candidate service sets are nearly non-overlapping.** The spec openly admits (Section 1 service-ownership note, NFR-009, Appendix) that two KG lookups returned largely disjoint service sets (`reporting-api`/`cost-revenue-workers`/… vs. the `sam-*` cluster) overlapping only on `frontend-mos`. The independent KG cross-check confirms *only* the `sam-*` set. Engineering cannot start against an undeclared codebase, and this directly threatens the hard renewal deadline. This is not merely an "open question" — it blocks implementation entirely. | Critical | 1 / Appendix / NFR-009 | Resolve codebase ownership BEFORE engineering kickoff. Run a definitive ownership trace (not two conflicting KG lookups) and name the single authoritative service set in the spec. Note: the independent cross-check re-found *only* `sam-*` services — Set A ("reporting-api", "cost-revenue-workers", "accounts-api", etc.) was NOT re-found and may be phantom/stale. See also FC-nothing; this is a reality discrepancy (see §3/§4). |
| CG-002 | **The exchange-rate conversion mechanism's very existence is undefined, yet conversion is core to the feature.** FR-008 and the Information Requirements both carry a [DECISION REQUIRED] on whether a rate source/mechanism *exists or must be built*, plus rate source and refresh frequency. If it must be built, that is potentially the single largest piece of work in the spec, and it is completely unscoped against a hard deadline. A feature whose central transformation is "TBD whether it exists" cannot be implemented or estimated. | Critical | 3 (FR-008) / 4 | Perform discovery NOW to confirm whether a conversion/rate mechanism exists. If it must be built, scope it explicitly or descope conversion from Phase 1 (e.g. Lumen Retail revenue natively in EUR → FR-009 path only, no rate dependency). |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must not materially degrade… p95… within the current baseline plus an agreed margin" | "Materially," "agreed margin," and baseline are all unspecified → untestable. [DECISION REQUIRED] acknowledged but leaves no acceptance threshold. | Given a representative report-generation load, When a K4A report is generated for an eligible EU customer, Then p95 generation time is ≤ [baseline_ms] + [agreed_margin_ms]. *(Author must supply both numbers.)* |
| FR-014 | "the presented contract-currency figures are consistent with the prior generation" | "Consistent" is ambiguous — bit-for-bit? Within rounding? Does a changed exchange rate between generations count as inconsistent? Not testable as written. | Given a report previously generated for period P with rate R, When regenerated for the same period P, Then each contract-currency figure equals the prior figure to [n] decimal places, using the rate effective-dated to P (not the current rate). *(Author must define rate effective-dating — see EC below.)* |
| FR-012 | "formatted according to the contract currency's locale conventions" | "Locale conventions" is unspecified (decimal separator, symbol placement, thousands grouping). Not a testable assertion. | Given an eligible EU customer with contract currency EUR, When the report is produced, Then amounts render as "1.234,56 €" per the de-DE/EUR formatting rule [specify the exact locale rule]. |
| NFR-002 | "apply an explicit rate cap protecting the shared resource" | No cap value given. "Explicit rate cap" without a number is not verifiable. | Given currency-resolution/rate-lookup traffic, When call rate exceeds [N] req/s against the shared resource, Then calls are throttled/rejected per the cap. *(Author must supply N.)* |
| NFR-006 | "Kochava's typical range is 12–24 months. [DECISION REQUIRED]" | Retention is a range, not a value → the audit/retention requirement cannot be verified. | Given a conversion trace record, When [retention_months] has elapsed, Then it is handled per the defined retention policy. *(Author must pick a single value.)* |

*Note: all Section 3 FRs are correctly written in Gherkin — no prose FRs found (see §5).*

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard renewal deadline (end of next month) | Partial (NFR-009) | **FAIL** | NFR-009 explicitly states the deadline is "at risk until" scope-population AND service-ownership are confirmed — both still [DECISION REQUIRED]/unresolved (CG-001). Acknowledging a risk and stating it is unmitigated is not a mitigation. |
| Billing/revenue data correctness (HIGH tier) | Yes (NFR-010, NFR-006, FR-009, FR-010/011) | **PASS** | Reproducibility, no double-conversion, no mislabeled USD — adequately mitigated, pending the retention value. |
| Multi-service / high blast radius | Yes (NFR-011 kill-switch) | **PASS (with gap)** | Kill-switch + on-call authority defined. BUT in-flight-report behavior at flip time is still [DECISION REQUIRED] — mitigation is incomplete until that is stated. |
| Legacy/external rate dependency | Partial (NFR-008 retry) | **FAIL** | NFR-008 defines retry/backoff, but the dependency itself may not exist (CG-002). You cannot define resilience for a dependency you haven't confirmed exists or chosen. Rate source + refresh cadence unresolved. |
| Service-ownership ambiguity | Acknowledged only (Section 1, NFR-009) | **FAIL** | Flagged, never mitigated. No requirement or action item forces resolution before build. This is the root blocker (CG-001). |
| Backward compatibility for USD consumers | Yes (NFR-012, FR-003) | **PASS** | Byte-for-byte non-EU unchanged + eligibility gate on downstream consumers is a concrete contract. |
| Sensitive data in logs | Yes (NFR-004) | **PASS** | Explicit prohibition on raw revenue in logs/error paths. |

**Reality-discrepancy flags (spec claims vs. independent KG evidence):**
- The spec's Set A (`reporting-api`, `cost-revenue-workers`, `analytics-query`, `accounts-api`, `reporting-dispatcher`) was **NOT re-found** by the independent cross-check, which returned only the `sam-*` cluster + `frontend-mos`. The spec presents both sets as "grounded fact"; the independent evidence supports only one. The spec's claim that `accounts-api` is "most likely home of the contract-currency answer" is unsupported by the independent lookup — no `accounts-api` appears in it. **Flag: do not treat Set A as grounded.**
- BR-4 / FR-005 assert the authoritative contract currency lives in "the stored currency field on the customer's account (billing settings)." No service in the independent KG evidence is confirmed to own a billing-settings currency field (`sam-agent` is flagged as needing *currency-contract verification*, i.e. unverified). **The authoritative-source claim is asserted but not corroborated by the evidence** — treat the location of the contract-currency field as unconfirmed, not settled.

---

## 4. Missed Edge Cases

1. **Exchange-rate effective-dating vs. report period.** FR-008/FR-014 never state whether conversion uses the rate as-of the report period or the rate at generation time. This directly determines whether regeneration (FR-014) is reproducible. Critical ambiguity for billing data.
2. **Rounding and summation drift.** When many per-line USD→EUR conversions are summed, round-then-sum vs. sum-then-round produce different totals. No rule defined. For billing/revenue this can cause reconciliation disputes — the exact problem the feature aims to fix.
3. **Currency precision mismatch.** USD/EUR are 2-decimal, but the spec never constrains currency to 2-decimal currencies. If a future EU-adjacent currency with 0 or 3 decimals is in scope, formatting/conversion breaks silently.
4. **Customer flips EU ↔ non-EU, or contract currency changes mid-period.** FR-004 re-derives eligibility per request, but there is no rule for a report spanning a period during which the customer's EU status or contract currency changed. Which currency applies to the historical portion?
5. **Fallback-flag leakage / customer confusion.** FR-006/FR-010 fall back to USD and "flag the fallback on the report." No spec of what the flag says, where it appears, or whether a customer-facing report should expose an internal failure state. A Lumen Retail renewal report that silently reverts to USD *with a flag they don't understand* undermines the retention goal.
6. **Partial conversion within one report.** If some currency pairs have a rate and others don't (FR-010), is the whole report forced to USD, or is it a mixed report (some EUR, some USD)? The flow diagram sends any rate failure to full USD fallback, but FR-010 is scoped per-figure. Contradiction between Section 2 flow and FR-010.
7. **Zero / negative / refund / credit revenue figures.** No handling for negative amounts, credits, or zero revenue through the conversion path. Sign and formatting of negative EUR amounts undefined.
8. **In-flight reports at kill-switch flip.** Called out as [DECISION REQUIRED] in NFR-011/NFR-013 but never resolved — a report mid-generation when the kill-switch fires has undefined currency behavior.
9. **Multi-account resolution default may ship incorrectly.** FR-007 lets engineering "build against per-account resolution" while the real rule is [DECISION REQUIRED]. If the decision later lands on primary-currency roll-up, shipped behavior for any multi-account EU customer is wrong — a latent correctness bug behind a kill-switch.
10. **Downstream reader `sam-agent` breaking on currency-semantic change.** The independent KG flags `sam-agent` as reading SAM revenue fields with an unverified currency contract. If revenue-field currency semantics change, this consumer can misinterpret EUR as USD. NFR-012 covers "report consumers" generically but names no concrete downstream reader; this specific one is unaddressed.
11. **Locale/encoding of the EUR symbol in the downloaded file.** CSV/file encoding (UTF-8 vs. Latin-1) for "€" is unspecified; a mis-encoded currency symbol in a finance file is a real reconciliation hazard.
12. **Two-column layout (FR-013) breaking existing CSV consumers.** Adding a second (USD) column to the downloaded report changes the file's column structure for EU customers. Any automated downstream consumer parsing by column index breaks. NFR-012 guarantees *non-EU* byte-for-byte, but says nothing about EU-customer consumers whose parsers predate the new column.
13. **Timeout/partial response from the authoritative currency source.** NFR-008 covers retry/backoff, but a *slow-but-successful* response that blows the NFR-001 latency budget has no defined behavior (degrade vs. wait).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-014 are all in valid Gherkin blocks; NFRs are legitimately prose (not FRs). No prose FRs found. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present. Trailing "Appendix: Engineering Reference" and "Revision Summary" — the Revision Summary is the sanctioned spec-revise trailer (not flagged). **However**, the "Appendix: Engineering Reference" is a multi-paragraph narrative section (two candidate service sets with prose role descriptions) — this is the kind of stray analysis appendix normally flagged. It is scoped as explicitly "not required reading for product review" and is engineering-reference grounding derived from the KG; given it duplicates the Section 1 service-ownership note as reference material rather than introducing new product requirements, it is treated as reference matter and not a blocking format failure. Author should relocate it out of the spec body to fully comply. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list; no attribute tables, no data types (types correctly deferred to eng-plan). |

---

## 6. Recommendations

1. **(Critical — CG-001) Resolve codebase ownership before kickoff.** Run a single authoritative ownership trace. The independent cross-check supports only the `sam-*` cluster + `frontend-mos`; do not plan against Set A until re-confirmed. The deadline cannot be assessed until this is settled.
2. **(Critical — CG-002) Confirm whether the exchange-rate conversion mechanism exists.** If it must be built, either scope it explicitly or descope conversion from Phase 1 (if Lumen Retail revenue is natively EUR, the FR-009 no-conversion path may remove the rate dependency entirely and de-risk the deadline).
3. **Define exchange-rate effective-dating and rounding rules** (Edge cases 1, 2, 3) — these are correctness requirements for billing data, not nice-to-haves, and FR-014 reproducibility depends on them.
4. **Resolve the per-figure vs. whole-report fallback contradiction** between the Section 2 flow diagram and FR-010 (Edge case 6).
5. **Resolve all open [DECISION REQUIRED] items that affect shipped behavior**: multi-account resolution rule (FR-007/BR-7), in-flight-report-at-flip behavior (NFR-011/013), on-screen-vs-download surface (Section 1).
6. **Supply concrete values** for NFR-001 (baseline + margin), NFR-002 (rate cap), NFR-006 (retention), and define FR-012 locale formatting precisely — currently untestable.
7. **Specify the customer-facing fallback flag** content/placement (Edge case 5) and the file encoding + two-column impact on downstream EU consumers (Edge cases 11, 12).
8. **Address the `sam-agent` downstream currency-contract risk** explicitly (Edge case 10), tying NFR-012 to the named reader.
9. **Relocate the "Appendix: Engineering Reference"** out of the spec body into the eng-plan to keep the spec to Sections 0–6 plus the sanctioned trailer.