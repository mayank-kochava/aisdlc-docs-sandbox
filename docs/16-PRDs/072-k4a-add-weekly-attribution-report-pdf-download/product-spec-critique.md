# Spec Critique: Brightway Retail Weekly Attribution Report

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
| CG-001 | **FR-002/BR-6 directly contradict each other.** BR-6 states the report is a *manual, user-initiated trigger from the console* ("not generated without a user-initiated trigger"). FR-001/FR-002 and the entire Section 2 flow define **system-triggered weekly scheduled generation** with no user trigger. Section 6 even lists "Manual on-demand report triggering" as *out of scope*. These cannot both be true — the core generation model of the feature is ambiguous. Engineering cannot build a trigger that is simultaneously required (BR-6) and out of scope (Section 6). | Critical | 1 (BR-6) / 3.1 / 6 | Reconcile the trigger model. Rewrite BR-6 to match the scheduled-generation model actually specified in FRs, or vice versa. Do not ship with the business rule and the FRs in opposition. |
| CG-002 | **"Partial data" report has no dead-end / correction path.** FR-013 produces a report marked partial, then makes it permanently available (FR-009) for 12 months (BR-8). There is no requirement that a later, fully-final report for the *same* window supersedes the partial one. A finance user performing year-end close could download a partial report believing it is final, with no mechanism to detect staleness or to regenerate once data finalizes. Given this spec's stated purpose is a **financial close**, serving stale/partial numbers is a data-integrity/financial-correctness risk. | Critical | 3.3 (FR-013) / 1 (BR-7, BR-8) | Add an FR: when a window previously generated as partial later has all data final, the next weekly cycle must regenerate and supersede the partial instance (idempotent on account+window per FR-004), and the UI must indicate recency/finality. |

---

## 2. Testability Review

Note: all FRs are already in Gherkin. The issues below are **untestable predicates** inside otherwise well-formed Gherkin, plus vague NFR prose.

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-006 | "it reads attribution data ... from **the authoritative attribution data source**" | "The authoritative attribution data source" is never named (and the KG passes disagree on which service it is — see §3). Not testable without a concrete source. | Given the system generates a report, When it assembles contents, Then it reads attribution data for the window from `pubsuite-attribution` (pending §1 service-map reconciliation), And it performs no write against that source. |
| FR-009 | "reports are **continuously available** for download" (note) | "Continuously" is unbounded and conflicts with BR-8's 12-month retention. Not measurable. | Given a report was generated for account+window, When ≤ 12 months have elapsed since generation, Then the report is downloadable via the K4A console; When > 12 months have elapsed, Then it is no longer offered for download. |
| FR-013 | "report is **clearly marked** as containing partial data" | "Clearly marked" has no testable assertion (where? what text? machine-readable flag?). | ...Then the report PDF displays a visible "PARTIAL DATA — some data incomplete as of <generation timestamp>" banner on page 1, And the report instance carries a machine-readable `data_completeness = partial` flag. |
| FR-019 | "the generated report PDF **includes the Brightway logo**" | No placement, size, or presence assertion that a test can bind to. | ...Then the PDF's first page header region contains the Brightway logo asset at the design-specified placement. (Placement must be pinned in eng-plan, but the acceptance assertion must name a region.) |
| NFR-002 | "throttled to a **bounded concurrency**" / "exact cap ... default set during eng-plan" | No number = not testable at acceptance. A deferred cap means NFR-002 cannot be verified. | Given generation reads against `pubsuite-attribution`, When generation runs, Then concurrent generation reads never exceed N (N pinned before acceptance), measurable under load. |
| FR-002 (note) | "generation early on a fixed weekday so a fresh report is available at the start of each business week" | "Early" and "start of business week" are not times. | Pin a concrete weekly cron time (e.g., Monday 02:00 UTC) before acceptance. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard year-end deadline gated by unsettled decisions (DEC-002/003/004/005/007/008) | Partial — working defaults declared | **FAIL** | Six open decisions are carried as "working assumptions." At least DEC-002/003 (eligibility & access scope) gate security behavior. Deadline risk is acknowledged but the mitigation is "answer them early" with no owner/date — not an actionable requirement. |
| Multi-service blast radius (6 services, KG passes disagree) | Yes — kill-switch (NFR-015) + reconcile in eng-plan | PASS (with caveat) | Mitigation exists, but scope is literally unlocked (which services own generation is undecided). This is on the critical path per the spec's own words. |
| Attribution data incompleteness / mid-processing reads | Yes — FR-013/FR-014, NFR-006 | PASS | But see CG-002: no supersession of a partial report once data finalizes. |
| UX scope creep | Yes — confirmed simple download button | PASS | Internal inconsistency: Appendix still describes frontend-mos as "report management interface," contradicting the confirmed "download button only." Cosmetic. |
| Attribution source starvation | Yes — NFR-002 | PASS (untestable) | Cap undefined; see NFR-002 above. |
| Report content sensitivity / Compliance | Assumption only (DEC-007) | **FAIL** | Treating financial-close attribution revenue data as "ordinary business data" with no masking/restricted store is an *unconfirmed* assumption on a security-relevant classification. If wrong, it's a data-handling defect found post-build. Needs Compliance sign-off tracked as a blocking gate, not a labeled assumption. |

