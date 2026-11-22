# Spec Critique: Brightway Retail Weekly Attribution Report (PDF)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 0
- **High Issues:** 5
- **Medium Issues:** 7
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **No defined "no data for the quarter" path.** FR-002 excludes data outside the window, but nothing defines behavior when the most-recent-completed-quarter has *zero* attribution rows (new account, data gap, upstream lag). Does generation succeed with an empty table, mark a failure, or produce a misleading zero report? Finance reconciliation with a silently-empty table is a correctness risk. | High | §3 FR-002 / §5 NFR-008 | Add an FR: Given the resolved window has no attribution data, Then the system produces a report that explicitly indicates "no data for period" OR raises a distinct named condition — not a blank table that reads as $0 attributed. |
| CG-002 | **Stale-report ambiguity against the availability gate.** FR-003 preserves the "prior report" on failure, but BR-5/NFR-003 require a *fresh* report before year-end close. If every cycle fails for weeks, the console shows a stale prior report with no visible staleness indicator to the finance user. They may reconcile against an out-of-date quarter unknowingly. | High | §3 FR-003 / §2 flow / BR-5 | Add requirement: the download/empty-state UI must surface the generation date/period of the available report and flag when the latest cycle failed so a stale artifact is never mistaken for current. |
| CG-003 | **Quarter-boundary race is undefined.** FR-002 resolves "most recent completed quarter as of that run." A weekly run executing within hours of a quarter close (or during late-arriving attribution data) may resolve a window whose data is still settling upstream, producing numbers that change on the next run. No rule governs attribution-data settlement lag vs. window selection. | High | §3 FR-002 | Define a settlement/lookback rule: Given a quarter closed < N days before the run, Then selection either waits for upstream finalization or marks the report provisional. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "operate under an explicit rate cap… The specific cap is an engineering default to be set during planning." | No numeric threshold = untestable acceptance criterion. "Explicit cap" with no value cannot be verified. | Given the weekly generation job reads the attribution source, When it issues queries, Then the request rate stays ≤ [X] req/s and existing-traffic p95 latency does not increase by > [Y]%. (Fill X/Y before test design.) |
| NFR-001 | "p95 ≤ 5s for serving an already-generated report" under "expected concurrent finance-user load" | p95 target is measurable, but "expected concurrent load" is undefined — the test harness has no concurrency number to reproduce. | Given [N] concurrent entitled finance users requesting download, When the report is already generated, Then p95 response ≤ 5s. (Define N.) |
| NFR-009 | "chosen to complete before the finance team's typical workday" | "typical workday" is not a testable clock time or timezone. | Given the weekly cycle runs, Then an available report exists by [HH:MM] [TZ] on the scheduled day. |
| NFR-005 | "time-limited secure link… validity window is an engineering default" | No validity duration = cannot test link expiry. | Given a download link is issued, When [T] minutes elapse, Then the link returns access-denied. (Define T.) |
| BR-5 | "target the first successful weekly report… no later than 4 weeks from kickoff" (working assumption, close date unconfirmed) | Acceptance depends on an unconfirmed calendar date; cannot sign off completion against an unknown deadline. | Blocker-adjacent: confirm the actual close date, then: Given kickoff date D, Then first successful report available ≤ D+4 weeks AND ≤ (close date − 1 week). |

Note: All ten FR-001–FR-010 are correctly in Gherkin — no prose-FR violations found. The vagueness above is concentrated in the NFRs, which are not required to be Gherkin but still need measurable thresholds.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (5 wks / year-end close) | Partial — scope held to MVP, kill-switch, buffer | PARTIAL | Mitigation hinges on BR-5's **unconfirmed** close date. A risk mitigated by an assumption is not fully mitigated. Confirm the date to close this. |
| Multi-service blast radius (5+ services) | Yes — NFR-004/008/009/010 | PASS | Server-side enforcement, bounded retries, idempotency, kill-switch are concrete and testable. |
| Unconfirmed UX scope | Yes — resolved, Out of Scope list | PASS | Clearly resolved and fenced in Section 6. |
| Recurring vs. one-time ambiguity | Yes — BR-2 | PASS | Resolved as time-bounded weekly through close. |
| Service-ownership ambiguity (report-content data owner) | Flagged, not resolved | PARTIAL (acceptable) | Correctly deferred to planning and not silently decided — appropriate for a spec, but it remains an open dependency that gates FR-001/FR-002 data sourcing. |
| Legacy/shared attribution source degradation | Yes — NFR-002 rate cap + NFR-008 backoff/Retry-After | PASS (once NFR-002 cap is numeric) | Mechanism present; only the threshold value is missing (see Testability). |
| Entitlement source unreachable | Yes — NFR-004 "fail closed" | PASS | Good: fail-closed explicitly stated. |

---

## 4. Missed Edge Cases

