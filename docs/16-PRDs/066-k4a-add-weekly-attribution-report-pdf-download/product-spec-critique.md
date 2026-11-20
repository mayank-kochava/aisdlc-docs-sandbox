# Spec Critique: K4A Weekly Attribution Report PDF Download

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-20

## Summary

- **Critical Issues:** 3
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Forward-only rollback leaves corrupted/incorrect artifacts downloadable.** NFR-020 names "report delivery of incorrect/corrupted data" as a rollback trigger, but explicitly states disabling is "forward-only — already-generated reports are not retroactively withdrawn." This is self-contradictory: the kill-switch cannot remediate the exact data-integrity scenario it lists as a trigger. A finance user can still download an already-issued (or already-link-regenerated via FR-005a) corrupted PDF after the kill-switch is flipped, and reconcile their books against bad data at year-end. This is a data-integrity failure with finance consequences. | Critical | 5 (NFR-020), 3 (FR-017) | Add an FR for artifact invalidation/recall: when a period's report is marked corrupted/invalid, downloads of that period (including re-issued FR-005a links) must be blocked server-side independent of the global kill-switch. The Revision Summary itself marks this as CG-002 "Deferred" — it remains unresolved and blocking. |
| CG-002 | **DEC-004 (historical window) is unresolved yet FRs are written against it.** FR-002 and Section 1 defer whether a report is most-recent-week-only vs. rolling multi-week "to engineering discretion." This is a core report-content decision with reconciliation implications for finance (a rolling window changes what "the weekly report" even means for year-end close). Delegating it to engineering means two implementations could both pass the FRs while producing materially different financial artifacts. | Critical | 1, 3 (FR-002) | Resolve DEC-004 as a product/finance decision, not engineering discretion. Finance reconciliation requirements determine the window; this cannot be deferred silently. Marked "Deferred" in Revision Summary — still open and blocking. |
| CG-003 | **Monday 9:00am deadline has no defined timezone/DST basis.** Business Rules and NFR-004 fix a hard "Monday 9:00am (account's local/business time)" availability deadline, but "local/business time" is undefined for an account that may span regions, and DST transitions are unhandled. On a DST-shift week, "9:00am local" is ambiguous and the generation-throughput SLO (NFR-004) cannot be tested. A missed deadline directly defeats the stated primary value. | Critical | 1, 5 (NFR-004) | Define the authoritative timezone (e.g., account-configured IANA zone), the DST handling rule, and the fallback when no account zone is configured. Marked CG-004 "Deferred" — still open and blocking. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "...should respond within a p95 latency target... Engineering may refine the exact threshold against real artifact sizes." | "Should" + "may refine" makes the p95 ≤ 2s non-binding and therefore untestable as an acceptance gate. | `Given a generated report is available And expected concurrent finance-user load is applied When the user requests download Then the download-initiation response p95 MUST be ≤ 2s measured over [N] representative requests` — with N and the load profile fixed, not "may refine." |
| NFR-004 | "...complete all enrolled accounts' reports within the weekly window with margin..." | "With margin" is unquantified; "the weekly window" start is undefined. Not testable. | `Given all enrolled accounts' generation runs start at [defined trigger time] When generation completes Then every enrolled account's report MUST be available no later than [N] minutes before Monday 09:00 in the account's authoritative timezone` |
| FR-017 / NFR-020 | "...within an established propagation window" / "5-minute default" | "Established propagation window" references an unstated external convention; the 5-minute figure is tagged a "working default," so it is non-binding. | `Given the kill-switch is flipped to disabled at T Then all new requests received after T+5min MUST observe the disabled state` — bind the 5 minutes as an acceptance criterion, not a default. |
| NFR-013 / FR-019 | "...retained for a minimum of 12 months within the established platform range; a longer period applies only if..." | "Established platform range" is unstated; the retention floor is conditionally variable, making it ambiguous to test. | `Given an audit record written at T Then it MUST remain readable and unmodified through T+12 months And MUST NOT be editable or deletable by any role or code path` — and state the max range explicitly. |
| NFR-005 / NFR-006 | "...exceeds 1% for 5 minutes, sustained" / "exceeds 80% for 5 minutes, sustained" | These are actually testable — included only to note they are the *good* model NFR-001/004 should follow. | No rewrite needed; use as the template for binding the vague ones above. |

