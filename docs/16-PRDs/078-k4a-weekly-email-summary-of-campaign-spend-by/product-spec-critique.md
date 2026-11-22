# Spec Critique: Meridian Travel Weekly Spend Summary Email

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
| CG-001 | FR-016/NFR-009 cap delivery retries at 3 bounded attempts, but no FR defines what "transient" vs "permanent" means. A hard bounce (invalid recipient address) is permanent and will burn all 3 retries pointlessly, then alert. There is no requirement distinguishing a 4xx permanent delivery rejection from a 5xx/timeout transient one. | High | 3 (FR-016), 5 (NFR-009) | Add an FR: permanent delivery rejections (e.g. invalid-address, hard bounce) are NOT retried, recorded with a distinct reason, and alerted immediately without consuming the transient-retry budget. |
| CG-002 | FR-002 (exactly-once) and FR-004 (idempotency) rely on a "reporting week" natural key, but BR-7 (reporting period) is `[DECISION REQUIRED]` and unresolved. If the period is "prior 7 calendar days" rather than "Monday–Sunday," the "reporting week" key is ambiguous and the exactly-once guard cannot be tested or built deterministically. | High | 1 (BR-7), 3 (FR-002/004) | Resolve BR-7 before FR-002/004 are testable; define the exact string/field that constitutes the idempotency key once the period is chosen. |
| CG-003 | FR-016 says "retry up to 3 attempts" but FR-003 (overlap guard) + NFR-001 give no bound on total wall-clock time. If delivery retries with exponential backoff push the send past the Monday delivery window, no requirement defines whether a late-but-successful send is acceptable or should be treated as a failed run. | Medium | 3, 5 | Define a hard deadline: if a run cannot deliver before time T, treat as failed-run and alert even if a retry might eventually succeed. |

No issue rises to Critical: the fail-closed posture (FR-009, NFR-004, NFR-009) and no-silent-drop requirements (FR-017) mean no identified gap causes silent data loss or advertiser data leakage. The gaps above degrade testability/behavior, not safety.

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must complete well within its weekly cadence" / "with margin for the bounded retries" | "Well within" and "margin" are not measurable; no concrete deadline. | Given a weekly run starts, When it executes query+aggregate+compose+send, Then it completes and delivers no later than [X hours] before the configured Monday delivery time. (X must be set.) |
| FR-012 | "structured so it can be forwarded as-is or copied into a spreadsheet without manual reformatting" | "Without manual reformatting" is subjective and untestable without a defined format (columns? CSV? plain table?). | Given the email is composed, When a recipient copies the spend block into a spreadsheet, Then each channel lands in its own row and spend in its own numeric column with no merged cells or embedded markup. Define the exact content structure. |
| FR-019 / NFR-010 | "change propagates to new runs within 5 minutes" | Testable, but "new runs" for a once-a-week job means this can only be exercised ~weekly; needs a stated verification method (e.g. flag-read timing independent of schedule). | Given the kill-switch is flipped off at time T, When the flag store is read by the scheduler, Then the scheduler observes the off state by T+5min, verified independently of the weekly trigger. |
| FR-008 | "if no curated subset is defined, all channels tracked for Meridian Travel are included" | "Channels tracked" is undefined — tracked where, as of when? A channel with zero spend this period: included with 0, or omitted? | Given no curated subset is defined, When composing the channel list, Then include every channel with ≥1 campaign for Meridian Travel; AND specify whether a zero-spend channel appears as a 0 line or is omitted (currently undefined). |

All FRs ARE written in Gherkin — this section flags vagueness inside otherwise-Gherkin FRs plus the prose NFRs, not format violations (see Section 5).

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard 5-week deadline | Yes — MVP scope cut + early decision resolution | PASS | Mitigation is a real scope action, not just a note. Contingent on `[DECISION REQUIRED]` items being answered early (dependency, not a gap). |
| Multi-service blast radius | Yes — NFR-002, NFR-004, NFR-012 + architecture spike | PASS | Guardrails correctly stated against capability, not named services. Spike is a real planned action. |
| Scheduled email delivery complexity | Yes — NFR-008/009/003/010/011 | PASS | Idempotency, fail-closed, bounded retry, kill-switch all concretely specified. |
| Legacy/shared data-source dependency (ownership unresolved) | Partial — NFR-002 bounds load, NFR-004 fail-closed | PASS (with caveat) | Mitigated for safety, but no FR defines behavior if the eventually-selected data source has a DIFFERENT scoping model than assumed. NFR-004's "fail closed if scoping cannot be confirmed" covers this adequately. |
| Permanent vs transient delivery failure | No specific requirement | FAIL | Spec treats all delivery failures as transient-retryable (see CG-001). A permanent bounce has no dedicated handling. Add the FR proposed in CG-001. |

---

## 4. Missed Edge Cases

