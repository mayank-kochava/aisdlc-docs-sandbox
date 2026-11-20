# Spec Critique: K4A Weekly Attribution Report PDF Download

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-20

## Summary

- **Critical Issues:** 0
- **High Issues:** 5
- **Medium Issues:** 7
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | DEC-004 (historical window: most-recent-week vs rolling multi-week) is "deferred to engineering discretion." This is a product-content decision, not an engineering one — it changes what the finance team actually reconciles against their books at year-end close. Leaving it open risks shipping a report that fails the business need. | High | §1, FR-002 | Product must decide the window before build; do not delegate report content scope to engineering. |
| CG-002 | Kill-switch (FR-017) disables the feature "forward-only" and already-generated reports are not withdrawn. If the trigger is "delivery of incorrect/corrupted data" (NFR-020), the corrupted PDFs remain downloadable after the flip. No requirement quarantines or invalidates a bad already-generated artifact. | High | FR-017, NFR-020 | Add an FR to invalidate/quarantine already-generated artifacts for a period when the rollback trigger is data corruption, not just halt new generation. |
| CG-003 | FR-005/FR-006 define download success and FR-014 defines distinct failures, but no requirement covers a secure link that is successfully issued then *expires mid-download* or is *reused after expiry by the same authorized user*. FR-006 says "no longer usable after expiry" but the user-facing flow for re-issuing a fresh link is undefined. | High | FR-006, FR-014 | Add behavior: on expired-link access, re-issue a fresh time-limited link transparently to the still-authorized user rather than surfacing a dead-end error. |
| CG-004 | The timezone basis for the "Monday 9:00am local/business time" deadline is undefined for the generation path. NFR-004 commits to the deadline but does not define whose timezone governs when accounts span regions, nor behavior across DST transitions. | Medium | §1 Cadence, NFR-004 | Define the authoritative timezone (account business TZ) and the DST handling rule explicitly. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "should respond within a p95 latency target... Engineering may refine the exact threshold against real artifact sizes" | The escape clause ("may refine") makes the 2s target non-binding and therefore untestable as an acceptance gate. | Given a generated report ≤ [defined size] exists, When 50 concurrent authorized users request download, Then the download-initiation response p95 ≤ 2s. (Remove the "may refine" clause or bound the artifact size.) |
| NFR-004 | "complete all enrolled accounts' reports within the weekly window with margin" | "with margin" is not quantified; "the weekly window" start is undefined. | Given the weekly schedule fires at [defined start time], When all enrolled accounts are processed, Then 100% of reports reach ready status no later than Monday 08:00 account business time (1h margin before the 9:00am deadline). |
| FR-017 | "new requests reflect the disabled state within an established propagation window" (note: 5-minute default) | "established propagation window" is undefined in the testable body; the 5-min value is only in a note. | Given the kill-switch is flipped to disabled at T, When a new request arrives at T+5min, Then the request is rejected with the disabled state. |
| NFR-013 / FR-019 | "retained for a minimum of 12 months within the established platform range" | "established platform range" is unquantified — the upper bound is untestable. | Given an audit record written at T, When queried at T+12 months, Then the record is still retrievable and unchanged; define the max retention value. |

> Note: All FRs are correctly written in Gherkin. The testability issues above are confined to vague NFR prose and under-specified thresholds, plus body-vs-note value placement in FR-017.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard year-end deadline (~5 weeks) — NFR-017 | Partial | PASS (weak) | Mitigation is "budget discovery time explicitly" — a process assertion, not a verifiable requirement. Acceptable given MVP minimization, but there is no requirement defining what happens if discovery overruns the 5 weeks. |
| Multi-service blast radius / unsettled scope — NFR-018 | Yes | PASS | Kill-switch (FR-017) + discovery-before-estimation. Reasonable. |
| Finance/billing data sensitivity — NFR-019 | Yes | PASS | Fully decomposed into NFR-008..013 with concrete enforcement. Strong. |
| Shared attribution source degradation — NFR-003/NFR-021 | Yes | PASS | Explicit rate cap + backward-compat constraint. Good. |
| Corrupted-data rollback (NFR-020 trigger) | No | FAIL | Rollback is forward-only; a "corrupted data" trigger has no mechanism to withdraw the already-served corrupted artifact (see CG-002). The stated trigger and the stated data-handling contradict each other. |

---

## 4. Missed Edge Cases

