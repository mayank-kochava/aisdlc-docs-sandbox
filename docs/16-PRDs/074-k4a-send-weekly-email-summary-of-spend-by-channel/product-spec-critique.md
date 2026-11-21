# Spec Critique: Meridian Travel Weekly Spend Summary Email

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-21

## Summary

- **Critical Issues:** 1
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Alert email is itself a delivery via the same downstream path that just failed.** FR-013/FR-014/NFR-014 all mitigate silent non-delivery by sending a *failure alert email* to the marketing ops owner. But if the failure mode is the email-delivery dependency being down (NFR-013), the alert email travels the exact same path and will also fail — producing the silent non-delivery the whole design claims to prevent. No alternate alert channel (ops alerting from NFR-003, paging, dashboard) is specified as the authoritative signal. | Critical | FR-013, FR-014, NFR-003, NFR-014 | Decouple the human-facing failure signal from the email path. Make the operational alert (NFR-003 monitoring) the authoritative, path-independent notification; treat the alert email as best-effort only. Add an FR stating delivery-path failures raise a non-email operational alert. |
| CG-002 | **Idempotency vs. intentional re-run after a fixed failure is ambiguous.** FR-005/NFR-007 forbid a duplicate email for the same account+reporting-week key. But NFR-015 rollback and manual recovery scenarios (operator re-runs after a data fix) would be *blocked* by idempotency — leaving the team permanently without that week's email once a bad/empty run is recorded. There is no defined "failed run does not consume the idempotency key" rule. | High → treated under CG for visibility | FR-005, NFR-007, NFR-015 | Clarify that the idempotency key is consumed only by a *successful send*, so a failed/alerted run can be legitimately re-run and still deliver exactly one email. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-011 | "the email **clearly identifies** the reporting week the totals cover" | "Clearly identifies" is subjective/not testable — no defined format, field, or location. | Given the email body is rendered, When the summary is composed, Then the email header contains the reporting week start date and end date in ISO 8601 (YYYY-MM-DD) with the timezone label "ET", e.g. "2026-11-16 to 2026-11-22 (ET)". |
| FR-004 | "the already-running job is allowed to complete" | No assertion on what the *second* attempt observes/returns — untestable outcome for the blocked run. | Given a run is in progress for account+week, When a second run is attempted for the same key, Then the second attempt exits without side effects And logs a "concurrent-run-skipped" status And the first run completes unaffected. |
| NFR-010 | "beyond what is **needed to diagnose a failure**" | "Needed to diagnose" is unbounded and unmeasurable; no test can assert what a log *should* contain. | Define an explicit allowlist: Then logs may contain account ID (opaque), run ID, failure reason code, and channel count, And must not contain raw recipient email addresses or raw spend amounts. |
| NFR-012 | "disables new weekly runs **within 5 minutes**" | Testable, but "in-flight runs already started complete under prior behavior" lacks a bounded definition of how long an in-flight run may persist. | Add: And any in-flight run must complete or self-terminate within its configured max run duration (define value). |

*Note: Most FRs are correctly in Gherkin — testability issues above are primarily in the NFRs and two FRs with soft predicates.*

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard ~5-week / ~4-week deadline (NFR-011) | Partial | **FAIL** | "Mitigated when the integration path is resolved early" is a hope, not a mitigation. The spec admits (Section 1 KG note) the service set is unresolved and conflicting. There is no fallback plan or decision deadline if reuse of k4a-sam-backend/sam-performance-reporter turns out *not* to be close enough. A deadline risk mitigated by "resolve it early" is unmitigated. |
| Multi-service blast radius / kill-switch (NFR-012) | Yes | PASS | Config kill-switch, 5-min disable, in-flight behavior defined. Minor: in-flight bound unspecified (see §2). |
| Email-delivery dependency failure (NFR-013) | Partial | **FAIL** | Mitigation is retry + alert, but the alert uses the same failed email path (CG-001). The mitigation does not survive the exact failure it names. |
| Silent non-delivery (NFR-014) | Partial | **FAIL** | Same root cause as CG-001 — the "human is notified" guarantee collapses when the email dependency is the thing that is down. |
| Shared query-resource contention (NFR-001) | Yes | PASS | p95 target + off-peak scheduling defined and measurable. |
| Cross-account data leakage (NFR-008) | Yes | PASS | Server-side re-derivation of scope specified. |

---

## 4. Missed Edge Cases

