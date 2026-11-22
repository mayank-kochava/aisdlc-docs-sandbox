# Spec Critique

---
id: product-spec-critique
title: Product Specification Critique
---

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
| CG-001 | Audit-write failure posture conflicts with send path. NFR-005 says the audit write is asynchronous so it "never blocks the send," but an audit-write failure "must raise an immediate operational alert." There is no FR describing what the system does if a send succeeds but the audit entry never lands — can a run be re-triggered and double-send because no run record exists? The exactly-once guarantee (FR-002/FR-004) depends on a run record that NFR-005 allows to silently not exist. | High | 5 (NFR-005), 3 (FR-002/004) | Add an FR defining behavior when the run record / audit write fails: must the run be treated as not-sent (risking duplicate) or sent (risking lost audit)? Define the ordering and the dedupe key's durability guarantee. |
| CG-002 | "All channels missing" is undefined. FR-009 covers data-source-unavailable (fail closed, no email). FR-010 covers "one or more channels missing/delayed" (send with available). But the boundary case where retrieval *succeeds* yet *every* channel is missing/zero is unspecified — does it send an all-flagged-incomplete email, or is that treated as FR-009? | High | 3 (FR-009/FR-010), flowchart node F | Define the all-channels-missing / all-zero case explicitly: send a fully-flagged email, suppress, or alert. |
| CG-003 | Partial-send is irreversible and un-retracted. When FR-010 sends a partial email at 08:00 and the missing channel's data arrives at 08:05, the VP has already received an understated total with no correction mechanism. No FR defines whether a corrected/follow-up summary is ever issued. This is the original business risk (manual copy *errors*) reintroduced as *systematic understatement*. | High | 3 (FR-010), BR-8 | Add an FR stating whether late-arriving channel data triggers a correction email or is simply absorbed into no further action; make the non-correction explicit if that is the decision. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must complete well within its weekly cadence… lands within the team's Monday workflow window… with margin" | "Well within," "workflow window," and "margin" are not measurable. No testable deadline exists. | Given a weekly run fires at 08:00 ET Monday, When the run executes, Then the email must be accepted by the delivery path no later than 08:30 ET the same Monday (including all FR-016 retries). |
| FR-012 | "structured so it can be forwarded as-is or copied into a spreadsheet without manual reformatting" | "Forward-and-paste-friendly" is subjective; no test can assert "friendly." No defined format (plain-text table? HTML table? TSV?). | Given the composed email, When a recipient copies the spend block and pastes into a spreadsheet, Then each channel lands in its own row and the total in its own numeric cell, with an explicit column delimiter (define: tab-separated / HTML `<table>`). |
| FR-016 | "bounded exponential backoff up to 3 attempts" | "Exponential backoff" has no base/ceiling defined; 3 attempts total-or-additional is ambiguous. | Given a transient delivery failure, When retrying, Then attempt 2 occurs after N seconds and attempt 3 after 2N seconds (define N), for a maximum of 3 total attempts, after which FR-017 applies. |
| FR-019 / NFR-010 | "propagates to new runs within 5 minutes" | Testable, but "new runs" for a once-weekly job means the 5-minute window is only exercised if a run fires inside it — essentially untestable in production cadence. | Clarify that the 5-minute propagation is asserted against the flag-read point, not run cadence: Given the flag is flipped off, When a run is scheduled ≥5 minutes later, Then it does not start. |
| NFR-002 | "must not issue unbounded or concurrent-heavy queries… a single sequential aggregation query path per run is sufficient" | "Concurrent-heavy" and "sufficient" are not assertions. | Given a single weekly run, When it queries spend, Then it issues at most one aggregation query against the shared data source and no concurrent queries. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard 5-week deadline | Yes — MVP scope cut + early decision resolution | PASS | Concrete and tied to Section 6. |
| Multi-service blast radius | Partial — NFR-002/004/012 + architecture spike | PASS (with caveat) | Guardrails are stated against the capability, which is correct. But the spike is a *dependency*, not a mitigation the spec controls; if ownership is unresolved at build start, no FR describes a fallback. Acceptable but note residual risk. |
| Scheduled email delivery complexity | Yes — NFR-008/009, NFR-003, kill-switch/rollback | PASS | Well covered. |
| Legacy/unresolved data-source ownership (Section 1) | Partial — NFR-004 fail-closed, NFR-002 bounded load | PARTIAL / FAIL | The spec acknowledges the *authoritative spend data owner is unresolved* but ships no fallback requirement for what the run does if the selected source turns out to lack tenant-scoping or spend-by-channel granularity. NFR-004 says "fail closed if scoping cannot be confirmed" — good — but there is no FR covering a source that returns spend *without* channel attribution (FR-006/FR-008 assume channel is always present). |
| Audit-write failure (NFR-005) | Acknowledged but mitigation is only "alert" | FAIL | Raising an alert is detection, not mitigation. See CG-001: the durability of the exactly-once dedupe key is not guaranteed if the audit/run record can silently fail. |

---

## 4. Missed Edge Cases

