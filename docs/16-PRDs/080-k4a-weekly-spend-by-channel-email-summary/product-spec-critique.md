# Spec Critique

---
id: product-spec-critique
title: Product Specification Critique
---

# Spec Critique: K4A — Weekly Spend-by-Channel Email Summary

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Unresolved data-source / report-generation path.** §1 engineering grounding caveat and the Appendix admit two KG lookups disagree on which services own the authoritative spend data source and report-generation path. FR-004/FR-005/NFR-011 all depend on "the authoritative spend data source," but the spec never names it or defines how disagreement is resolved. Entire figure-correctness guarantee rests on an undefined source. Given a hard 5-week deadline, this is a build-blocker. | Critical | §1, FR-004, FR-005, Appendix | Resolve service ownership before build. Spec must state which service is authoritative (reporting-api/cost-v2 vs. the SAM stack / sam-performance-reporter) and gate build-start on that decision. See Risk Mitigation Audit RMA-001. |
| CG-002 | **DEC-008 (FR-011a first-send CC to VP) is unresolved AND the VP address is unprovided.** FR-011a is marked `[DECISION REQUIRED: DEC-008]` and the VP email is "`[DECISION REQUIRED]` not yet provided." A requirement that sends customer-facing **restricted financial data** to an unspecified recipient, under an undecided rule, cannot be implemented. If left ambiguous, it risks leaking spend figures to an unintended/unconfirmed address. | Critical | FR-011a, §4 | Either resolve DEC-008 and supply the VP address via confirmed config, or move FR-011a wholly Out of Scope for MVP. Do not ship an unresolved restricted-data recipient. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "must complete with a p95 latency target measured as a defined number of concurrent summary runs... engineering sets the concrete p95 target and concurrency count during planning." | No number = not testable. "Defined number" is a placeholder, not a value. | Given N concurrent weekly summary runs (N defined), When per-channel totals are retrieved for a reporting window, Then p95 retrieval latency ≤ [X ms] and the shared source's rate cap is never exceeded. (Fill X and N before build.) |
| FR-016 / NFR-013 | "new scheduled runs stop starting within 5 minutes" | "within 5 minutes" is testable, but "a run already in flight... completes under the prior behavior" has no defined max in-flight duration, so the overall settle time is unbounded. | Given a run is in flight when the kill-switch flips, When that run continues, Then it completes within [max run duration] and no run started after the flip begins; full quiescence is reached within [X] minutes. |
| FR-009a | "the logo renders correctly in the recipients' email clients" | "renders correctly" across unspecified clients is not objectively testable. | Given an HTML summary email, When rendered in [named target clients], Then the logo displays with alt text and the two-column layout below is unbroken at [defined widths]. |
| FR-001 | "fires on Tuesday morning" / "any day other than Tuesday morning" | "morning" is not a precise boundary; the trigger day is defined but the window edges are vague relative to NFR-001's 09:00 ET. | Given the scheduler, When the current time is Tuesday 09:00 US Eastern (DST-aware), Then a run starts; and for any other time Then no run starts. |

*Note: Most FRs are well-formed Gherkin; the above are the testability gaps worth fixing.*

---

## 3. Risk Mitigation Audit

