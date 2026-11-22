# Spec Critique

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 1
- **High Issues:** 5
- **Medium Issues:** 6
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Direct contradiction between the business rules and the FRs on partial data.** BR-3a states the email **is still sent** with available data plus a caveat note ("the report does not wait for complete data"). The User Flow (Section 2, node F2) matches this — it assembles partial data + a note and sends. But FR-012 states the system **does not send** partial/incomplete totals and instead records a "data-not-ready" failure. FR-014 and NFR-007 ("never sends partial/incomplete spend totals") reinforce the *opposite* of BR-3a. This is a fundamental behavioral contradiction: should partial data be SENT-with-caveat or NEVER sent? Engineering cannot implement both. | Critical | BR-3a vs FR-012/FR-014/NFR-007, Section 2 flow | Pick one policy and make all of BR-3a, the Mermaid flow, FR-012/013/014, and NFR-007 agree. Given the FRs and NFR-007 are internally consistent (fail-closed on correctness), the likely fix is to rewrite BR-3a and the flow's F2 node to match "never send partial — retry within lag window, else missed-send." If the business genuinely wants send-with-caveat, the entire Feature Area E and NFR-007 must be rewritten. |
| CG-002 | **FR-003 idempotent-overwrite conflicts with the fail-closed/no-unsend posture.** FR-003 says a re-run "overwrites the prior generation for that week rather than producing a duplicate email record." But email is a push channel — once delivered (FR-010/FR-011) it cannot be overwritten or unsent (Section 6 Rollback confirms "already-sent emails cannot be unsent"). "Overwrite the generation record" vs. "re-send a corrected email" is undefined. If a corrected re-run produces new numbers, does a second email go out to recipients or not? | High | FR-003 vs Section 6 Rollback | Clarify that FR-003 idempotency applies to the *generation record / send-attempt* state only, and explicitly state whether a corrected re-run triggers a NEW email to recipients (and if so, how recipients distinguish it from the original). |

---

## 2. Testability Review

The FRs are generally well-formed Gherkin. The issues below are measurability/ambiguity gaps, not prose-format violations.

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-009 | "...laid out so the marketing team can copy them directly into their existing spreadsheet **without manual reformatting**" | "Without manual reformatting" is not objectively testable — there is no defined target format (column order, delimiter, currency formatting, header row). | `Given the one-line-per-channel content has been assembled, When the email is formatted, Then the body contains an HTML/plain table with a header row "Channel, Total Spend (USD)", one row per channel in the fixed order Email→paid search→paid social→display, with spend rendered as a numeric value with exactly 2 decimal places and no currency symbol, such that a paste into a spreadsheet lands each value in its own cell.` |
| FR-013 | "...proceeds to send once the data is available **and complete**" — but "complete" is never defined; FR-012 references a "data-readiness/completeness signal" whose semantics are unspecified. | "Complete" is untestable without a definition of completeness (all 4 channels present? ad-platform sync watermark reached?). | `Given spend data was not ready, When all four fixed channels report a sync-watermark timestamp ≥ the covered week's end, Then the data is treated as complete and the email is sent.` (Define the actual completeness predicate.) |
| NFR-003 | "...must be generated and delivered **within the acceptable lag window** of the configured send time" | Circular/self-referential — the "acceptable lag window" is itself declared an undefined "engineering/operational default" (FR-013 note, Info Req "Acceptable lag window"). No concrete number anywhere means NFR-003 is untestable. | `Given a normal week with complete data, When the send schedule fires Monday 08:00 ET, Then the email is delivered to all recipients no later than [X hours] after the covered week ends.` Business or eng must supply X. |
| NFR-004 | "...failures... exceed 1% of scheduled sends over a **15-minute window, sustained**" | For a single standard-plan account sending **once per week**, "1% over a 15-minute window" is nonsensical — there is at most one send per week, so a 15-minute windowed percentage threshold can never meaningfully trigger. The "sustained" qualifier is also undefined. | Drop the percentage/window threshold for this single-account MVP; rely on the "alert on any single missed-send" clause, which is the only testable rule here: `Given a scheduled weekly send, When it results in missed-send, data-not-ready at window-exceed, or delivery-failure, Then an operational alert fires immediately.` |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (5 weeks) | Yes — MVP scope + reuse-existing-pipeline direction | PASS | Mitigation is a concrete scope constraint, not just an acknowledgement. |
| Multi-service blast radius / conflicting service names | Partially — kill-switch, bounded retries, "reconcile names in planning" | FAIL | The mitigation defers the core unknown ("engineering reconciles names") rather than resolving it. No requirement pins down WHICH service is the source of truth for spend, nor what happens if the chosen scheduler/email pipeline already serves other consumers. See CG/edge cases below. |
| Email delivery system integration | Yes — FR-015, NFR-004/005, NFR-009/010 | PASS | Delivery failure is a distinct, retried, alerted, access-confined condition. |
| Data freshness / sync-timing | Yes — FR-012–014, NFR-007 | FAIL (tainted) | The mitigation set is *internally contradicted by BR-3a* (CG-001). A mitigation that contradicts its own business rule cannot be relied upon. Must resolve CG-001 before this passes. |

---

## 4. Missed Edge Cases