*All FRs (FR-001–FR-019) are in valid Gherkin. The testability issues above are concentrated in the NFR prose, which is acceptable format-wise but weakens the acceptance gate.*

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard year-end deadline (~5 weeks) — NFR-017 | Partial | **FAIL** | Mitigation is "budget discovery time explicitly" + "minimal MVP scope." That is a process intention, not a requirement with a testable condition. No requirement defines what happens to the finance team's year-end close if the deadline is MISSED (no manual-fallback requirement). The stated problem says finance "cannot complete its year-end attribution review without manual intervention" today — there is no fallback FR if the feature slips. |
| Multi-service blast radius / unsettled scope — NFR-018 | Partial | **FAIL** | Mitigation defers blast-radius confirmation to discovery (acceptable) and points to the kill-switch (FR-017). But the kill-switch is global on/off; there is no requirement addressing the specific scheduling-ownership overlap (reporting-dispatcher vs reporting-subscriptions-worker) or PDF-generation overlap (reporting-api vs reporting-subscriptions-worker) that could cause duplicate or missed generation. FR-004/FR-012 mitigate duplicate generation within one owner but not cross-service double-dispatch. |
| Finance/billing data sensitivity — NFR-019 | Yes | **PASS** | Well-covered: FR-009/010 (server-side auth, fail-closed), FR-016/NFR-011 (log hygiene), FR-006/NFR-012 (secure expiring delivery), NFR-010 (account isolation), FR-018/019 (immutable audit). This is the strongest-mitigated risk. |
| Corrupted/incorrect data in a delivered report | No | **FAIL** | Named as a rollback trigger (NFR-020) but no mitigation exists (see CG-001). The kill-switch is forward-only and cannot withdraw the bad artifact. |
| Attribution source unavailable at generation | Yes | **PASS** | FR-013/NFR-014 (bounded retry + Retry-After), FR-015 (not-ready marking + alert), FR-010 (fail closed). Adequate. |

---

## 4. Missed Edge Cases

1. **Corrupted-artifact recall:** No flow to block download of an already-issued report found to be incorrect (see CG-001). FR-005a regeneration would re-issue a link to the *same* corrupted artifact.
2. **Partial/incomplete attribution data:** FR-003 assumes the source is "available," but nothing covers the source returning a *partial* dataset (e.g., some channels missing). Is a partial report generated, or marked not-ready? Undefined.
3. **Zero-activity week:** No requirement for a week with no spend/installs/revenue. Is an empty-but-valid PDF produced, or treated as "no report available" (conflating FR-007's meaning)?
4. **Mid-week enrollment:** An account enrolled partway through a week — does it get a partial-week report, wait for the next full week, or error? Undefined (FR-001 enrollment timing unspecified).
5. **Account unenrolled/deactivated between generation and download:** User holds a valid link; account becomes unauthorized. FR-009 re-derives auth per request (good), but the interaction with FR-005a regeneration for a now-deactivated account is not explicitly stated.
6. **Concurrent kill-switch flip during in-flight download:** FR-017 says in-flight downloads complete under prior behavior — but a download whose link is regenerated (FR-005a) *after* the flip: is that a "new request" that should be blocked? Ambiguous.
7. **Clock skew / scheduler misfire:** FR-004 prevents concurrent same-period runs, but a schedule that fires twice with clock skew across periods (or fires late past the Monday 9:00am deadline) has no defined behavior.
8. **Link regeneration abuse / rate limiting:** FR-005a lets a user regenerate links indefinitely. NFR-002 rate-limits the read path, but there is no explicit cap on link regeneration specifically — a potential vector to generate many live 7-day links.
9. **Timezone-boundary report labelling:** FR-008 labels the period, but with the undefined timezone (CG-003), the period boundaries printed on the PDF may not match the finance team's expectation of "the week."
10. **DEC-004 downstream ambiguity:** If rolling-window is chosen, idempotency key (FR-012, "account + period") semantics for an overlapping multi-week window are undefined.
11. **Audit-write async failure then download succeeds:** FR-018 writes audit async and alerts on failure — but the download already succeeded. For finance-sensitive access, is a successful download with a *failed* audit write acceptable, or should it fail closed? Policy undefined.
12. **Expired link reused after account's plan changes:** FR-011 says no plan gating today, but the Information Requirements still track "Account & Plan Entitlement" and FR-005 account-scopes — behavior if the account later moves to a plan where the feature *is* gated is undefined (future-proofing gap).
13. **Retry-After larger than the weekly window (NFR-014/FR-013):** If the source returns a very long Retry-After, honoring it could push generation past the Monday 9:00am deadline with no override. No interaction defined between retry honoring and the hard deadline.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-019 (incl. FR-005a) are all valid Gherkin. No prose FRs found. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present. The trailing **"## Appendix: Engineering Reference"** is a multi-section narrative KG-grounding appendix with its own service analysis — this is the kind of stray appendix the base rule targets. **However**, it is explicitly cross-referenced by Section 1's Blast-Radius Note and NFR-018 as sanctioned grounding evidence and is labelled "not required reading for product/requirement review"; combined with the trailing **"## Revision Summary"** (expected and correct per spec-revise's own required template — NOT a violation), the document's structure is compliant. Treating the Engineering Reference as the sanctioned grounding appendix it declares itself to be, no FAIL is raised. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 (Information Requirements) is a pure bullet list. No attribute tables or type declarations. (Note: Section 6's Out-of-Scope table is in Section 6, not Section 4, and is appropriate there.) |

