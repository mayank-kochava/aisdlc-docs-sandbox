# Spec Critique: K4A Weekly Attribution Report PDF Download

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-21

## Summary

- **Critical Issues:** 2
- **High Issues:** 7
- **Medium Issues:** 6
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Unresolved attribution data source blocks a testable correctness contract.** NFR-011 demands the PDF "exactly match the values computed by the confirmed attribution data source," but there is no confirmed data source — two KG lookups disagree (`pubsuite-attribution`/`sam-report-scheduler` vs `koch-query-api`/`aaas-reporting`), and the independent cross-check re-confirms `koch-query-api` + `aaas-reporting` and does NOT re-find `pubsuite-attribution` or `sam-report-scheduler`. The single most important requirement (financial correctness feeding a year-end close) is defined against a source that does not exist yet. This is not merely a scoping-confidence note: it means NFR-011 is currently untestable and the "partial data" marker (FR-003) could mask a source-selection error that produces wrong finance numbers. | Critical | §1 KG Note, NFR-011, FR-003 | Make data-source resolution a hard pre-build gate with a named owner and date, AND add an FR requiring a reconciliation/validation check (generated figures vs. source-of-truth figures) before any PDF is marked available. Do not let "partial data" labeling substitute for figure-level validation. |
| CG-002 | **Download-link delivery vs. authorization is a potential access-control dead end / leak.** FR-009/FR-013 deliver via a "time-limited secure link," and FR-011 re-derives authorization "on each request." But the spec never states whether the secure link itself is access-controlled (re-checks entitlement when fetched) or is a bearer/capability URL that grants access to anyone holding it within the validity window. For finance-restricted data feeding a year-end close, a shareable/forwardable link that bypasses FR-011's per-request entitlement check is a data-exposure hole. The two requirements are in unreconciled tension. | Critical | FR-009, FR-011, FR-013, NFR-010 | Add an FR explicitly stating the download link resolves identity and re-checks FR-011 entitlement at fetch time (not a bare capability token), define link TTL, single-use vs. reusable, and behavior if the holder's entitlement was revoked mid-window. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001/002 | "measured under expected concurrent finance-user load" | "Expected concurrent load" is undefined — not a testable threshold. | Given N concurrent authorized finance users (N = [defined peak, e.g. 50]) When each requests an available report Then the download link is served with p95 ≤ 2s. |
| NFR-013 | "scope is not discovered mid-build" | Not a system behavior; not testable. It is a project-management aspiration, not a requirement. | Reframe as a gating checklist item with owner + due date, or remove from NFRs. |
| FR-013 | "after its validity window expires" | The validity window duration is never specified anywhere in the spec. Untestable without a number. | Given a download link issued at T0 with validity window W (W = [defined, e.g. 15 min]) When access is attempted at T0 + W + 1s Then access is denied. |
| FR-003a | "the logo appears consistently on each weekly report" | "Consistently" is not measurable (placement? size? presence?). | Given a weekly PDF is rendered Then the Brightway Retail logo appears at [defined position] on [page 1 / every page] at [defined dimensions]. |
| NFR-012 | "reflect the disabled state within 5 minutes" | Testable, but "requests already in flight at flip time complete under the prior behavior" lacks a defined max in-flight bound. | Define a max in-flight drain window so the kill-switch has a hard upper bound on full effect. |
| FR-003 | "marked as partial data" | The mechanism and placement of the "partial data" marker (watermark? header? metadata flag?) is undefined, so it cannot be tested or relied on by finance. | Given a report generated before full metric/layout confirmation Then the PDF displays a visible "PARTIAL DATA" marker at [defined location] AND the report record carries a partial=true flag. |

All FRs are correctly written in Gherkin — no prose-FR violations found.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Unresolved attribution data source (two conflicting KG results) | Partial — deferred to design review (NFR-013) | **FAIL** | Deferring is not a mitigation for a correctness-critical dependency. No figure-level validation requirement exists (see CG-001). The independent cross-check does NOT support `pubsuite-attribution`; the spec's own Appendix still lists it as the computation source — unreconciled. |
| Financial correctness / integrity (NFR-011) | Yes — no-transform rule stated | **FAIL** | NFR-011 is defined against an unconfirmed source, so it cannot currently be validated end-to-end. No reconciliation step between generated PDF figures and source values. |
| 6-service blast radius / 5-week deadline | Partial — NFR-013 names it, defers resolution | **PARTIAL** | Names the risk and assigns resolution to design review, but no fallback/descope plan if resolution slips past the hard deadline. |
| Legacy/shared scheduling infra (SAM/K4A) — double-fire | Yes — NFR-015 backward-compat guardrail + FR-007 | **PASS** | Backward-compat and non-overlap are concretely specified. |
| Shared attribution backend degradation | Yes — NFR-003 rate cap | **PASS** | Cap value deferred to design (acceptable; mechanism required). |
| Data-source unavailable at generation | Yes — FR-004/NFR-006 fail-closed | **PASS** | Well handled. |
| Security / access-control defect | Yes — kill-switch NFR-012/014 | **PASS (conditional on CG-002)** | Mitigation exists but the link-security gap (CG-002) undermines it. |
| Audit-write failure | Yes — NFR-009 async + alert | **PASS** | Good. |

