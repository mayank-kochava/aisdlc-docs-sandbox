# Spec Critique

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-21

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Download-consistency rule (FR-005) is explicitly unsettled (DEC-003) yet drives a functional requirement AND two information requirements (snapshot identity, availability state).** If the "frozen weekly snapshot" assumption is wrong, FR-004 (overwrite-in-place) and FR-005 directly conflict: an overwrite during the week would change what a repeat download returns, violating FR-005. The spec ships a Critical behavioral ambiguity as a "working assumption." | Critical | §1 Download consistency, FR-004, FR-005 | Force a decision on DEC-003 before engineering. If snapshots are frozen, FR-004's overwrite must be scoped to *regeneration-of-a-failed-run only*, not routine weekly overwrite, or the two FRs are mutually exclusive. |
| CG-002 | **No defined behavior when "last quarter" boundary rolls over mid-week.** FR-001 generates weekly; FR-002 scopes to "most recently completed calendar quarter." On the first weekly run after a quarter boundary (e.g., Jan 1), the report silently changes the entire dataset underneath finance mid-reconciliation. There is no requirement defining whether the just-closed quarter's final report is preserved, nor whether finance is notified the scope shifted. For a year-end-close use case this is a data-integrity dead end. | Critical | FR-001, FR-002, §1 Value | Add an FR: when the last-quarter window advances, define (a) whether the prior quarter's final weekly report remains downloadable, and (b) how the UI communicates which quarter the current report covers. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "completes within a bounded generation window such that the report is marked available before the finance team's expected weekly access time" | "Bounded window" and "expected weekly access time" are undefined — not testable; no clock value exists. | Given a weekly generation cycle starts, When the account's report is generated, Then it is marked available no later than 06:00 UTC on the scheduled delivery day (value TBD by product). |
| NFR-001 | "p95 latency target of 3 seconds ... under expected concurrent finance-user load" | "Expected concurrent finance-user load" is unquantified — a p95 with no defined concurrency/N is not reproducible. | Given N=[TBD] concurrent authorized finance users, When each triggers download of an already-generated report, Then p95 time-to-first-byte ≤ 3s. |
| FR-018 | **"No PII in logs" — header only, NO Gherkin body at all.** | Not written as a requirement; empty FR. Untestable because it does not exist. | Given the report generation/download path writes log entries, When any entry is written, Then it contains no raw PII, credentials, secrets, or tokens (ties NFR-011). |
| FR-007 | "a clear 'report not available' message" | "Clear" is subjective/untestable. | ...Then the user is shown message text matching the approved copy for "report not available" (string ID [TBD]), And no download request is issued. |
| NFR-014 / NFR-015 | "kill-switch propagation window" (NFR-015) references a window only loosely tied to NFR-014's "within 5 minutes." | Rollback target latency is indirect; a tester cannot assert the SLA without the explicit number restated. | Given the kill-switch is flipped, When 5 minutes elapse, Then 100% of new generation and download requests reflect the disabled state. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Shared query-resource contention (koch-query-api) | Yes — NFR-003 rate cap, NFR-005 bounded retry/Retry-After | PASS | Well specified. |
| Downstream transient failure | Yes — NFR-005, FR-003 | PASS | — |
| Authorization-source unavailability | Yes — FR-014, NFR-007 fail-closed | PASS | — |
| HIGH-risk, multi-service work | Yes — NFR-014 kill-switch, NFR-015 rollback | PASS | — |
| **Affected-services uncertainty (summary-report-cloud / koch-query-api low confidence; reporting-dispatcher/subscriptions-worker low confidence)** | Partially — flagged for discovery, but no requirement handles "summary-report-cloud does not exist or does not produce required content" | **FAIL** | §1 commits to *reusing* summary-report-cloud "if one exists." There is no fallback FR for the case where it does not exist or lacks the attribution content. This is an unmitigated dependency risk on the chosen assembly path. |
| **Where the last-quarter filter is applied (KG flags koch-query-api "may be where the filter must be applied")** | No | **FAIL** | FR-002 asserts the filter semantics but no requirement assigns responsibility for *where* the quarter boundary is enforced. If koch-query-api applies its own window, FR-002 could double-filter or mis-align. Needs a requirement pinning the filter owner. |
| **Audit-write failure path** (FR-017/NFR-012: async, alert-not-block) | Partial | **FAIL** | Async non-blocking audit means a download can succeed with NO audit record (only an alert). For a "traceable, access-controlled operation," a silently-missing audit record on a successful download is a compliance gap. No requirement defines reconciliation/backfill of dropped audit writes. |

