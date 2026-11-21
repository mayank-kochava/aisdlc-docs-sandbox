# Spec Critique: K4A Weekly Attribution Report PDF Download

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
| CG-001 | The attribution **data source is unresolved** — the spec openly admits two KG lookups disagree (`pubsuite-attribution`/`sam-report-scheduler` vs `koch-query-api`/`aaas-reporting`). NFR-011 ("figures must exactly match the confirmed attribution data source") is unverifiable because there is no confirmed source. This blocks implementation: engineering cannot build generation (FR-001/FR-002) or guarantee financial correctness against an undetermined backend. The spec defers this to "engineering design review," but for a finance-critical, 5-week-deadline feature, shipping an FR set that cannot be built without first resolving the source is a build blocker, not a design-review footnote. | Critical | 1, 5 (NFR-011), FR-001/002 | Resolve the authoritative attribution data source before engineering starts, or explicitly make source-resolution a gating Phase-0 spike with its own exit criteria. The independent cross-check confirms the ambiguity is real: it re-found neither `pubsuite-attribution` nor `sam-report-scheduler` and instead named `koch-query-api` and `aaas-reporting`. |
| CG-002 | **Download-link validity window is never specified.** FR-013 and NFR-010 require a "time-limited secure link" but no FR defines the TTL, nor the behavior when a user clicks an *expired* link, nor whether a new link can be re-minted. The user flow ends at "User receives/opens the PDF" (node Q) with no expired-link branch. A finance user who saves the link and returns the next day hits an undefined dead end for finance-restricted data. | Critical | FR-013, NFR-010, Flow node P→Q | Define the TTL value and add an FR: expired link → access denied + mechanism to re-request a fresh link from the authorized interface. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-013 | "delivered via a time-limited secure link ... ceases to grant access after its validity window expires" | "Validity window" has no value and no expired-access behavior — untestable. | Given a download link issued at time T with TTL of N minutes, When the link is used after T+N, Then the system returns access-denied and delivers no PDF, And the user is offered a path to request a fresh link from the authorized interface. |
| NFR-001/002 | "expected concurrent finance-user load" | "Expected ... load" is undefined — no concurrency number, so p95 ≤ 2s is not a testable target. | Given N concurrent authorized finance users (N = [define]), When they request listing/download, Then p95 response ≤ 2s. |
| NFR-003 | "honor an explicit rate cap ... The specific cap is an engineering decision" | No cap value = no test. Acceptable to defer to design, but flag that it is currently untestable. | Given generation reads the attribution backend, When the run executes, Then request rate against the backend stays ≤ [cap] req/s. |
| FR-003 | "the standard K4A attribution report metric set" | "Standard set" is not enumerated anywhere in the spec; a tester cannot assert report contents match. | Given a generated report, When content is assembled, Then it contains exactly the metrics defined in [linked metric-set definition]. |
| NFR-009 | Minimum retention "12 months. [DECISION REQUIRED]" | The requirement states 12 months but simultaneously marks it undecided — ambiguous whether 12 is binding. | Resolve: state the retention value as binding or remove the number until decided. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Unresolved attribution data source (KG disagreement) | Partial — flagged + deferred to design review (NFR-013) | FAIL | For a finance-correctness feature, "settle during design review" is not a mitigation that lets FR-001/002/NFR-011 be built or tested now. See CG-001. |
| HIGH risk / 5-week deadline / 6-service blast radius | Yes — NFR-013 (settle scope at design review), kill-switch NFR-012, rollback NFR-014 | PASS | Mitigations are concrete and actionable. |
| Financial data sensitivity / year-end close | Yes — NFR-010, NFR-011, NFR-009 audit | PASS (with caveat) | NFR-011 is a strong mitigation but non-verifiable until CG-001 resolved. |
| Scheduling ownership split (dispatcher/subscriptions/scheduler) | Yes — NFR-013, NFR-015 backward-compat | PASS | Reasonably handled; backward-compat guardrail is specific. |
| Shared attribution backend contention | Yes — NFR-003 rate cap | PASS (value TBD) | Mitigation exists; cap value deferred. |
| Audit-write failure | Yes — NFR-009 (async + alert on failure) | PASS | Well specified. |
| Expired/leaked download link | No specific expired-link handling FR | FAIL | TTL and expired-link behavior undefined (CG-002). |

---

## 4. Missed Edge Cases