1. **All channels missing but retrieval succeeded:** Falls between FR-009 and FR-010 — undefined (see CG-002).
2. **Late-arriving channel data after a partial send:** No correction path (see CG-003).
3. **Zero spend vs. missing data ambiguity:** A channel with a legitimate $0 spend for the week is behaviorally indistinguishable from a missing channel unless the flag in FR-010 explicitly separates "reported zero" from "did not report." Not stated.
4. **Negative or adjusted spend (refunds/credits):** FR-006/FR-007 assume spend is additive and non-negative. No handling for a negative per-channel total or a mid-week spend correction/reversal.
5. **DST transition on 8:00 AM Eastern:** BR-2 says "Eastern Time." On the spring-forward / fall-back Mondays, "8:00 AM Eastern" is unambiguous, but the reporting-period boundary (BR-7, Mon–Sun) crosses a DST change twice a year — spend records near the boundary may be double-counted or dropped depending on timezone handling. Not addressed.
6. **Reporting-period boundary timezone:** FR-007/BR-7 say "Monday to Sunday" but never state the timezone of that window. If spend is timestamped in UTC and the window is Eastern, boundary spend is misallocated. Undefined.
7. **New channel introduced mid-period:** FR-008 lists four fixed channels. If K4A begins tracking a fifth channel for Meridian, BR-5/FR-008's enumerated set silently excludes it. No "all currently-tracked channels" dynamic behavior is tested.
8. **Recipient list empty or all addresses invalid/hard-bounce:** FR-015 assumes a configured list exists. No behavior for an empty recipient config, or when every recipient hard-bounces (vs. transient failure in FR-016).
9. **Partial recipient delivery:** FR-016/FR-017 treat delivery as all-or-nothing. If 3 of 5 recipients accept and 2 bounce, is that "success" (FR-018) or "failure" (FR-017)? Undefined.
10. **Kill-switch flipped mid-run:** NFR-010 says an in-progress run "may complete under prior behavior." This means an email can still be sent *after* an on-call engineer disabled the feature during an incident — directly contradicting the incident-mitigation intent. Worth an explicit decision.
11. **Scheduler misses the trigger entirely (job host down at 08:00):** FR-001 covers the trigger firing; nothing covers a *missed* fire. Is there catch-up, or is the week simply skipped silently? NFR-003 alerts on *failed* runs, but a run that never started may not register as failed.
12. **Advertiser has zero campaigns:** FR-005 retrieves "all campaigns belonging to Meridian Travel." If there are none, the downstream aggregation (FR-006) produces an empty result — intersects with CG-002 but is a distinct trigger.
13. **Clock skew / duplicate reporting-week key across year boundary:** The dedupe key is "advertiser + reporting week" (NFR-008). ISO week numbering across a year boundary (week 52/53 → week 1) can collide if the key isn't year-qualified. Not specified.
14. **Audit retention vs. deletion request:** NFR-005 mandates append-only 12-month retention and NFR-006 handles recipient addresses as user-identifying data. A data-subject deletion request for a recipient address stored in an immutable audit trail is an unaddressed conflict.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-020 are all in Gherkin blocks. NFRs are intentionally prose, which is acceptable. |
| Document contains only Sections 0–6 | FAIL | The document contains an **"## Appendix: Engineering Reference"** between Section 6 and the Revision Summary. This is a multi-paragraph narrative appendix (Affected Services, "Key discrepancies for the spike to resolve") with its own prose findings — exactly the kind of stray analysis appendix the format rule is meant to catch, not a sanctioned one-line metadata trailer. The trailing **"## Revision Summary"** is expected and correct (not flagged). |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list of business information items with no attribute tables or type declarations. |

---

## 6. Recommendations

1. **Resolve the audit/run-record durability gap (CG-001, Risk Audit FAIL).** Define exactly-once behavior when the audit/run record write fails, so the dedupe guarantee can't silently break and double-send.
2. **Define the all-channels-missing / all-zero boundary (CG-002) and distinguish "reported $0" from "missing" (Edge Case 3).** These two gaps together determine whether an empty or understated email ever reaches the VP.
3. **Decide and document the late-data correction policy (CG-003).** State explicitly whether a corrected follow-up is ever sent, or that understated partials stand.
4. **Pin timezone on the reporting-period window and handle DST (Edge Cases 5, 6).** State the window timezone and boundary rule; this directly affects numeric correctness.
5. **Make NFR-001 and FR-012 testable.** Add a concrete delivery deadline and a defined machine-parseable content format (TSV/HTML table).
6. **Move the Engineering Reference appendix out of the spec body** (or into a linked companion doc) to satisfy the Sections 0–6 format rule.
7. **Define partial-recipient and empty-recipient delivery semantics (Edge Cases 8, 9).**
8. **Clarify the kill-switch-mid-run behavior (Edge Case 10)** so incident responders know an email can still land after they disable the feature.
9. **Add missed-trigger / catch-up behavior (Edge Case 11)** so a never-started run is observable, not silent.
10. **KG cross-check note:** The spec's own Section 1 and appendix already acknowledge the ownership ambiguity across the two lookups, and the independent KG evidence matches the spec's reproduced "fresh lookup" list (k4a-sam-backend, sam-report-scheduler, sam-performance-reporter, sam-campaign-data-sync, k4a-sam-campaign-data-sync, reporting-subscriptions-worker, reporting-dispatcher, notifications-api, cost-v2, frontend-mos) — **no contradiction found** between the spec's stated implementation reality and the KG evidence; the spec correctly refrains from asserting a single owner and defers to the architecture spike. This is handled correctly and is not a defect.