---

## 4. Missed Edge Cases

1. **Quarter rollover mid-week (data swap):** See CG-002 — the dataset changes under the user with no versioning of the prior quarter's report.
2. **Empty-result quarter:** No FR covers a last quarter with zero attribution data. Is a valid PDF with a zeroed table produced, or treated as a generation failure? FR-003 (failure) and FR-015 (content present) both ambiguously apply.
3. **Partial channel data / late-arriving attribution:** Attribution data commonly settles over time. No requirement addresses whether the frozen weekly snapshot is taken before data has fully settled, affecting reconciliation accuracy.
4. **User authorized at page render, deauthorized before clicking download:** FR-012 gates the affordance at render; FR-010/FR-011 re-derive on request — good — but no edge case defines the UX when the affordance is visible but the request is then denied (stale-page scenario).
5. **Plan downgrade mid-week:** Account on standard plan when report generated, downgraded before download. FR-013 enforces eligibility per-request (good), but no requirement states whether an already-generated report becomes inaccessible or is retained for retention compliance.
6. **Concurrent downloads of a report during an in-flight overwrite (FR-004):** A download initiated while an idempotent overwrite is replacing the file — torn/partial read of the PDF is undefined.
7. **Kill-switch flip during generation (not download):** NFR-014 defines in-flight *download* behavior but not an in-flight *generation* run at the moment of the flip.
8. **Secure-link scope (FR-009/NFR-010):** The link is defined as valid 7 days but there is no requirement that it be single-recipient / non-enumerable / revocable before expiry. An expired link is handled; a leaked-but-unexpired link is not.
9. **Browser "Back" / re-submit during download:** The Happy-Path flow (§2) shows a single linear download; no handling for re-click, double-submit, or cancelled download.
10. **Two-year retention expiry behavior:** NFR-013 retains PDFs two years; no requirement defines what happens *at* expiry (hard delete? access denied? audit of the deletion?).
11. **Audit record for the "report not available" path (FR-007):** FR-017 covers obtained/denied, but a not-available view is neither — ambiguous whether it is audited.
12. **Timezone of "week" and "quarter":** FR-002 note implies a calendar basis but no timezone is specified. "Last quarter" and "same week" (FR-005) are timezone-dependent; cross-TZ finance users could see boundary disagreements.
13. **Multiple finance users, concurrent first-ever generation:** FR-004 covers scheduler-level overlap for the same key, but not a user-triggered download arriving before the week's first generation completes (race with FR-007).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | FAIL | **FR-018 ("No PII in logs") has a header and inline comment context but NO Gherkin body** — it is an empty/prose stub, not a Given/When/Then requirement. All other FRs (FR-001–FR-017, FR-019) are valid Gherkin. |
| Document contains only Sections 0–6 | FAIL | The document contains an **"## Appendix: Engineering Reference"** section (Affected Services narrative) placed *before* the Revision Summary. A multi-item engineering appendix with derived service descriptions is beyond Sections 0–6 and is not the sanctioned trailing Revision Summary nor a one-line labeled metadata blockquote. The trailing **"## Revision Summary"** is expected and is NOT flagged. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list prose with no attribute tables or type declarations. |

---

## 6. Recommendations

1. **Resolve DEC-003 (download consistency) before engineering** and reconcile FR-004 vs FR-005 — these are mutually contradictory under routine overwrite and constitute a Critical data-integrity gap (CG-001).
2. **Add a quarter-rollover requirement (CG-002):** define prior-quarter report preservation and explicit UI indication of which quarter the current report covers; critical for the year-end-close use case.
3. **Write FR-018 as real Gherkin** — it is currently an empty stub and fails format compliance.
4. **Move the "Appendix: Engineering Reference" content out of the spec body** (or relocate into an approved Section 0–6 location); as written it breaks the Sections 0–6 constraint.
5. **Add a fallback requirement for the summary-report-cloud assembly dependency** — the spec commits to reuse "if one exists" with no behavior if it does not exist or lacks attribution content (Risk Audit FAIL). Also pin *where* the last-quarter filter is applied (KG flags koch-query-api as a candidate), to avoid double/mis-filtering of FR-002.
6. **Quantify NFR-001 concurrency and NFR-002 generation deadline** with concrete values so they are testable.
7. **Specify empty-quarter, data-settling, retention-expiry, timezone, and secure-link-revocation edge cases** (Section 4 items 2, 3, 8, 10, 12) as explicit FRs.