---

## 4. Missed Edge Cases

1. **Partial report never corrected:** (see CG-002) data finalizes after a partial generation; no regeneration/supersession requirement.
2. **Quarter-boundary recalculation drift:** BR-3 recalculates "last completed quarter" weekly. On the first week after a quarter rolls over, the reporting window shifts. No FR defines behavior for the transition week or whether both quarters are ever surfaced. Finance could silently lose visibility of the just-closed quarter.
3. **Week-over-week change when prior week has no report:** FR-006 requires "change from the prior week," but FR-014 allows a week with *no report* (no-data) and FR-013 allows a *partial* prior week. What is "change from prior week" computed against when the prior week was failed/partial? Undefined — risks garbage deltas.
4. **Download link expiry mid-download:** NFR-011 sets a 15-min expiring link. No requirement covers a download in progress when the link expires, or resuming an interrupted large-PDF download.
5. **Concurrent downloads of the same report:** no limit or behavior defined for many finance users downloading simultaneously (NFR-001 mentions concurrent load but no cap/behavior on contention).
6. **Kill-switch flipped mid-generation:** FR-017 says in-flight requests complete under prior behavior, but a *generation job* mid-run is not an interactive "request." Undefined whether an in-flight generation completes, aborts, or leaves a partial artifact.
7. **403 path for a user authorized to the account but feature-disabled (FR-016 vs FR-017):** If the kill-switch disables the feature, do requests return 403 (eligibility), a "disabled" state, or the "no report" state? Overlapping gates with no defined precedence.
8. **Retention expiry while a secure link is live:** a report hits the 12-month BR-8 boundary while a user holds an unexpired download link. Does the link still resolve? Undefined.
9. **Audit-write failure during a 403:** NFR-013 alerts on audit-write failure, but it's unclear whether a denied (403) request is itself audited. Security audits usually require logging denials.
10. **PDF generation itself fails (rendering error) with data present:** FR-007/FR-008/FR-014 cover *data-source* failures and *no-data*, but not a failure in the PDF rendering/export step with valid data. No named status for "render failed."
11. **Empty-channel / zero-revenue window:** valid data exists but all channels are zero. Is that "no data" (FR-014) or a valid report? Ambiguous.
12. **Timezone of "weekly" and "quarter":** cadence and quarter boundaries have no stated timezone; Brightway's fiscal quarter may not equal a calendar quarter — critical for a *financial close*.
13. **Logo asset unavailable (FR-019):** if the Brightway logo asset can't be fetched/rendered, does generation fail or proceed without it? Undefined.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-019 are all in Gherkin fenced blocks. Notes/prose are explanatory annotations, not the requirement bodies. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"Appendix: Engineering Reference"** — a multi-paragraph narrative section with its own "Affected Services" analysis/list and prose confidence commentary, appearing *after* Section 6 and *before* the Revision Summary. This is exactly the kind of stray multi-section appendix the base rule catches; it is not a sanctioned one-line metadata trailer and not the Revision Summary. **Flag.** (The trailing `## Revision Summary` itself is EXPECTED per spec-revise and is NOT flagged.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 contains only bullet-list concept items; no attribute tables or types. |

---

## 6. Recommendations

1. **Resolve CG-001 (trigger-model contradiction) before engineering starts.** BR-6 (manual trigger) vs FR-001/002 (scheduled) vs Section 6 (manual out of scope) must be reconciled to one coherent model. This defines the feature's core behavior.
2. **Resolve CG-002 (partial-report supersession).** Add a requirement that a window regenerates and supersedes a partial report once data finalizes, with UI finality/recency indication — this is a financial-correctness issue on a close deadline.
3. **Remove or relocate the "Appendix: Engineering Reference" section** so the document contains only Sections 0–6 (plus the sanctioned Revision Summary). Move its service-map content into the Section 1 service-map note or an external eng-context artifact.
4. **Convert blocking assumptions into tracked gates, not "working assumptions."** DEC-002/003 (access/eligibility scope — security-relevant) and DEC-007 (Compliance classification of financial data) must be confirmed before acceptance, with owners and dates.
5. **Pin all deferred numbers required for acceptance:** NFR-002 concurrency cap, FR-002 cron time + timezone, and the quarter/cadence timezone (BR-3). Untestable NFRs cannot gate acceptance.
6. **Define week-over-week delta behavior** when the prior week was failed or partial (edge case #3), and define quarter-boundary transition behavior (edge case #2) — both directly affect reported financial numbers.
7. **Name the authoritative attribution source in FR-006** once the §1 service-map discrepancy is reconciled. Note the independent KG cross-check **did not re-find `k4a-mcp-server`** (which the spec's own Appendix lists from the "summary pass only"), and both passes agree on `pubsuite-attribution`, `reporting-api`, `reporting-dispatcher`, `reporting-subscriptions-worker`, `frontend-mos`, `aaas-reporting`, and `summary-report-cloud`. The spec correctly flags this as unreconciled — keep `k4a-mcp-server` marked low-confidence/"confirm involvement" and reconcile before scope lock.
8. **Add named failure statuses for PDF-render failure and logo-asset failure** (edge cases #10, #13) to complement the data-source failure taxonomy in FR-007/008/014.