1. **Hard bounce / invalid recipient address:** No FR distinguishes a permanent delivery rejection from a transient one; FR-016 would waste the retry budget (see CG-001).
2. **Zero-spend channel:** FR-008/FR-011 don't state whether a channel with $0 spend this period appears as a "0" line or is omitted. Ambiguous for both composition and test.
3. **All channels zero / no spend at all:** If Meridian Travel had zero spend across every channel in the period, does an email still send (an all-zero summary) or is that treated like missing data? FR-009 only covers retrieval *failure*, not legitimately-empty results.
4. **Partial-data rule undefined at build time:** BR-8 is `[DECISION REQUIRED]`, yet FR-010 depends on it. Until resolved, "the email clearly indicates incomplete channels" cannot be fully tested (what does "hold" vs "send-and-flag" produce?).
5. **Clock change / DST and timezone:** BR-2 timezone is `[DECISION REQUIRED]`. Spring-forward/fall-back could cause a run to fire twice or skip; no FR addresses scheduler behavior across a DST boundary.
6. **New channel appears mid-period:** A channel with first-ever spend during the reporting window — is it auto-included (FR-008 "all tracked") even if the curated subset predates it? Undefined.
7. **Audit-write failure vs send success:** NFR-005 says audit is async and an audit-write failure alerts — but if the email sent successfully and the audit write then fails, FR-018 ("records the successful send") is violated while the recipient already has the email. No FR reconciles "delivered but unaudited."
8. **Recipient list empty or misconfigured:** FR-015 assumes a configured recipient list exists. If the fixed operational setup has an empty/invalid list, no FR defines whether the run fails-closed, alerts, or sends to no one silently.
9. **Scheduler fires while a run is mid-DST-delayed:** Interaction of FR-002 (once per week) and FR-003 (overlap guard) with a run that spans unusually long is covered conceptually but not bounded in time (see CG-003).
10. **Kill-switch flipped mid-run:** NFR-010 says an in-progress run "may complete under prior behavior." This means the email can still go out seconds after an on-call tries to stop it. Acceptable per spec, but no FR records that a completed-after-disable send occurred, which could confuse incident forensics.
11. **Currency / formatting of spend figures:** No requirement specifies currency, decimal precision, or thousands formatting for the per-channel totals — directly affects FR-012's "paste into spreadsheet" testability.
12. **Duplicate channel names from different sources:** FR-006 guarantees "each channel appears at most once," but if the eventually-selected data source names the same channel differently (e.g. "Paid Search" vs "Search - Paid"), de-duplication logic is unspecified.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-020 are all in proper Gherkin blocks. NFRs are intentionally prose per the template and are not FRs. |
| Document contains only Sections 0–6 | FAIL | The document contains a trailing **"## Appendix: Engineering Reference"** with a multi-paragraph narrative "Affected Services" analysis, confidence ratings, and a "Key discrepancies for the spike to resolve" findings list. This is a multi-paragraph narrative KG cross-check appendix with its own prose findings — exactly the stray-appendix shape the base skill flags, NOT a one-line labeled metadata trailer and NOT a Revision Summary. It must be removed or relocated out of the spec body. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 contains only bullet-list items. No attribute tables or type declarations present. |

Note on the Appendix (SR-001, feeding the FAIL above): the appendix is substantive narrative content (service candidates, confidence tiers, discrepancy analysis), not a sanctioned trailing Revision Summary or a run of short `> **Label:** ...` metadata blockquote lines. Section 1 already summarizes the ownership ambiguity adequately for product review; the detailed service evidence belongs in a separate engineering planning artifact, not inside the spec's numbered-section body.

### KG Cross-Check (against independent evidence)

No contradiction found between the spec's implementation claims and the independent KG lookup. The spec's embedded "fresh knowledge-graph lookup" list matches the independent cross-check evidence (k4a-sam-backend, sam-report-scheduler, sam-performance-reporter, sam-campaign-data-sync, k4a-sam-campaign-data-sync, reporting-subscriptions-worker, reporting-dispatcher, notifications-api, cost-v2, frontend-mos) with the same confidence tiers. Importantly, the spec does **not** assert a resolved data flow, join, or existing endpoint/table — it explicitly flags ownership as unresolved and defers to an architecture spike. Because it makes no concrete implementation-reality claim, there is nothing for the KG evidence to contradict. This is handled correctly; the only issue is the *placement* of the evidence (Section 5 FAIL above), not its accuracy.

---

## 6. Recommendations

1. **Remove or relocate the "Appendix: Engineering Reference"** out of the spec body to restore Sections 0–6 compliance (Format FAIL). Section 1 already captures the ownership ambiguity sufficiently.
2. **Add a permanent-vs-transient delivery-failure FR** (CG-001 / Risk Audit FAIL): permanent rejections (hard bounce/invalid address) are not retried, are recorded distinctly, and alert immediately.
3. **Resolve BR-7 (reporting period)** before FR-002/004/007 are treated as testable — it defines the idempotency key and the meaning of "total spend" (CG-002).
4. **Define zero-spend and all-zero behavior** (edge cases 2 & 3): whether a $0 channel renders a line and whether a legitimately-empty period sends an email at all.
5. **Make NFR-001 measurable**: set a concrete deadline (hours before Monday delivery) and define whether a late-but-successful send counts as a failed run (CG-003).
6. **Specify FR-012's exact content structure** (columns, currency, precision) so "paste into spreadsheet" is testable (edge case 11).
7. **Address scheduler DST/timezone behavior** once BR-2 is resolved (edge case 5), and reconcile the "delivered-but-unaudited" case between FR-018 and NFR-005 (edge case 7).
8. **Define empty/invalid recipient-list handling** for FR-015 (edge case 8).