1. **Partial-send policy contradiction (see CG-001):** Not an edge case so much as a core-path contradiction, but the "some channels synced, others not" sub-case is entirely undefined once CG-001 is resolved — if the policy becomes "never send partial," is 3-of-4 channels "incomplete"? What if a channel legitimately has zero spend vs. hasn't-synced-yet? FR-008 (zero spend) and FR-012 (data-not-ready) are indistinguishable at the data layer unless a completeness signal separates "real zero" from "not yet reported."
2. **Currency / multi-currency:** No requirement specifies currency. If Meridian Travel spends in more than one currency across channels, totals are meaningless without a currency column or normalization rule.
3. **Negative / adjusted spend:** Ad platforms issue refunds/credits producing negative spend. No FR states whether a negative channel total is displayed as-is, floored at zero, or flagged.
4. **Email rendering fidelity:** FR-009 mandates an in-body table but no requirement covers clients that strip HTML tables (plain-text fallback). "Copy directly into spreadsheet" silently fails in a plain-text client.
5. **Recipient list empty / all-invalid:** FR-010 says "one or more recipients configured." No FR covers the case where the configured list is empty or every address hard-bounces — is that a delivery-failure, a missed-send, or silent?
6. **DST / timezone boundary:** Week boundary and send are "Eastern time." The Monday 08:00 ET send and the Monday–Sunday ET week boundary are not tested across the spring-forward / fall-back DST transition, where a "day" is 23 or 25 hours. Contiguity (FR-005) across a DST weekend is unverified.
7. **Kill-switch flipped mid-week vs. at send time:** NFR-012 says an in-flight run completes under prior behavior. But if the switch is off at the Monday trigger and turned on Tuesday, is the missed Monday send backfilled or permanently skipped? (Section 6 says no backfill — but that's a historical-backfill exclusion, not a kill-switch-recovery rule.)
8. **Account leaves standard plan mid-cycle:** NFR-008 re-derives eligibility at run time. No FR states the recipient-visible outcome if Meridian Travel is downgraded/suspended between weeks — silent stop, or notify owner?
9. **Audit-write failure during a successful send:** NFR-011 says audit is async and a write failure alerts but doesn't block delivery. This means an email can reach the customer with NO audit record — acceptable for availability but a traceability gap that should be explicitly acknowledged as an accepted trade-off.
10. **Duplicate alert storm:** NFR-004 alerts on "any single missed-send" AND FR-012/014/015 each raise operational alerts. For one failed week this could fire multiple overlapping alerts; no de-duplication rule.
11. **Retry (FR-013) crossing the Monday 08:00 send time:** If retries for last week's data are still running when the NEXT week's trigger fires, FR-002's overlap guard is keyed per-week, so two *different* weeks' runs could execute concurrently and contend on the rate-limited data source (NFR-002). Interaction untested.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-015 are all in fenced Gherkin blocks. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an `## Appendix: Engineering Reference` section (with "Affected Services," cross-check passes, and "Reconciliation notes for planning" — multi-paragraph narrative analysis with its own findings). This is exactly the kind of stray multi-paragraph KG cross-check analysis appendix that is NOT the sanctioned Revision Summary / one-line metadata trailer exception. The trailing `## Revision Summary` section itself is EXPECTED and correctly NOT flagged. The violation is the Engineering Reference appendix sitting between Section 6 and the Revision Summary. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 (Information Requirements) is a bullet list; field names/types are explicitly deferred to eng-plan, and no attribute tables appear. |

---

## KG Cross-Check Findings (grounding evidence vs. spec claims)

The independent KG cross-check re-found the SAME eight candidate services the spec already records in its own cross-check pass (`k4a-sam-backend`, `sam-report-scheduler`, `sam-performance-reporter`, `reporting-subscriptions-worker`, `summary-report-cloud`, `notifications-api`, `k4a-mcp-server`, `sam-campaign-data-sync`), with matching roles and confidence levels. **No new discrepancy** between the fresh lookup and the spec's recorded evidence — the spec has already honestly surfaced the two-pass naming conflict and flagged it in Section 1 for engineering reconciliation. This is handled correctly as grounding, not silently resolved. The spec makes no hard claims about specific tables/joins/endpoints that contradict the evidence (it deliberately defers all of that). The only KG-related issue is organizational, not factual: per Section 5 Format Compliance, the reconciliation detail belongs in eng-plan, not in an appendix on the product spec.

---

## 6. Recommendations

1. **Resolve CG-001 (Critical) first.** Reconcile BR-3a + the Mermaid F2 node with FR-012/FR-014/NFR-007. The spec currently tells engineering to both send and never-send partial data. Pick one policy and make every section agree before engineering starts.
2. **Resolve CG-002 / FR-003.** Define precisely whether a corrected idempotent re-run re-sends a new email to recipients, given emails cannot be unsent.
3. **Define "complete" and the "acceptable lag window" with concrete values.** FR-012/FR-013, NFR-003, and the Info Reqs all lean on these undefined terms; NFR-003 is currently circular and untestable.
4. **Fix NFR-004's threshold** — a "1% over 15-minute window" rule is meaningless for a once-weekly single-account send; keep only the per-miss immediate alert and add de-duplication.
5. **Remove the Engineering Reference appendix from the product spec** (Format FAIL) — relocate the affected-services/reconciliation content to eng-plan; keep only the short Section 1 service note pointer.
6. **Make FR-009 testable** by specifying the concrete table format (column order, headers, numeric formatting, plain-text fallback).
7. **Add FRs for the missed edge cases**, prioritizing currency (#2), empty/all-bounced recipient list (#5), DST week-boundary/send (#6), and zero-spend-vs-not-synced disambiguation (#1).