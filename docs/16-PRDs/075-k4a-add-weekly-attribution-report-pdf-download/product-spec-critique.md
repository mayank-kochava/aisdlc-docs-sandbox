# Spec Critique: Brightway Retail Weekly Attribution Report (PDF)

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
| CG-001 | DEC-001 (last-quarter semantics) is unresolved but FR-002 ships a provisional "most recently completed calendar quarter" default. For a year-end-close reconciliation use case, calendar-quarter vs rolling-90-day produces *materially different numbers finance reconciles against*. Shipping the wrong default silently corrupts the very deliverable's purpose. Not data loss, but a correctness landmine. | High | §1 / FR-002 | Treat DEC-001 as a hard gate before generation code merges; do not let the provisional default reach production for the year-end-close customer without explicit finance sign-off. |
| CG-002 | FR-005 / DEC-003 conflict with FR-004. FR-004 says a re-run **overwrites the existing weekly report in place**; FR-005 provisionally guarantees **repeated downloads in the same week return identical data**. If generation legitimately re-runs mid-week (retry, late data), the snapshot promised by FR-005 is broken by the overwrite in FR-004. These two provisional rules are mutually inconsistent as written. | High | FR-004 / FR-005 | Define whether an overwrite is permitted to change downloaded content within a week. Either (a) overwrite only replaces *failed/empty* reports, never a good snapshot, or (b) FR-005's snapshot guarantee is dropped. Resolve before engineering. |
| CG-003 | Push model (FR-008/FR-009) delivers via a time-limited secure link, but no requirement defines behavior when the recipient clicks an **expired** link in their inbox days later (finance reads email late). FR-009 says "expired link does not return the report" but there is no re-request / regeneration / "request a fresh link" path — dead end for a legitimate user. | High | FR-008/FR-009 | Add an FR for expired-link recovery: a signed-in authorized user can obtain a fresh link or download in-session even after the original link expires. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "completes within a bounded generation window such that the report is marked available before the finance team's expected weekly access time" | "Bounded window" and "expected weekly access time" are unquantified — not testable. | Given a weekly generation cycle starts, When generation runs, Then the report is marked available no later than [specific clock time, e.g. 06:00 UTC Monday] / within [N] minutes of cycle start, measured at p95. |
| NFR-001 | "p95 latency target of 3 seconds ... measured under expected concurrent finance-user load" | p95/3s is testable, but "expected concurrent finance-user load" is undefined (how many concurrent users?). | Given [N] concurrent authorized finance users, When each triggers download of a pre-generated report, Then p95 response-to-first-byte ≤ 3s. Define N. |
| FR-015 | "the metrics, dimensions, grouping, and ordering follow the defined report content specification" | References a "defined report content specification" that does not exist in this doc (DEC-005 unresolved). Untestable — no fixture to assert against. | Blocked on DEC-005. Once content is defined, assert: Given a generated PDF, Then it contains exactly [enumerated metrics] grouped by [dimension] ordered by [key]. |
| NFR-013 | "a shorter or longer period for the PDFs themselves depends on whether finance has an archival/compliance obligation" | PDF retention period is unspecified — cannot write a retention/expiry test. | Blocked on DEC-006. Define a concrete minimum PDF retention (e.g. 12 months) to make expiry assertable. |

All FRs are otherwise written in Gherkin (except FR-018 — see §5).

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Shared attribution query source contention | Yes — NFR-003 (rate cap), NFR-005 (bounded retry + Retry-After) | PASS | Well specified. |
| Downstream transient failure | Yes — NFR-005 (3-attempt bounded backoff), FR-003 (failure recorded/alerted) | PASS | Good. |
| Authorization source unreachable | Yes — FR-014 + NFR-007 (fail-closed) | PASS | Correct posture. |
| Serving report to ineligible/unauthorized party | Yes — FR-011, FR-013, NFR-009 (server-side, re-derived) | PASS | Strong. |
| Legacy/shared endpoint regression (extends existing reporting services) | Partial — NFR-016 asserts "must remain unchanged" | FAIL | NFR-016 states the *requirement* but defines no mitigation: no regression-test gate, no contract test, no requirement that existing consumers' outputs are verified unchanged before rollout. "Must remain unchanged" is an assertion, not a mitigation. Add a verification requirement. |
| HIGH-risk, multi-service rollout | Yes — NFR-014 (kill-switch), NFR-015 (rollback) | PASS | Kill-switch + rollback defined. |
| last-quarter semantics ambiguity (DEC-001) | No mitigation, only provisional default | FAIL | Risk acknowledged but the mitigation is "pick a default and hope." For a finance reconciliation deliverable this needs an explicit decision gate, not a silent default. |
| PII in logs | Yes — NFR-011, FR-018 | PASS | Present (FR-018 has format issue — see §5). |

---

## 4. Missed Edge Cases

