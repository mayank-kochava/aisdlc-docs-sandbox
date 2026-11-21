# Spec Critique: Meridian Travel Weekly Spend Summary Email

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-21

## Summary

- **Critical Issues:** 0
- **High Issues:** 5
- **Medium Issues:** 7
- **Blocking:** No

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Partial data vs. missing data is conflated.** FR-009 fails-closed only when retrieval "fails or no spend data can be obtained." There is no defined behavior for *partially* retrieved data — e.g. 4 of 6 channels return, 2 time out. The system could silently render a summary that looks complete but is missing channels, violating NFR-005 ("correctness over availability"). | High | FR-009 / NFR-005 | Define a requirement for partial-retrieval detection: if any expected channel in the known channel set fails to resolve, treat the run as an unrecoverable data-retrieval failure (fail-closed), not a success with missing lines. |
| CG-002 | **Idempotency key collides with a legitimate re-send need.** FR-005 + NFR-007 block any duplicate email for the same account+reporting-week. But if the *first* email was rendered with wrong/incomplete totals and corrected, the idempotency guard would block the corrected re-send. No "supersede/correct" path is defined. | High | FR-005 / NFR-007 / NFR-015 | Add a requirement distinguishing an idempotent *retry* (same inputs) from an authorized *re-issue* (corrected data). Rollback (NFR-015) says emails are never retracted — but offers no corrected-resend path either, leaving a bad email as the final state. |
| CG-003 | **"Configured send day/time" has no defined source-of-truth timezone for the trigger itself.** FR-001 fires when "the system clock reaches the configured send day and time," but DEC-002 leaves the timezone unsettled. A trigger evaluated in UTC against a locally-intended send time can fire on the wrong calendar day. This is a scheduling correctness gap, not merely a product decision. | High | FR-001 / BR-3 / DEC-002 | Require the trigger evaluation, the reporting-week boundary, and the email's displayed week to all resolve against a single declared timezone. Make the dependency explicit so DEC-002 cannot be resolved for display only while leaving the trigger ambiguous. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-003 | "failure rate exceeds 1% of scheduled runs over a 15-minute window, **sustained**" | "Sustained" is unquantified — how many consecutive windows? Not automatable as written. | Given the weekly job failure rate exceeds 1% of scheduled runs, When that condition holds across 2 consecutive 15-minute windows, Then an operational alert is raised. |
| NFR-012 | "disables new weekly runs **within 5 minutes** of being flipped" | Testable number, but "in-flight runs complete under prior behavior" has no bound. A run stuck in retry could run far past 5 min. | Given the kill-switch is flipped, When an in-flight run is already started, Then it is allowed to complete within its own retry/timeout bound, And no new run starts after the switch propagates (≤ 5 min). |
| NFR-011 | "Mitigated when the integration path is resolved early in engineering planning" | "Early" is not testable and the mitigation is a planning activity, not a verifiable system condition. | Not a system requirement — move to a planning checklist item with a dated gate, or define a concrete acceptance condition. |
| FR-014 | "retries delivery a **bounded number of times** using exponential backoff" | FR body omits the number; only NFR-004 names "default 3." FR should be self-testable. | Given delivery fails for a transient reason, When recovery is attempted, Then delivery is retried at most 3 times with exponential backoff, And honoring any Retry-After, And on exhaustion the failure is recorded and alerted. |

> Note: The FRs are overall strong and genuinely Gherkin-formatted. The vagueness is concentrated in the NFR prose, which is acceptable as NFRs but should still be tightened.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard ~5-week deadline / schedule risk (NFR-011) | Partial | FAIL | Mitigation is a planning activity ("resolve integration path early"), not a verifiable requirement. No concrete gate or launch date (DEC-006 open). |
| Multi-service blast radius (NFR-012) | Yes | PASS | Config kill-switch, ≤5 min propagation, in-flight completes. Well-specified. |
| Email-delivery dependency failure (NFR-013) | Yes | PASS | Covered by FR-013, FR-014, bounded retry, distinct reason, alert. |
| Silent non-delivery (NFR-014) | Yes | PASS | Every failure raises an alert (NFR-003, BR-8). Strong. |
| Shared query-resource degradation (NFR-001) | Yes | PASS | p95 ≤ 5s cap + off-peak scheduling defined. |
| Partial/incomplete data correctness (NFR-005) | Partial | FAIL | Fails-closed on *total* retrieval failure but not on *partial* channel loss (see CG-001). |
| Backward compatibility on shared services (NFR-016) | Yes | PASS | Additive-only requirement stated. Verification method not specified but requirement is testable. |

---

## 4. Missed Edge Cases

