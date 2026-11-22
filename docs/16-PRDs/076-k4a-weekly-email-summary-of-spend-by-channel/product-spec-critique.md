# Spec Critique: Meridian Travel Weekly Spend-by-Channel Email

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
| CG-001 | FR-003 (idempotent overwrite) says a re-run "overwrites the prior generation" and "no duplicate email record." But if the first run already **sent** an email to recipients, a re-run overwriting the record does not un-send it. The spec does not define whether a re-run after a successful send re-sends, suppresses, or is even permitted. Recipients could receive two emails for the same week with different numbers. | High | 3 / FR-003 | Add a guard: once a week's email has been delivered successfully, further runs for that week must not re-send unless an explicit re-trigger path is defined. Distinguish "overwrite the generation record" from "re-send to recipients." |
| CG-002 | FR-013 retry "proceeds to send once the data is available and complete," but there is no defined signal for what "complete" means — the Data-readiness signal (Section 4) is described but no FR defines how completeness is determined (e.g., all expected channels present? a sync-done flag? a row-count threshold?). The retry loop could send on the first partial response it interprets as "available." | High | 3 / FR-012–013 | Add an FR defining the completeness criterion explicitly (what makes data "complete" for a covered week) rather than leaving it implied. |
| CG-003 | FR-014 "notifies the responsible owner" but "responsible owner" is never defined as a configurable recipient anywhere in Section 4 (recipient list is the marketing team, not an operational owner). A missed-send alert may have no destination. | High | 3 / FR-014 | Add an operational-owner / escalation contact to Information Requirements and reference it in FR-014. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-009 | "...laid out so the marketing team can copy them directly into their existing spreadsheet without manual reformatting" | "without manual reformatting" is not objectively testable — it depends on DEC-002 (unresolved). No measurable layout contract exists. | Given the assembled content, When the email is formatted, Then each channel appears as one delimited row of exactly two fields (channel name, total spend) using a consistent machine-parseable delimiter and a fixed decimal format, And pasting the body into a spreadsheet produces one row per channel with spend in a numeric column. (Pending DEC-002.) |
| FR-011 | "the email contains all channel spend totals required for their spreadsheet" | "required for their spreadsheet" is subjective; not a testable completeness assertion. | Given a delivered email, When content is inspected, Then it contains exactly one line for every channel in the configured channel set (BR-6) with a numeric total for each. |
| NFR-003 | "within the acceptable lag window of the configured send time" | The acceptable lag window is itself undefined (BR-3 data-delay default), so the NFR has no numeric target to test against. | Define the lag window as a concrete value (e.g., Then delivery occurs within N hours of the configured send time) once DEC-001/lag default is set. |
| NFR-004 | "exceed 1% of scheduled sends over a 15-minute window, sustained" | For a single standard-plan account sending ~once/week, "1% over a 15-minute window" is statistically meaningless (denominator is effectively 0 in any 15-min window). The per-account single-miss alert is the only meaningful part. | Scope the 1%/15-min threshold to fleet-wide sends only, and make the per-account single-miss alert the primary, independently-testable rule. |

*Note: All FR-001 through FR-015 are written in valid Gherkin — no prose FRs found. The vague items above are testability (measurability) issues, not format violations.*

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (5 weeks) | Yes — MVP scope lock (Section 6), reuse-existing-pipeline lever | PASS | Mitigation is a planning action, not a requirement, but appropriate for a schedule risk. |
| Multi-service blast radius (conflicting service names) | Yes — kill-switch (NFR-012), bounded retries (NFR-005/007), required name reconciliation | PASS | Behaviorally sound. See CG and Section 7 note on the KG discrepancy. |
| Email delivery integration | Yes — FR-015, NFR-004/005/009/010 | PASS | Distinct delivery-failure condition and recipient confinement defined. |
| Data freshness / sync-timing | Yes — FR-012–014, NFR-007 | PASS (with CG-002 caveat) | Mitigation exists but hinges on an undefined "complete" criterion — see CG-002. |
| Double-send after successful delivery (implied by idempotency + delivery) | No | FAIL | Idempotency is defined for the generation record but not for re-delivery to recipients. See CG-001. |
| Missed-send notification has no destination | No | FAIL | "Responsible owner" is referenced but never defined as tracked info. See CG-003. |

---

## 4. Missed Edge Cases