1. **Alert path = failure path (recursive silent failure):** Covered in CG-001 — the single most important missed case; no path-independent escalation.
2. **Partial channel data (some channels retrieved, one source silent):** FR-008 covers zero/missing *per channel*, and NFR-005 fails closed on total retrieval failure, but there is no defined behavior for a *partially successful* retrieval where one channel's source is up but returns suspect/stale data vs. genuinely zero. "No data" and "zero spend" are collapsed into the same zero rendering (FR-008), which can silently mask a data outage as a legitimate $0.
3. **DST transition week:** Reporting week and send time are bounded in "Eastern time." The spring/fall DST changeover produces a 23- or 25-hour day and an 8:00 AM that occurs twice or not at all. No rule for scheduler behavior across the transition.
4. **Clock skew / missed scheduler tick:** FR-002 says "do not run outside cadence," but there is no catch-up rule if the scheduler is down *at* 8:00 AM Monday and recovers at 8:30. Does the week get skipped entirely (silent non-delivery) or run late? NFR-002's 60-min window implies late-run is acceptable but no FR specifies catch-up.
5. **Currency / multi-currency spend:** "Total spend" assumes a single currency. If channels report in different currencies, summing is invalid. No currency field in Section 4's spend records or aggregation.
6. **Recipient list is a shared mailbox (marketing-team@):** If the distribution list bounces or is reconfigured, delivery "succeeds" at the SMTP hop but no human reads it. No bounce-handling requirement.
7. **New channel appears / channel retired mid-quarter:** BR-6 hardcodes four channels. FR-008 keys off "every channel in the same state" but the channel *set* source (Section 4) is system-derived — behavior when a 5th channel appears or one of the four disappears is undefined.
8. **Negative spend / refunds / adjustments:** Aggregation sums spend records. If a credit/refund produces a negative record, the "total" could be negative or understated. No rule.
9. **Idempotency across a data correction (CG-002):** Re-run blocked after a bad run leaves the week permanently undelivered.
10. **Kill-switch flipped mid-render:** NFR-012 disables "new runs," but a run already past the scheduler check but mid-aggregation/mid-send is "in-flight" — the boundary between "new" and "in-flight" is not precisely defined for the render/deliver stages.
11. **Alert fatigue / alert storm:** NFR-003 alerts on >1% failure rate sustained — but the per-account alert email (NFR-003 first clause) has no dedup; a total outage would send one alert email per account attempt.
12. **Timezone of the recipient vs. Eastern:** Email identifies the week in ET, but marketing team members in other timezones have no stated handling; minor but worth a decision.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-014 are all in valid Gherkin blocks. |
| Document contains only Sections 0–6 | **FAIL** | The document contains a full **"Appendix: Engineering Reference"** — a multi-paragraph narrative section with "Affected Services" prose, bulleted service analyses, and reconciliation guidance — placed *before* the Revision Summary. This is exactly the stray multi-paragraph KG cross-check/analysis appendix the format rule is written to catch, not an exempt one-line metadata trailer and not the sanctioned Revision Summary. Move its engineering-planning content out of the product spec. (The trailing "## Revision Summary" section itself is EXPECTED and is **not** flagged.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list; each item uses prose annotations (Source/Why it matters) but contains no attribute tables or type declarations. |

---

## Grounding Cross-Check (KG evidence vs. spec claims)

- The independent KG lookup **agrees** with the spec's own Section 1 cross-check note: the service set is genuinely unresolved, and k4a-sam-backend / sam-performance-reporter are the strongest reuse candidates (medium confidence). The spec is honest about this discrepancy rather than contradicting the evidence — good.
- **However:** the spec commits to a ~5-week (really ~4-week) deadline and to a *reuse* strategy while admitting via its own evidence that the owning service for both scheduling and delivery is only *medium/low confidence* and unconfirmed against code. This is an internal-consistency / schedule-risk contradiction (see NFR-011 FAIL), not a factual contradiction with the evidence. No spec claim about data flow, joins, tables, or existing endpoints is asserted as confirmed fact that the evidence refutes — the spec correctly hedges all of them as "engineering to confirm."

---

## 6. Recommendations

1. **(Critical — CG-001) Decouple failure notification from the email path.** Make a path-independent operational alert the authoritative signal that a run failed; downgrade the alert email to best-effort. This is blocking.
2. **(High) Define failed-run idempotency semantics (CG-002):** key is consumed only on successful send, so recovery re-runs can still deliver.
3. **(High) Add a concrete deadline-risk fallback (NFR-011):** a decision gate date by which reuse-vs-build must be settled, and a reduced-scope contingency if reuse is not close enough.
4. **(High) Distinguish "zero spend" from "data unavailable" (Edge #2):** do not render a data outage as a legitimate $0 with no caveat.
5. **(High) Specify DST and missed-tick/catch-up behavior (Edges #3, #4).**
6. **(Medium) Add currency and negative/refund handling to aggregation (Edges #5, #8).**
7. **(Medium) Tighten vague predicates:** FR-011 ("clearly identifies"), NFR-010 ("needed to diagnose"), and in-flight-run bound for NFR-012.
8. **(Format) Remove the "Appendix: Engineering Reference" section** from the product spec (relocate to an engineering planning doc); keep only Sections 0–6 plus the sanctioned Revision Summary.
9. **(Medium) Add alert dedup/suppression to prevent per-account alert storms (Edge #11).**