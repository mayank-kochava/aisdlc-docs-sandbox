# Spec Critique: K4A Weekly Attribution Report (Brightway Retail)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-21

## Summary

- **Critical Issues:** 0
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | FR-005 ("no overlapping generation runs") says the second run "does not start concurrently" but never defines the second run's terminal state — is it dropped, queued, or deferred to next cycle? A run that is silently dropped could skip a week's report with no trace. | High | 3.1 | Specify the loser's outcome: dropped-as-redundant vs. queued-after. Add a Then clause and a named status so the drop is observable. |
| CG-002 | FR-002/FR-003 never define behavior across a quarter boundary. If the weekly cycle runs on the first day of a new quarter, does "most recent completed quarter" silently shift the window, producing a report for a different quarter than last week with no indication to the finance user? For a year-end close this is a correctness landmine. | High | 3.1 | Add an FR defining window selection at quarter boundaries and whether the reported quarter label is surfaced to the user so a shift is visible. |
| CG-003 | FR-013 generates against partial data and flags it, but there is no requirement that a *later, now-complete* report supersedes the earlier partial one, nor how the finance user distinguishes a partial report from the final close-ready report when both exist. Finance could close the year on a report flagged partial. | High | 3.3 | Add an FR: when a window's data finalizes after a partial report was issued, regeneration produces a superseding final report and the partial is clearly marked superseded/stale. |
| CG-004 | FR-010/FR-015 define the success path and the secure-link expiry, but no FR covers the user clicking an **expired** download link (the 15-min NFR-011 window lapses while the user is away). The flow diagram has no node for "link expired." Dead end for the user. | High | 3.4 | Add an FR: expired link → clear re-request path that re-authorizes (FR-012) and mints a fresh link, not a raw error. |

No Critical-severity issues were found: the spec consistently defines failure statuses, a fail-closed auth posture, idempotency, and a kill-switch, so nothing rises to blocking data-loss/security level. The gaps above are serious testability/behavior gaps (High) rather than implementation blockers.

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-002 (note) | "generation early on a fixed weekday so a fresh report is available at the start of each business week" | "early" and "start of business week" are untestable; no timezone. | Given weekly generation is enabled, When the scheduled time (fixed weekday HH:MM in a named timezone, e.g. 02:00 America/New_York Monday) is reached, Then generation starts, And a report is available before 09:00 local that day. |
| FR-013 | "the report clearly indicates that some data was incomplete" | "clearly indicates" is not test-assertable — where, what text, what machine-readable flag? | Given a report is generated against partial data, When the PDF is produced, Then a visible "Data incomplete as of <timestamp>" banner appears on page 1, And the report instance carries a completeness=PARTIAL flag queryable via the report list. |
| FR-011 | "presents a clear 'no report available' state" | "clear" is subjective. | Given no report exists for a requested week, When the user opens the report area, Then the UI displays the literal empty-state message for that week, And the download control is rendered disabled/absent (assertable in a UI test). |
| NFR-001 | "measured under expected concurrent finance-user load" | "expected load" is undefined — not a reproducible test condition. | Given N concurrent download requests for one account (define N, e.g. 10), When measured over the test window, Then p95 PDF serve/stream-start latency ≤ 3s. |
| NFR-002 | "throttled to a bounded concurrency against the source; the exact cap is an engineering default" | No numeric cap means no pass/fail test for "cannot starve existing traffic." | Given generation reads against the attribution source, When generation runs, Then concurrent reads never exceed the configured cap C (define C), And existing attribution traffic p95 latency rises ≤ X% during generation. |

All Section 3 FRs are correctly written in Gherkin; the items above are either embedded prose *within* Gherkin Then-clauses that are non-measurable, or NFRs stated as prose with undefined thresholds.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard 5-week year-end deadline | Partial | FAIL | Mitigation is "answer the decisions early" — an aspiration, not a requirement. No FR enforces a decision-by date or defines what ships if DEC-001/006 slip. The deadline risk has no buildable mitigation. |
| Multi-service blast radius (6+ services, KG passes disagree) | Yes | PASS | Kill-switch (NFR-015/FR-017) + reconciliation task in eng-plan + backward-compat NFR-017. Adequate as a product-level posture. |
| Unconfirmed UX scope | Yes | PASS | Phase 1 constrained to minimum download surface; richer management deferred in Section 6. Testable boundary. |
| Attribution data incompleteness / mid-processing reads | Yes | PASS | FR-013/FR-014 + BR-7 + NFR-006. Covered, though see CG-003 (supersession gap). |
| Attribution source dependency failure | Yes | PASS | FR-007/FR-008 + NFR-006 define bounded retry, retry-after honoring, and distinct statuses. Strong. |
| Auth-system unavailability | Yes | PASS | NFR-010 fail-closed + FR-012 re-derive per request. |
| Restricted data class in PDF (DEC-007) | Deferred | FAIL | Risk is acknowledged but handling is entirely conditional on an open decision. No default masking/separate-store requirement exists if the answer is "yes." Until DEC-007 lands this is an unmitigated data-sensitivity risk, not just an open question. |

