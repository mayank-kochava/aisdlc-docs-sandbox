# Spec Critique: Meridian Travel Weekly Campaign Spend Email

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
| CG-001 | The spec defers the data-cutoff/freshness definition (DEC-002) yet FR-004 and NFR-012 make "fail-closed on incomplete data" a core correctness guarantee. Without a concrete cutoff value, "the data for the window is not yet complete" is undecidable — the system cannot know when to defer vs. send. The entire fail-closed behavior rests on an undefined threshold. This blocks implementation of FR-004, FR-016, and NFR-012. | Critical | 3 (FR-004), 5 (NFR-012) | Resolve DEC-002 to a concrete rule (e.g., "window data is complete 24h after window end per SAM sync watermark") before engineering. Define the signal the system reads to decide completeness. |
| CG-002 | Deferral loop has no terminal state. FR-004 says if data is incomplete the run is "deferred or flagged," but no FR defines how long it can be deferred, how many times, or what happens when a deferred run never gets complete data before the NEXT week's window opens. Two un-sent windows could silently stack, or a window could be permanently skipped with no email ever sent and (per the flow) only a one-time alert. This is a dead-end / data-loss path. | Critical | 2 (flow), 3 (FR-004) | Add an FR defining max deferral horizon, escalation when exceeded, and explicit behavior when a window is abandoned (send partial-with-disclosure per FR-011, or skip-with-alert). Define interaction between a deferred window and the next scheduled run. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "must complete within a p95 latency target... Default target: p95 ≤ 5 seconds... soft operational target, not a user-facing SLA" | "Soft target" is not testable — a test either passes a threshold or it doesn't. "Representative single-account weekly load" is undefined. | Given a Meridian Travel weekly run against a dataset of [defined representative size], When the aggregation query executes, Then p95 latency over [N] runs must be ≤ 5 seconds. |
| FR-004 | "meets the agreed data-cutoff expectation" / "the agreed data-cutoff" | "Agreed" refers to an unresolved decision (DEC-002). Not testable until resolved. | (Pending DEC-002) Given the window closed at T, When [cutoff rule] is met, Then the run proceeds; otherwise it defers. |
| FR-002 | "it uses the configured send day, send time, and timezone" | No default or testable value; "configured" is a placeholder for DEC-001. Test cannot assert a timing without a value. | (Pending DEC-001) Given send config = [day, time, tz], When the clock reaches that instant, Then the job fires within [tolerance] minutes. |
| NFR-003 | "within the delivery SLA. Default: email sent within 2 hours of the window's data-cutoff being met" | Depends on data-cutoff (DEC-002) which is undefined, so the SLA clock has no start. | Given data-cutoff for window W is met at T, When delivery is attempted, Then the email is delivered by T + 2 hours or an alert fires. |
| FR-008 | "it uses the agreed set of marketing channels and channel naming" | "Agreed set" is a placeholder (DEC-003); untestable until the list exists. | (Pending DEC-003) Given channel set = [enumerated list], When spend is grouped, Then exactly those channels appear with the specified names. |

Note: All FRs are written in valid Gherkin — no prose-FR violations. The issue above is vagueness/undefined values within otherwise well-formed Gherkin, not format.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (~5 weeks) | Yes — MVP scope (Sec 6) + verify existing report-subscription path | PASS | Mitigation is a process action, not a requirement, but acceptable given it's a schedule risk. |
| Multi-service blast radius | Yes — kill-switch (FR-018/NFR-015), single-in-flight (FR-005/NFR-002), immediate alerting (NFR-004) | PASS | Well covered. |
| Billing/compliance data exposure | Yes — account-scoping (NFR-007), server-side eligibility (NFR-006), authorized-recipients (NFR-008), no-logs (NFR-009), audit trail (NFR-014) | PASS | Strong. |
| Legacy/uncertain service topology (reconciliation note, Sec 1) | Partial — recorded as a finding; NFR-017 covers backward-compat conditionally | PARTIAL / FAIL | The risk that the feature is built net-new when an existing scheduled-report path already exists is flagged but no requirement forces the engineering verification before build. NFR-017 only applies *if* the path is confirmed — there is no gate requiring confirmation first. Add a planning-gate requirement. |
| Downstream dependency failure (SAM, delivery) | Yes — FR-015, FR-016, NFR-013 | PASS | Bounded retry, named statuses, fail-closed all present. |
| Audit-write failure as compliance event | Yes — FR-019, NFR-005 | PASS | Good. |

---

## 4. Missed Edge Cases