1. **Partial channel retrieval:** Some channels resolve, others time out — not covered (see CG-001).
2. **Channel appears/disappears mid-week:** A new channel starts spending, or a tracked channel is retired, between runs. FR-008 handles zero/missing but not a dynamically-changing channel *set* (BR-6 / DEC-004 open).
3. **Corrected re-send after bad email:** No path to supersede an already-sent incorrect summary (see CG-002).
4. **Empty account — zero channels entirely:** FR-008 handles a zero-spend *channel*, but not the case where the account has *no* channels at all for the week. Is an email with zero lines sent, or is this a failure?
5. **Recipient list empty or all addresses invalid:** FR-012 delivers "to all configured recipients" — undefined if the list is empty or every address hard-bounces. Does this count as a delivery failure for NFR-003?
6. **Partial recipient delivery:** Email delivered to 2 of 3 recipients. Is this success, failure, or partial? FR-013 treats delivery as binary.
7. **Clock skew / missed scheduler tick:** If the scheduler is down at the configured send time and recovers 90 min late, does the run still fire (and violate NFR-002's 60-min window) or get skipped? No catch-up policy defined.
8. **DST transition on send day:** Related to CG-003 — a send time during a spring-forward/fall-back transition could fire twice or not at all.
9. **Retry-After exceeds the retry window:** NFR-004 honors Retry-After, but a downstream `Retry-After: 3600` would blow past NFR-002's 60-min delivery window. Interaction undefined.
10. **Kill-switch flipped mid-render:** NFR-012 addresses "in-flight runs already started" but not a run paused between aggregation and delivery — does it deliver or abort?
11. **Duplicate alert storm:** If one account fails every week, NFR-003's per-account alert fires repeatedly with no dedup/suppression guidance.
12. **Spend record currency/unit mismatch:** No requirement that spend amounts share a currency/unit before summing in FR-007. Mixed-currency channels would produce a meaningless total.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-014 are all proper Gherkin blocks. |
| Document contains only Sections 0–6 | FAIL | The document ends with an **"Appendix: Engineering Reference"** — a multi-subsection narrative appendix (Affected Services list plus prose cross-check reconciliation). This is not a sanctioned Revision Summary nor a short one-line labeled metadata blockquote; it is extra content beyond Sections 0–6 and should be removed or relocated out of the reviewed spec. (Note: Section numbering also starts at 1, not 0 — no Section 0 present.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list with italic *Source/Why it matters* annotations and no attribute tables or type declarations. |

---

## 6. Recommendations

1. **Define partial-retrieval behavior (CG-001).** Decide and specify that any missing expected channel triggers fail-closed; do not let a partial summary render as complete. Highest correctness risk.
2. **Add a corrected-resend / supersede path (CG-002).** Idempotency must not permanently block re-issuing a corrected summary after a bad email, since rollback never retracts.
3. **Bind trigger, week-boundary, and display to one declared timezone (CG-003).** Resolve DEC-002 as a single timezone dependency, not a display-only decision, and specify catch-up and DST behavior (edge cases 7, 8).
4. **Convert NFR-011's deadline mitigation into a verifiable gate** with a concrete date (resolve DEC-006) rather than "resolve early in planning."
5. **Tighten NFR-003 "sustained"** into a concrete consecutive-window count, and add alert dedup/suppression guidance (edge case 11).
6. **Specify empty/invalid/partial recipient handling** (edge cases 4, 5, 6) and empty-channel-set handling.
7. **Add a currency/unit consistency precondition to FR-007** before summing spend per channel.
8. **Remove or relocate the "Appendix: Engineering Reference"** so the document conforms to Sections 0–6; its KG reconciliation content is planning material, not spec content.
9. **Reconcile the KG integration-path discrepancy before build** (see note below) — the spec correctly flags it but leaves it open, which is appropriate for product review but blocks engineering.

> **Note on KG cross-check (grounding, not a document-content defect):** The spec is commendably honest that the Context Seed's curated service list (reporting-api, reporting-dispatcher, k4a-mcp-server, sam-api) does not fully agree with an independent knowledge-graph lookup. The independent evidence corroborates the spec's own caveats: the *scheduler* owner is ambiguous (reporting-dispatcher vs. sam-report-scheduler / reporting-subscriptions-worker), the *email delivery* owner is ambiguous (k4a-mcp-server vs. notifications-api), and spend-summary logic may already exist in k4a-sam-backend (possible scope reduction). The spec does **not** make any false concrete implementation claim (no asserted joins, tables, or endpoints that contradict the evidence) — it explicitly defers the integration path to engineering planning, which is the correct posture. This is flagged as an open planning item, not a contradiction defect in the spec's stated behavior.