1. **Expired download link reuse:** No behavior defined when an authorized user clicks a link after the validity window (CG-002). Flow node Q has no expired branch.
2. **Download-link leakage / replay:** If a secure link is forwarded to an unauthorized party before expiry, does the link itself re-check authorization, or is the time-limited token alone sufficient? FR-011 re-derives auth "on each request" for list/download requests, but it is unclear whether the final link-fetch re-validates the actor. Ambiguous for finance-restricted data.
3. **Retention horizon aging out a report mid-session:** A user views the list (FR-008), a report ages past the retention horizon, then the user clicks download. No defined behavior for list-then-gone.
4. **Partial PDF render / empty-quarter data:** FR-004 covers render *failure*, but not the case where the attribution source returns successfully with **zero rows** for the last quarter. Is an empty-but-valid report "available," or treated as failed? Undefined — material for finance reconciliation.
5. **Clock/timezone boundary for "last quarter":** "Last quarter" is computed from generation run date (Info Req: Report Covered Period) but no timezone or fiscal-vs-calendar-quarter definition is given. Brightway's *year-end close* strongly implies fiscal quarters may differ from calendar quarters — a silent mismatch here corrupts the reconciliation the feature exists to serve.
6. **Retry exhaustion straddling the next week's run:** NFR-005 bounds retries to 3 with backoff; if retries run long and the next weekly schedule fires, FR-007's concurrency guard is keyed on account+week, not across weeks. No defined interaction between a still-retrying week-N run and a starting week-N+1 run on the same backend.
7. **Mid-generation feature-flag flip:** NFR-012 says in-flight requests complete under prior behavior, but does not define what happens to an in-flight *generation* job when the kill-switch flips — does it finish and mark available, or abort?
8. **Concurrent idempotent overwrite during active download:** FR-006 overwrites a week's report; if a user is mid-download of the old version when an overwrite completes, FR-009's "contents match the report generated for that week" could be violated. No defined behavior.
9. **Email delivery failure (if FR-014 in scope):** FR-014 defines the happy path for email delivery but no failure/bounce/retry behavior for undeliverable finance recipients.
10. **Authorization source *slow* (not fully unreachable):** FR-012/NFR-008 fail closed when the source is *unreachable*; timeout/partial-degradation (slow but responding) behavior vs the NFR-001 2s budget is undefined.
11. **Account loses standard-plan eligibility between weeks:** FR-001 gates generation on standard plan; no defined behavior for already-generated reports if the account is downgraded (still downloadable or revoked?).
12. **No "generation never ran" vs "failed" distinction in UI:** FR-010 lumps "not yet generated" and "failed" into one "report not available" state, but Info Req Report Availability Status distinguishes available/failed/not-yet-generated. Inconsistency between FR-010 and the data model.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-014 are all in Gherkin blocks. Explanatory `> Note:` annotations are acceptable supplements, not prose FRs. |
| Document contains only Sections 0–6 | FAIL | The document contains a trailing **"## Appendix: Engineering Reference"** section — a multi-item narrative appendix (Affected Services list plus a prose Reconciliation note with findings). This is NOT a sanctioned Revision Summary nor a one-line labeled metadata blockquote trailer; it is exactly the "stray KG cross-check / engineering appendix with its own narrative findings" shape the base rule is written to catch. It belongs outside Sections 0–6. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list; italicized `*System-derived*`/`*Product-configured*` provenance labels are descriptive annotations, not attribute tables or type declarations. |

---

## 6. Recommendations

1. **Resolve the attribution data source (CG-001)** before engineering starts, or make source-resolution a gated Phase-0 spike with explicit exit criteria. NFR-011's financial-correctness guarantee is unverifiable until this is settled — this is the single highest risk to the year-end-close deadline.
2. **Define the download-link TTL and expired-link behavior (CG-002)** — add an FR covering expiry, re-minting, and whether the link re-validates authorization on fetch (edge cases 1 & 2).
3. **Move the Engineering Reference appendix out of Sections 0–6** to restore format compliance, or fold its grounded content into Section 1 / NFR-013 where scoping ambiguity already lives.
4. **Pin "last quarter" semantics** — fiscal vs calendar quarter and timezone — given this feeds a year-end close (edge case 5).
5. **Define empty/zero-row quarter handling** (edge case 4) and reconcile FR-010's two-state "not available" with the three-state Availability Status data model (edge case 12).
6. **Quantify the vague NFRs** — concurrency number for NFR-001/002, rate cap for NFR-003, and resolve the contradictory "12 months [DECISION REQUIRED]" in NFR-009.
7. **Specify cross-week and overwrite-during-download interactions** (edge cases 6 & 8) and in-flight generation behavior on kill-switch flip (edge case 7).