---

## 4. Missed Edge Cases

1. **Quarter-boundary window shift:** Weekly run crossing into a new quarter silently changes the reported quarter (see CG-002).
2. **Partial report never superseded:** Data finalizes but no regeneration replaces the partial close-ready report (see CG-003).
3. **Expired secure link re-request:** No defined recovery when the 15-min link lapses (see CG-004).
4. **Empty report after a previously populated week:** FR-014 covers "no data at all," but a window that *had* data and now returns zero rows (data deletion/correction upstream) isn't addressed — could produce a misleadingly empty report that passes the "data available" check.
5. **PDF generation itself fails after data read succeeds:** FR-007/008 cover *data-source* failure; a failure in the PDF rendering/export step (reporting-api) has no named status or alert.
6. **Report size / timeout on generation:** No bound on generation duration or report size; a pathological quarter could run unbounded. NFR-001 explicitly exempts generation but nothing caps it.
7. **Kill-switch flipped mid-generation:** FR-017 covers in-flight *requests* completing under prior behavior, but an in-flight *generation job* at flip time is undefined — does the half-produced report persist?
8. **Concurrent downloads of the same report by multiple finance users:** NFR-001 mentions concurrent load but no FR confirms the report is immutable/consistent across simultaneous downloads.
9. **Audit-write failure during a download:** NFR-013 fires an alert on audit failure but doesn't say whether the download still proceeds (async implies yes) — confirm the primary op is never blocked AND the gap is reconcilable.
10. **Clock skew / timezone of "weekly" and "last quarter":** No timezone anchor for either the cadence or the quarter boundary; finance close is date-sensitive.
11. **Retention expiry mid-download:** A report whose retention (BR-8) lapses while a user holds a valid secure link — does FR-015's link outlive retention deletion?
12. **Browser "Back" during download / re-trigger:** Happy-path UI assumes a single clean download; repeated/interrupted download clicks and their effect on the time-limited link aren't specified.
13. **DEC-007 "yes" path has no requirements:** If the PDF carries restricted data, no masking/separate-store FR exists to test against.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001–FR-018 are all Gherkin. The sentence "Now reporting the decisions I flagged in Sections 1–3." after FR-018 is stray narrative prose, not an FR — recommend removing, but it is not a prose FR. |
| Document contains only Sections 0–6 | FAIL | The document includes an "## Appendix: Engineering Reference" (Affected Services) beyond Section 6. This is a multi-item engineering appendix with its own narrative content, not a sanctioned Revision Summary or a one-line `> **Label:** ...` metadata trailer. It should be removed or folded into the Section 1 service-map note / an eng-plan artifact. (The Section 1 "Service-map note" itself is in-bounds as a subsection of Section 1.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list of concepts with no attribute tables or declared types. |

---

## 6. Recommendations

1. **Remove or relocate the "Appendix: Engineering Reference" section** so the document contains only Sections 0–6 (format FAIL). The service-map disagreement is already captured in the Section 1 note; the affected-services list belongs in an eng-context/eng-plan artifact, not the spec.
2. **Close the quarter-boundary and partial-supersession gaps (CG-002, CG-003)** — these directly threaten year-end-close correctness. Add FRs defining window selection across quarter boundaries and supersession of partial reports once data finalizes.
3. **Add the expired-secure-link recovery FR (CG-004)** and the empty-after-populated and PDF-generation-failure cases (edge cases 4, 5) with named statuses/alerts consistent with FR-008.
4. **Convert the deadline and DEC-007 risks into buildable mitigations** (Risk Audit FAILs): define a decision-by date / default-ship behavior for the open decisions, and a default masking/separate-store requirement for the restricted-data path rather than leaving it fully conditional.
5. **Make NFR-001 and NFR-002 testable** by supplying the concurrency number N and the read cap C (or the test conditions), and anchor FR-002's cadence and FR-003's quarter to an explicit timezone.
6. **Tighten FR-005** to name the losing run's terminal state (CG-001) and **FR-017** to define in-flight *generation job* behavior at kill-switch flip (edge case 7).
7. **Clarify the KG cross-check discrepancy (no contradiction with evidence found):** The independent fresh KG lookup does *not* re-find `k4a-mcp-server` (summary-pass only in the spec) and agrees the other seven services are at best low-medium confidence. The spec's own Section 1 note and appendix honestly flag this as unreconciled — no claim in the spec contradicts the evidence; it correctly defers reconciliation to eng-plan. Keep that deferral, but ensure the appendix removal (Rec 1) doesn't drop the unreconciled-service flag — preserve it in the Section 1 note.