*Format verdict: COMPLIANT. No row reads FAIL.*

### KG Cross-Check (grounding reconciliation — not a document-content format issue)

The spec's Section 1 Blast-Radius Note and appendix **accurately** reflect the independent KG cross-check: it correctly reports the primary 6-service list, correctly notes the fresh lookup did **not** re-find `k4a-mcp-server`, and correctly surfaces `koch-query-api` and `summary-report-cloud` at low confidence. No contradiction between the spec's grounding claims and the evidence. The notable implication worth flagging: because `k4a-mcp-server` (the plan-entitlement-check service) is the un-re-found service, and FR-011/NFR-008 reference "plan-entitlement checks" that per OQ-004 are *not needed* for this feature, engineering should confirm whether any entitlement-service touchpoint is required at all — the spec's own "no plan gating" resolution (FR-011) suggests `k4a-mcp-server` may be out of scope, which would resolve the discrepancy cleanly rather than requiring its reconciliation.

---

## 6. Recommendations

1. **Resolve CG-001 (corrupted-artifact recall) before engineering.** Add a requirement to block server-side download of any report (and its FR-005a re-issued links) for a period marked invalid, independent of the global kill-switch. This is the highest-severity gap — it defeats the feature's own stated rollback trigger and risks finance reconciling against bad data.
2. **Resolve CG-002 (DEC-004 historical window) as a product/finance decision.** Do not leave most-recent-week vs. rolling-window to engineering discretion; it changes the artifact's meaning for year-end close.
3. **Resolve CG-003 (timezone/DST basis for Monday 9:00am).** Define the authoritative IANA timezone, DST rule, and no-zone fallback; without it NFR-004 is untestable and the primary value is at risk.
4. **Add a missed-deadline fallback requirement (NFR-017 gap).** Define what finance does if the feature slips past year-end close — the current mitigation is process-only, not a requirement.
5. **Bind the vague NFR thresholds** (NFR-001 ≤2s, NFR-004 "with margin," FR-017/NFR-020 propagation, NFR-013 retention) into testable acceptance criteria; remove "should"/"may refine"/"established range" hedging.
6. **Add requirements for partial-data and zero-activity weeks** (Edge Cases 2–3) — both are plausible for a retail account and currently undefined.
7. **Cap FR-005a link regeneration** and reconcile Retry-After honoring (FR-013) against the hard Monday deadline (Edge Cases 8, 13).
8. **Confirm during discovery whether `k4a-mcp-server` is in scope** given FR-011's "no plan gating" resolution, and resolve the scheduling/PDF-generation ownership overlaps flagged in NFR-018.

*Note: Per the Revision Summary, this round was PM-scoped to CG-003 (expired-link re-issuance), now correctly resolved via FR-005a. CG-001, CG-002, CG-003-timezone, and Edge Cases 2–12 remain intentionally deferred and are re-raised here as still-blocking.*