1. **Re-send after successful delivery:** A week already emailed successfully, then re-triggered — re-send, suppress, or forbidden? (CG-001)
2. **Partial channel data:** Some channels synced, others not, for the same week. Is that "incomplete" (fail-closed) or "available" (send)? The completeness definition is missing. (CG-002)
3. **Channel appears mid-week / new channel not in configured set:** Spend exists for a channel that is not in the configured channel set — is it dropped silently, or flagged? FR-007 only iterates the configured set.
4. **Channel-set change between runs:** If BR-6's channel set changes between two weekly runs, the two emails become non-comparable. No requirement addresses what happens when configuration changes mid-stream.
5. **Negative / refunded spend:** Credits, refunds, or negative adjustments producing negative channel totals — are they allowed, floored at zero, or flagged?
6. **Currency / multi-currency:** Spend is shown as a figure with no currency defined. If the account has spend in multiple currencies, the "total spend" per channel is ambiguous.
7. **Timezone/DST boundary:** FR-004 resolves the week by the account's timezone; the behavior across a DST transition (a 23- or 25-hour day at the week boundary) is undefined and could drop or double-count an hour of spend.
8. **Recipient list empty at send time:** FR-010 assumes "one or more recipients are configured." No FR covers the zero-recipient case (a formatted email exists but nobody to send to).
9. **All recipient addresses invalid / all bounce:** FR-015 covers delivery failure and bounded retry, but not the terminal case where every recipient permanently bounces — does this become a missed-send?
10. **Account becomes ineligible between config and send:** NFR-008 re-derives eligibility at run time, but no FR defines the user-visible/operational outcome when a previously-eligible account fails the check at send time.
11. **Kill-switch flipped mid-run:** NFR-012 says an in-flight run completes under prior behavior — but a run that is mid-retry (FR-013) during a kill-switch flip could still send after the operator intended to stop it. Behavior for in-flight *retrying* runs is ambiguous.
12. **Audit-write failure during a successful send:** NFR-011 makes audit async and alerts on failure — but the send then has no durable audit record. No requirement defines reconciliation of a delivered-but-unaudited send.
13. **Clock skew / missed trigger:** If the scheduler misses a fire window entirely (outage spanning the send time), is there a catch-up run, or is that week simply missed? FR-001/FR-002 only cover duplicate suppression, not missed triggers.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001–FR-015 are all valid Gherkin blocks; no prose FRs. |
| Document contains only Sections 0–6 | FAIL | The document ends with an **"## Appendix: Engineering Reference"** containing a multi-paragraph narrative "Affected Services" analysis with its own reconciliation-notes prose and findings. This is not a Revision Summary and not a one-line `> **Label:**` metadata trailer — it is exactly the kind of stray multi-paragraph analysis appendix that is outside the sanctioned exceptions and must be removed or relocated to an eng-plan doc. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 contains only bullet-list items; field names/types are explicitly deferred to eng-plan. No attribute tables present. |

---

## 6. Recommendations

1. **Define post-successful-send re-trigger behavior (CG-001).** Separate "overwrite generation record" from "re-send to recipients" so recipients cannot get two conflicting emails for one week.
2. **Add an explicit data-completeness criterion (CG-002).** Define what makes a covered week's spend "complete" so the retry loop cannot send on a partial response it misreads as ready.
3. **Define the "responsible owner" / escalation contact (CG-003)** in Section 4 and wire it into FR-014's missed-send notification.
4. **Remove or relocate the Engineering Reference appendix** to bring the document back to Sections 0–6 (format FAIL). The KG-reconciliation content belongs in eng-plan, not the spec.
5. **Add FRs for the missed edge cases**, prioritizing currency ambiguity (#6), DST/timezone boundary (#7), empty-recipient and all-bounce cases (#8, #9), and channel-set-change-between-runs (#4).
6. **Make NFR-003 and NFR-004 testable** by supplying the concrete lag-window value and scoping the 1%/15-min threshold to fleet-wide rather than single-account math.
7. **Resolve DEC-001–DEC-006** (especially DEC-001 week boundary and DEC-002 output format) — several FR testability gaps unblock only once these decisions land.

---

## 7. KG Cross-Check Note (grounding evidence vs. spec claims)

The spec's own Section 1 "Source-of-truth / service note" and its Engineering Reference appendix already disclose the two-pass naming conflict honestly and state that **no product behavior depends on which service names are real** — which is the correct posture for a product spec. My independent KG lookup re-found the same cross-check set (`k4a-sam-backend`, `sam-report-scheduler`, `sam-performance-reporter`, `reporting-subscriptions-worker`, `summary-report-cloud`, `notifications-api`, `k4a-mcp-server`, `sam-campaign-data-sync`) and found **no contradiction** between the spec's implementation claims and the evidence: the spec makes no concrete claim about specific endpoints, tables, joins, or data flow that the evidence refutes — it explicitly defers all of that to engineering reconciliation. The only action item is the one the spec already flags: engineering must reconcile the report-builder and email-delivery roles (named only in the cross-check pass, not the Context Seed pass) during planning. No new discrepancy to raise beyond what Section 1 already surfaces.