1. **Deferred window collides with next scheduled run:** If window W is deferred for incomplete data and window W+1's schedule fires, behavior is undefined (see CG-002). Which window runs? Do both queue?
2. **Partial-channel data under fail-closed:** NFR-012 says fail-closed on incomplete data (no email), but FR-011 says send the email with "data unavailable" disclosure for a missing channel. These two directly conflict — is a single missing channel a fail-closed abort or an honest-disclosure send? The boundary between "degrade by withholding" (NFR-012) and "represent honestly" (FR-011) is not defined.
3. **Recipient list empty or all-invalid:** FR-013/FR-014 assume a non-empty authorized list. No FR covers "the configured recipient list is empty" or "every recipient address bounces/is invalid." Does the run count as success, delivery-failed, or skipped?
4. **Account becomes ineligible mid-run:** FR-017 re-derives eligibility at run start. If eligibility (plan downgrade, account suspension) changes *during* generation (after the check, before send), nothing re-checks it. Billing-sensitive email could send to a now-ineligible account.
5. **Email delivered but to a recipient removed that week:** Recipient-list change timing vs. run timing (FR-014 says "subsequent" emails reflect changes) — a removal applied mid-run has undefined effect.
6. **Zero spend / zero installs legitimately:** FR-011 forbids presenting *unavailable* data as a misleading zero, but a genuine zero (no spend that week) must be distinguishable from "unavailable." No FR defines how a true zero is rendered vs. a null.
7. **Timezone / DST transition:** FR-002/FR-003 configure a timezone, but behavior across a DST shift (spring-forward skips the send hour; fall-back repeats it) is unspecified — a run could be missed or double-fired. Interacts with FR-005's duplicate prevention only if scoped to the window, not the clock time.
8. **Audit record written but run then crashes before terminal state:** FR-019 writes the audit record at terminal state. If the process dies after send but before writing the audit record, the compliance record is missing for a sent email — only an alert fires (NFR-005), but no reconciliation is defined.
9. **Kill-switch flipped mid-window after data queried:** FR-018 says in-flight runs complete under prior behavior. If the kill-switch was flipped due to *incorrect spend data* (the NFR-016 rollback trigger), an in-flight run will still send the bad email. Rollback posture (NFR-016) and FR-018's "in-flight completes" directly conflict for the data-corruption trigger.
10. **Retry-After exceeds delivery SLA:** FR-015 honors Retry-After; NFR-003 mandates delivery within 2 hours. A delivery dependency returning a long Retry-After could force a choice between the two — unspecified which wins.
11. **Idempotency key definition:** FR-006 guarantees no duplicate email per window, but the uniqueness key (account + window boundary) is not stated. If the window definition (DEC-002/window) changes between runs, two "same window" runs could be seen as different and double-send.
12. **Data-cutoff never met permanently (dependency outage spanning the week):** Related to CG-002 — if SAM is down all week, the window never completes; no email, and only operational alerts. No defined manual override / forced-send path.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-020 are all valid Gherkin blocks. Trailing prose notes on FR-003 and FR-014 reference decisions but the FRs themselves are Gherkin. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present (Section 0 absent but that is not extra content). No appendix is embedded in the document body; the service-evidence "appendix" is only *referenced* in Section 1, not included. The two sentences "Now reporting the business trade-offs flagged in Sections 0–3." and the KG cross-check block are outside the END-OF-DOCUMENT marker and are not counted. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet items with prose descriptions; no attribute tables, no type declarations. Types explicitly deferred to engineering. |

---

## 6. Recommendations

1. **(Critical, CG-001) Resolve DEC-002 (data-cutoff/freshness)** to a concrete, testable rule before engineering. The fail-closed correctness model is unimplementable without it.
2. **(Critical, CG-002) Add a terminal-state FR for deferred/abandoned windows** — max deferral horizon, collision with the next window, forced-send/skip-with-escalation, and a manual override path for prolonged dependency outages.
3. **(High) Reconcile FR-011 vs. NFR-012** — define precisely when a missing channel/metric triggers honest-disclosure send vs. fail-closed abort. These currently contradict.
4. **(High) Reconcile FR-018 (in-flight completes) vs. NFR-016 (rollback on bad data)** — for the data-corruption kill-switch trigger, in-flight runs must NOT be allowed to send.
5. **(High) Add a KG-reconciliation planning gate requirement** — require engineering to confirm whether the existing scheduled report-subscription path (see below) is extended or a net-new pipeline is built, *before* build starts, not as a conditional NFR. See KG note below.
6. **(High) Cover empty/all-invalid recipient list and mid-run eligibility change** as named failure conditions.
7. **(Medium) Specify DST/timezone transition behavior** and bind FR-005's duplicate prevention to the window key, not clock time.
8. **(Medium) Define the idempotency key** (account + canonical window boundary) explicitly in FR-006.
9. **(Medium) Distinguish genuine zero from unavailable** in FR-011 rendering rules.
10. **(Medium) Resolve DEC-001, DEC-003, DEC-004, DEC-005** to replace untestable "configured/agreed" placeholders with concrete values.

### KG Cross-Check Note (grounding against independent lookup — feeds Recommendation 5)

The independent codebase-KG lookup **reinforces the spec's own Section 1 reconciliation concern and strengthens it into a near-certain finding**: `sam-report-scheduler` (confidence: **high**, "Schedules recurring SAM reports (weekly cadence)") and `reporting-subscriptions-worker` ("Delivers scheduled report subscriptions via email") both strongly suggest a pre-existing scheduled-report-subscription pipeline already exists. The spec treats "an existing path may exist" as an open possibility (Section 1, NFR-017); the fresh evidence indicates it is likely real. This does not contradict the spec's product behavior, but it materially raises the stakes of Recommendation 5: the spec's conditional NFR-017 is too weak — building a net-new 5-service pipeline when `sam-report-scheduler` + `reporting-subscriptions-worker` already cover scheduling and delivery would be a significant, avoidable duplication, especially under the 5-week deadline. Additionally, the spec's Information Requirements name "SAM campaign-spend data services" generically; the KG surfaces **two overlapping candidate spend-data sources** (`sam-campaign-data-sync` and `k4a-sam-campaign-data-sync`, both low confidence) — engineering must disambiguate which is authoritative to satisfy BR-1/FR-010 account-scoping, since querying the wrong source risks cross-account leakage. No direct contradiction of the spec's stated data-flow was found, but these are unresolved implementation-reality gaps the spec should route to planning rather than leave implicit.