1. **Empty-but-valid quarter:** The account has zero attribution data in the last-quarter window. Is this a successful empty report, or treated as "not available"? FR-003 forbids marking an *empty* report available on failure, but a legitimately empty quarter is not a failure. Undefined.
2. **Partial attribution data / late-arriving data:** If attribution data for the quarter is still settling when generation runs, the PDF may be incomplete. No requirement addresses data-completeness/finality before generation.
3. **Quarter-boundary race:** Weekly generation running exactly on a quarter boundary — which quarter is "last quarter" at the moment of execution? Timezone of the boundary is unspecified (NFR uses UTC elsewhere, business rules don't).
4. **FR-004 overwrite of a report already downloaded:** If finance already downloaded week N's report and an overwrite changes it, finance now holds a reconciliation artifact that no longer matches the system. No requirement about notifying or versioning. (See CG-002.)
5. **Mid-download kill-switch flip during push delivery:** NFR-014 covers in-flight *downloads*, but not an in-flight *scheduled push* at the moment of the flip. Undefined.
6. **Plan downgrade mid-week:** Account downgrades off standard plan after a report is generated but before download. FR-013 gates on current plan server-side — so finance loses access to an already-generated report. Intended? Undefined.
7. **User removed from finance team between generation and download:** FR-010 re-derives eligibility per request (good), but no edge case covers a pending/expired secure link issued while the user was still authorized.
8. **Concurrent download + overwrite:** User downloads while FR-004 idempotent re-run overwrites in place. Is the download atomic against the overwrite, or can it serve a torn/partial PDF?
9. **Secure-link validity window length:** FR-009/NFR-010 require expiry but never state the window duration (DEC-007). Untestable and a potential security gap if left to an implementer.
10. **Audit-write failure under fail-closed:** FR-017 says audit writes are non-blocking and alert on failure; FR-014/NFR-007 say fail-closed on auth. What is the behavior if the *audit* subsystem is down but auth is fine — serve (audit lost) or deny? Interaction undefined.
11. **Multiple quarters requested for reconciliation:** Year-end close often needs more than one quarter. Scope says last-quarter only (correctly out of scope), but there is no acknowledgement of how finance reconciles full-year from single-quarter weekly snapshots. Product gap worth flagging.
12. **Duplicate alert storm:** NFR-004 alerts on *any* individual generation failure AND on >1% sustained. For a single-account (Brightway) rollout, 1 failure = 100% error rate — both alerts fire for the same event. Alert-dedup behavior undefined.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | FAIL | **FR-018 — "No PII in logs"** has a title only and NO Gherkin body (no Given/When/Then block). It is referenced by NFR-011 but is not an implementable/testable requirement as written. Add a Gherkin body. |
| Document contains only Sections 0–6 | FAIL | The document contains an **"## Appendix: Engineering Reference"** after Section 6, containing an "Affected Services" narrative and a lower-confidence KG cross-check paragraph. This is multi-item narrative appendix content beyond Sections 0–6, not a sanctioned trailing Revision Summary or one-line labeled metadata blockquote. It should be removed or relocated out of the spec body. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely a bullet list; no attribute tables or type declarations present. |

---

## Implementation-Reality Check (vs. Independent KG Cross-Check)

- The spec's own "Note on affected-services evidence" (§1) and Appendix **already disclose** the discrepancies the independent KG lookup surfaces: `reporting-dispatcher` and `reporting-subscriptions-worker` at low confidence, and two additional services (`summary-report-cloud`, `koch-query-api`) flagged for discovery. This matches the fresh cross-check (all those services at low confidence; `koch-query-api` as attribution data source). **No contradiction** — the spec handles this correctly by recording it in §1 for reconciliation rather than asserting a settled list.
- One thing to verify in discovery: the spec asserts the last-quarter filter is a report-layer concern (FR-002) while the KG evidence suggests `koch-query-api` "may be where the last-quarter filter must be applied." This is a *potential* data-flow mismatch worth confirming — but it is flagged as low confidence and the spec already notes it, so it is not raised as a blocking contradiction.

---

## 6. Recommendations

1. **Resolve CG-002 (FR-004 overwrite vs FR-005 snapshot conflict)** — these two provisional rules are mutually inconsistent and will produce reconciliation discrepancies. Highest-priority correctness fix.
2. **Gate DEC-001 (last-quarter semantics) before generation code merges** — do not let the calendar-quarter default silently ship for a year-end-close finance deliverable (CG-001).
3. **Add an expired-secure-link recovery requirement** (CG-003) so late-reading finance recipients are not dead-ended.
4. **Fix FR-018** — give it a Gherkin body so "No PII in logs" is testable.
5. **Relocate/remove the "Appendix: Engineering Reference"** so the document conforms to Sections 0–6.
6. **Add a regression-verification mitigation to NFR-016** (contract/regression tests proving existing reporting consumers are unchanged) rather than a bare "must remain unchanged" assertion.
7. **Quantify NFR-002** (generation window → concrete availability time) and **NFR-001's "expected concurrent load"** so both are testable.
8. **Define empty-but-valid-quarter behavior and data-finality** before generation (edge cases 1–3).
9. **Specify DEC-007 secure-link validity window** and resolve the audit-down-vs-fail-closed interaction (edge case 10).
10. **Define alert-dedup for the single-account case** (edge case 12) to avoid duplicate NFR-004 alerts.