1. **Expired secure link re-issuance:** FR-006 kills the link after expiry but no flow re-issues one for a still-authorized user (see CG-003).
2. **Corrupted artifact already downloaded before kill-switch flip:** No recall/notification path once a bad PDF is in finance's hands.
3. **Partial attribution data:** Q (data source reachable?) is binary in the flow, but what if the source is reachable yet returns *incomplete* channel data (e.g., revenue missing for one channel)? No requirement distinguishes "reachable but incomplete" from success — risks shipping a silently wrong finance report.
4. **Zero-activity week:** No requirement defines whether a week with no spend/installs/revenue produces an empty-but-valid report or a "no report" state. Finance reconciliation needs to distinguish "zero activity" from "not generated."
5. **Account enrolled mid-week:** FR-001 generates for enrolled accounts, but enrollment timing relative to the weekly period boundary is undefined — does a mid-week enrollment get a partial first week?
6. **Account de-enrolled / plan downgrade mid-period:** Report says no entitlement gating, but what if the account is deactivated/offboarded after generation but before download? Authorization re-derivation (FR-009) covers the user but not account lifecycle state.
7. **Concurrent download by multiple finance users:** FR-004 handles concurrent *generation*; nothing addresses multiple finance users requesting the same report simultaneously (likely benign, but rate-limit interaction with NFR-002 is unspecified).
8. **Retry-After larger than the generation window:** FR-013/NFR-014 honor Retry-After, but a Retry-After exceeding the time left before the Monday 9:00am deadline is not handled — could silently blow the deadline.
9. **DST / timezone boundary for weekly cadence:** See CG-004 — the "Monday 9:00am local" definition is ambiguous across DST and multi-region accounts.
10. **Audit-write failure after successful download:** FR-018 writes audit asynchronously and alerts on failure, but does not define whether the download is considered compliant if the audit never persists for finance-sensitive data. Possible compliance gap depending on policy.
11. **Kill-switch flipped mid-generation at deadline:** FR-017 lets in-flight runs complete under prior behavior; if a run completes a corrupted report after the operator intended to stop it, the operator's intent is defeated.
12. **Stale period displayed after rollback:** If generation is disabled, FR-007's "no report" state vs. a previously-available report for the current period is unspecified when the kill-switch is active.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-019 all use Gherkin blocks. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present. The trailing "Appendix: Engineering Reference" is sanctioned grounding evidence (labelled engineering reference, KG-derived, explicitly marked not-required-for-review) and the trailing "## Revision Summary" is the expected spec-revise trailer — neither is flagged. No duplicate or stray narrative sections found. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list items; no attribute tables or type declarations present. |

> Note on Section 0: no explicit "Section 0" header exists, but the document opens directly at Section 1 (Business Logic & Context), which is conventional. Not a failure, but author may confirm Section 0 is intentionally merged/omitted.

---

### Grounding cross-check against independent KG evidence

The spec's Section 1 and appendix **correctly and transparently** surface the two-list discrepancy rather than silently picking one side: the primary list includes `k4a-mcp-server`; the independent cross-check did not re-find it and added `koch-query-api` and `summary-report-cloud` at low confidence. This matches the independent KG evidence provided. One minor note: the spec leans on `k4a-mcp-server` for "plan entitlement check," but FR-011 resolves that there is **no plan-entitlement gating** for this feature — so the entitlement-check justification for that service is partly moot, which further weakens confidence that `k4a-mcp-server` is truly in blast radius. Recommend engineering drop the entitlement-check rationale when confirming that service. No hard contradiction between spec claims and KG evidence; the data-flow and service roles are consistent.

---

## 6. Recommendations

1. **Resolve CG-002 / NFR-020 contradiction:** Add a requirement to invalidate or quarantine already-generated artifacts when the rollback trigger is data corruption — forward-only rollback is insufficient for a "corrupted data" trigger.
2. **Pull DEC-004 (historical window) back to product:** This is report-content scope affecting finance reconciliation, not an engineering-discretion item.
3. **Define expired-link re-issuance (CG-003):** Specify transparent re-issue of a fresh secure link for still-authorized users.
4. **Add "reachable but incomplete data" handling (Edge Case 3):** Distinguish partial/incomplete attribution from success to avoid silently wrong finance reports.
5. **Quantify vague NFRs:** Bind NFR-001 (remove "may refine"), NFR-004 ("with margin" → explicit margin + window start), and move the FR-017 propagation value into the testable body.
6. **Define timezone/DST basis** for the Monday 9:00am deadline and handle Retry-After exceeding the remaining deadline window (Edge Cases 8, 9).
7. **Specify zero-activity and mid-week-enrollment report behavior** (Edge Cases 4, 5).
8. **Confirm audit-write-failure policy** for finance-sensitive downloads (Edge Case 10) — define whether a download is compliant without a persisted audit record.