---

## 4. Missed Edge Cases

1. **Capability-link forwarding / revocation mid-window:** If a finance user's access is revoked after a link is issued but before it expires, does the link still work? (See CG-002.)
2. **Quarter boundary rollover during a weekly run:** FR-002 recalculates "most recent completed quarter" each week. What happens when a weekly run straddles a quarter boundary (e.g., the run starts in Q-prev and a new quarter completes mid-run)? Which quarter wins is undefined.
3. **Empty / zero-data quarter:** If the most-recent-completed-quarter has no attribution data (new account, no activity), is that a "success with empty PDF," a "failed" status, or a distinct state? FR-004 only covers retrieval/render failure, not legitimate emptiness.
4. **Partial-data marker + year-end close:** Finance may reconcile against a report marked "partial data" without understanding what is missing. No requirement forces finance acknowledgment or defines what "partial" omits.
5. **Logo asset missing/unavailable at render:** FR-003a requires the logo, but render behavior when the branding asset is missing/corrupt is undefined — does the whole PDF fail (FR-004) or render without the logo?
6. **Download of a report whose underlying data was later recalculated:** FR-006 overwrites a week's report on re-run. If a user downloaded the old version, there is no requirement about versioning or notifying that figures changed — a finance reconciliation risk.
7. **Plan downgrade/eligibility change between generation and download:** Report generated while on standard plan; account changes plan before download. Access behavior undefined.
8. **Retention horizon rollover:** As weeks age out of the "full last quarter" window, what happens to a link already issued for a now-aged-out report? Undefined.
9. **Concurrent kill-switch flip during an in-flight download:** NFR-012 addresses generation/new requests but not a download already streaming when the flag flips.
10. **Audit log for FAILED access (403) attempts:** NFR-009 logs list/download actions; it is ambiguous whether denied/unauthorized attempts are audited — important for a finance-sensitive security posture.
11. **Time-zone of "weekly schedule" and "completed quarter":** No time zone defined for schedule triggers or quarter boundaries; affects which quarter is "completed" and reconciliation correctness.
12. **Simultaneous multi-user download load vs. link generation:** No defined behavior/limit if many finance users request the same report link simultaneously.
13. **Email path (FR-014) recipient changes / bounces:** If conditionally built, no handling for invalid recipients, bounces, or delivery failure.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-014 are all in Gherkin blocks. No prose FRs. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"Appendix: Engineering Reference"** with a multi-paragraph "Affected Services" narrative and a "Reconciliation note" — this is a stray narrative/analysis appendix beyond Sections 0–6, not a sanctioned trailer. The trailing **"## Revision Summary"** is EXPECTED and correctly NOT flagged. The violation is the Engineering Reference appendix only. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet-list only; "System-derived"/"Product-configured" source tags are annotations, not attribute tables or data types. No tables present. |

*Note on the Revision Summary:* correctly treated as the sanctioned final section per the revise template — not counted as a violation.

---

## 6. Recommendations

1. **(CG-001 — Critical) Resolve the attribution data source before build and add figure-level validation.** The independent cross-check confirms `koch-query-api` + `aaas-reporting` and does NOT re-find `pubsuite-attribution`, yet the Appendix still lists `pubsuite-attribution` as the computation source. Settle this discrepancy, name the authoritative source, and add an FR requiring generated PDF figures to be reconciled against source-of-truth values before a report is marked available.
2. **(CG-002 — Critical) Specify download-link access semantics.** State explicitly that the secure link re-checks FR-011 entitlement at fetch time, define TTL, single-use vs. reusable, and mid-window revocation behavior. Resolve the FR-009/FR-011/FR-013 tension.
3. **(Format — High) Remove or relocate the "Appendix: Engineering Reference" section** so the document contains only Sections 0–6 (plus the sanctioned Revision Summary). Fold its grounded-service context into Section 1's KG note or an external engineering doc.
4. **(High) Define concrete numbers for the validity window (FR-013), expected concurrent load (NFR-001/002), and the partial-data marker mechanism (FR-003).** These make three otherwise-untestable requirements testable.
5. **(High) Add FRs/states for empty-data quarter, quarter-boundary rollover timing, and missing-logo render behavior** (Edge cases 2, 3, 5).
6. **(Medium) Define versioning/notification behavior for FR-006 overwrites** so finance is not silently reconciling stale downloaded figures (Edge case 6).
7. **(Medium) Clarify audit coverage of denied (403) access attempts** in NFR-009 (Edge case 10).
8. **(Medium) Specify time zone for weekly schedule and quarter-boundary determination** (Edge case 11).
9. **(Medium) Add a descope/fallback plan tied to the 5-week deadline** if data-source/scheduling-ownership resolution slips (NFR-013 risk).