| Risk (from spec §1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------|---------------------|---------|-------|
| Hard external deadline (5 weeks) | Partial | **FAIL** | Mitigation is conditional ("only if the contested data-source path is resolved before build") but no requirement forces that resolution or defines a fallback if it slips. The deadline risk is restated, not mitigated. See CG-001. |
| Unresolved data-source / report-generation path | No | **FAIL** | Acknowledged as a risk but no fallback requirement, no decision owner, no deadline for resolution. A named risk with no binding mitigation requirement. |
| Multi-service blast radius | Yes | PASS | Kill-switch (NFR-013), per-dependency failure handling (NFR-010), monitoring (NFR-003). Adequately mitigated. |
| Billing/spend data handling | Yes | PASS | NFR-011, FR-006, NFR-006, NFR-004, NFR-012. Block-on-incomplete and correctness enforcement present. |
| DEC-008 unresolved recipient (restricted data) | No | **FAIL** | FR-011a sends restricted spend to an unconfirmed VP address under an undecided rule; no mitigation beyond "config." See CG-002. |
| Retention period for audit records | Working assumption only | Partial | NFR-012 uses an unconfirmed 12-month assumption. Flag for confirmation, not blocking. |

---

## 4. Missed Edge Cases

1. **DST boundary double/zero send:** NFR-001 says the delivery window observes DST transitions, but no FR defines behavior for the spring-forward (09:00 ET skipped) or fall-back (09:00 ET occurs twice) Tuesday. Could the run fire twice or be skipped? FR-002/FR-003 idempotency keys on (customer + window), which likely protects it, but this is not stated for the DST case.
2. **Partial-channel data:** FR-006 blocks on "incomplete" data for the window, but what counts as "incomplete" when 3 of 4 channels return and one channel's source is silent (not errored)? FR-010 treats genuine zero-spend as zero — how is "missing channel data" distinguished from "zero spend"? This is the exact distinction FR-010 claims to protect but the detection rule is undefined.
3. **Reporting window with no data at all / brand-new customer:** If the customer had zero activity across all four channels for the whole window, is an all-zeros email sent, or suppressed? Not defined.
4. **Scheduler misses the Tuesday run entirely (outage during 09:00 ET):** NFR-003 alerts if no delivered email by end of window, but no FR defines catch-up/late-send behavior. Is a missed Tuesday ever sent late, or skipped to next week?
5. **Email delivery "accepted" ≠ "delivered":** FR-012/FR-013 record success on delivery being "accepted for delivery." A provider accept followed by a hard bounce (invalid distro list) is not covered. No bounce-handling requirement.
6. **Distribution-list expansion:** BR-5 sends to a single distro list. If the list is empty, invalid, or the mail provider rejects it, behavior is undefined (vs. individual-recipient bounce).
7. **Spend figures update between retrieval and send:** NFR-016 uses "latest available aggregate at run time." If retried (FR-003/FR-007) and the underlying aggregate changed between attempts, idempotency says "same end state" — but which snapshot wins? Potential conflict between NFR-016 (freshest) and FR-003 (same end state).
8. **Currency/locale assumptions:** BR-2 hardcodes USD to two decimals. No requirement for what happens if the source returns non-USD or null spend for a channel.
9. **Negative/credit spend:** Refunds or credits could produce a negative channel total. Not addressed — FR-008 format and FR-010 zero-handling don't cover negatives.
10. **Audit write fails AND alert fails:** NFR-012/FR-017 raise an alert on audit-write failure, but if the alerting path itself is down, the outcome is silently lost. No secondary guarantee.
11. **Entitlement flips mid-window:** BR-6 says standard plan is entitled, but FR-014 re-derives each run. If entitlement is revoked between windows, the entitlement-blocked path is defined, but no notification to any stakeholder that a previously-delivered report stopped.
12. **Concurrent kill-switch flip during compose vs. send:** FR-016 covers "in flight completes under prior behavior," but doesn't define the state boundary — if flipped after compose but before send, does the composed email still go out?

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-017 (including FR-009a/b, FR-011a) are all written in Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"Appendix: Engineering Reference"** section with multi-paragraph narrative content (Affected Services lookups A/B, Discrepancy to resolve) beyond Sections 0–6. The trailing **"## Revision Summary"** is expected/sanctioned and is NOT flagged. The multi-paragraph engineering appendix with its own narrative analysis is the flagged content. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet-list only; no attribute tables or type declarations. (Note: Section 6 contains a table, which is acceptable for that section.) |

---

## KG Cross-Check (Discrepancy Note — informational)

The requirement-agent's independent fresh KG lookup re-found the **SAM-specific stack** (k4a-sam-backend, sam-report-scheduler, sam-performance-reporter, reporting-subscriptions-worker, notifications-api, sam-campaign-data-sync, cost-v2, frontend-mos) — i.e. it corroborates the spec's **Lookup B**, but did **not** re-find Lookup A's `reporting-api` or `accounts-api`. This strengthens the case that the SAM stack is the real report-generation path and that Lookup A's generic reporting stack may be the wrong attribution. The spec already correctly flags this discrepancy as unresolved (§1 caveat, CG-001) rather than picking a side, which is the right call — but the fresh lookup's failure to re-find `accounts-api` is worth noting, since FR-014's entitlement check needs a confirmed entitlement source and the spec never names one in-band. Recommend engineering confirm the entitlement source alongside the data-source decision.

---

## 6. Recommendations

1. **Resolve CG-001 before build starts.** Name the authoritative spend data source and report-generation path (reconcile Lookup A vs. Lookup B; fresh KG favors the SAM stack). This gates the 5-week deadline and all figure-correctness guarantees.
2. **Resolve CG-002 / DEC-008.** Either finalize the VP-CC rule and supply a confirmed VP address via config, or drop FR-011a from MVP. Do not ship restricted financial data to an unconfirmed recipient.
3. **Close the "incomplete data" detection gap (Edge Case #2/#3).** Define precisely what distinguishes a missing-channel from a genuine zero-spend channel, and what happens on all-zeros / no-data windows.
4. **Fill in NFR-002 numbers.** Set concrete p95 latency and concurrency values so the NFR is testable.
5. **Define DST, missed-run, and late-send behavior (Edge Cases #1, #4).** Specify spring-forward/fall-back handling and whether a missed Tuesday is ever caught up.
6. **Add bounce/acceptance-gap handling (Edge Case #5/#6).** Distinguish provider-accepted from delivered; define behavior on hard bounce or invalid distro list.
7. **Confirm the entitlement source** (KG cross-check did not re-find accounts-api) and the 12-month audit retention assumption (NFR-012).
8. **Move the multi-paragraph "Appendix: Engineering Reference" out of the product spec** or relocate it to a linked engineering doc to satisfy the Sections 0–6 structure (the Revision Summary trailer may stay).