1. **Empty-quarter data:** No defined behavior when the window has zero attribution rows (see CG-001).
2. **Stale-but-available report:** Repeated generation failures leave a stale prior report with no staleness signal (see CG-002).
3. **Partial-channel data:** FR-006 reports "by channel," but behavior when some channels are missing/late upstream is undefined — the table may under-report revenue without flagging it. Finance totals could be wrong.
4. **Concurrent download during an in-flight regeneration swap:** FR-005 retains a single report per cycle; nothing defines what a user downloading at the exact moment the new PDF replaces the old one receives (old, new, or a torn read / 404).
5. **Link expiry mid-download:** NFR-005 time-limited link — behavior if the validity window expires during an in-progress large download is unspecified.
6. **Timezone of "quarter" and "weekly run":** No timezone is pinned for quarter boundaries or the weekly schedule. Brightway's fiscal/calendar quarter vs. system UTC could misalign the window for a finance (year-end-close) use case.
7. **Audit-write failure path:** NFR-007 says audit failure raises an alert and must not block the operation — but does the download still succeed if its audit entry failed? For a "sensitive finance export" with a 2-year audit mandate, a successful download with no audit record may be non-compliant. Behavior should be explicitly chosen.
8. **Logo asset unavailable:** FR-008a mandates the Brightway logo on the PDF (confirmed requirement). No path defines whether generation fails, retries, or proceeds without the logo if the brand asset is missing/corrupt.
9. **Kill-switch flipped mid-cycle vs. download:** NFR-010 handles in-flight generation, but says nothing about a download request for an already-available report after the feature is disabled — is the existing report still downloadable or access-denied?
10. **Entitlement revoked between generation and download:** A report generated while entitled, then account entitlement removed — NFR-004 fail-closed implies denial on download, but this interaction with an already-generated artifact isn't stated explicitly.
11. **Retry-once vs. NFR-008 backoff interaction:** FR-003 says "retries the generation once"; NFR-008 also describes "bounded retry with exponential backoff" for transient attribution-source calls. The relationship between the single cycle-level retry and the sub-call-level backoff retries is ambiguous and could be read as conflicting retry policies.
12. **Weekly cycle after year-end close / cadence stop:** BR-2 bounds the cadence to year-end close, but no requirement defines *how* the schedule stops (manual disable, configured end date) or what the console shows once the cadence ends.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-010 (incl. FR-008a) are all in valid Gherkin blocks; no prose FRs. |
| Document contains only Sections 0–6 | FAIL | An `## Appendix: Engineering Reference` (with "Affected Services" + a multi-paragraph "Cross-check discrepancy" narrative) sits between Section 6 and the Revision Summary. This is NOT a sanctioned trailer: it is a multi-paragraph narrative analysis appendix with its own findings/recommendations prose — exactly the "stray KG cross-check appendix" the base rule is written to catch. The trailing `## Revision Summary` itself is EXPECTED and is NOT flagged. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list of business concepts; no attribute tables or field types. (Note: Section 5's risk-mitigation and Section 6 tables are permitted in those sections.) |

**Detail on the FAIL:** The `## Appendix: Engineering Reference` block is a full multi-paragraph narrative section (service list + a prose "Cross-check discrepancy (for planning)" subsection with its own reasoning and recommendations), not a short one-line `> **Label:** ...` metadata blockquote and not the Revision Summary. It must be removed or relocated out of the spec body to comply with the Sections 0–6 constraint. The service-ownership ambiguity is already appropriately captured in Section 1's service-ownership note and Section 4, so the appendix is redundant as well as out-of-structure.

---

## 6. Recommendations

1. **Remove/relocate the `## Appendix: Engineering Reference` section** (Format FAIL) — its content is already covered by the Section 1 service-ownership note; keep the document to Sections 0–6 plus the sanctioned Revision Summary.
2. **Confirm the actual year-end-close date** and convert BR-5 from a working assumption to a committed, testable deadline — this unblocks the deadline-risk mitigation and the first-report acceptance criterion.
3. **Add an empty/partial-data FR** (CG-001, edge cases 1 & 3): define behavior for zero-row and missing-channel quarters so finance never reconciles against a silently-empty or under-reported table.
4. **Add a staleness-visibility requirement** (CG-002): surface the available report's generation date/period and flag when the latest cycle failed, so a stale prior report is never mistaken for current.
5. **Define quarter-boundary / attribution-settlement rules and pin a timezone** (CG-003, edge cases 6): govern window selection relative to upstream data finalization.
6. **Backfill numeric thresholds** for NFR-001 (concurrency N), NFR-002 (rate cap), NFR-005 (link TTL), and NFR-009 (concrete completion clock/TZ) so each becomes testable.
7. **Resolve the audit-vs-download and entitlement-revocation interactions** (edge cases 7, 10): explicitly state whether a download proceeds when its audit write fails, given the 2-year compliance mandate.
8. **Clarify the retry model** (edge case 11): reconcile FR-003's "retry once" with NFR-008's "exponential backoff" so engineers implement one coherent policy.
9. **Specify logo-asset-unavailable and kill-switch-vs-existing-download behaviors** (edge cases 8, 9) and how the weekly cadence terminates at year-end close (edge case 12).