# Spec Critique

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 1
- **High Issues:** 5
- **Medium Issues:** 6
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The authoritative spend data source and report-generation path are **not confirmed** (§1 engineering grounding caveat; two KG lookups disagree). FR-004/FR-005 require totals that "equal the authoritative spend figure" but the spec cannot name the authoritative source. Build cannot begin, and figure-correctness (the project's whole value) cannot be validated, until this is resolved. | Critical | §1, FR-004, FR-005 | Resolve service ownership during planning before build. Pin the single authoritative spend source and record it as a confirmed FR input. |
| CG-002 | **Monday-through-Sunday reporting window (BR-3) collides with latest-aggregate freshness (NFR-016).** BR-3 says each email covers the completed preceding Mon–Sun week; NFR-016 says figures reflect "the latest available aggregate at run time... rather than being held back to the completed totals of the prior full week." These two statements contradict: is the window the completed prior week, or a live partial aggregate? A customer-facing billing figure cannot be both. | High | BR-3, NFR-016, Info Req "Per-channel spend totals" | Choose one window semantic. If BR-3 wins, delete the "not held back to completed totals" clause in NFR-016. If freshness wins, redefine the window. |
| CG-003 | **No defined behavior when a channel is missing entirely from the data source** (vs. present-with-zero). FR-010 covers zero spend, FR-006 covers whole-run data-incomplete, but a run that returns 3 of 4 channels with no incomplete flag has no defined path — risk of silently dropping a channel. | High | FR-004, FR-006, FR-010 | Add an FR: if any of the four scoped channels is absent from the returned data (not explicitly zero), treat as data-incomplete per FR-006 (block send + alert). |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "must complete with a p95 latency target measured as a defined number of concurrent summary runs... engineering sets the concrete p95 target and concurrency count during planning" | No numeric target = not testable. Defers the actual acceptance threshold indefinitely. | Given N concurrent summary runs (N defined in planning), When per-channel totals are retrieved, Then p95 retrieval latency ≤ X ms. **Both N and X must be filled before this is a valid AC.** |
| FR-001 | "fires on Monday morning" / "any day other than Monday morning" | "Monday morning" is defined in NFR-001 (08:00 ET) but FR-001 does not reference that definition, so the trigger window boundary is untestable in isolation. | Given the scheduler, When the run fires at 08:00 America/New_York on Monday, Then generation begins; And When it fires at any other datetime, Then no summary is generated or sent. |
| FR-016 / NFR-013 | "new scheduled runs stop starting within 5 minutes of the flag being flipped" | "within 5 minutes" is testable, but "a run already in flight completes under the prior behavior" is undefined if the in-flight run's dependency later fails after the flip. | Given a run in flight at flip time, When it completes, Then it follows pre-flip behavior including FR-006/FR-012 failure handling; And no new run starts > 5 min after flip. |
| NFR-003 | "run failure rate exceeds 1% for 5 minutes sustained" | For a once-weekly single-customer batch, a "failure rate over 5 minutes" is statistically meaningless (sample size is effectively 1 run/week). | Given the single weekly run, When it does not reach a delivered outcome by the end of the Monday delivery window, Then an alert fires. (Drop the percentage-rate clause for this single-customer scope, or define the population it measures.) |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard external deadline (5 weeks) | Partial | **FAIL** | "Mitigated only if contested data-source path is resolved before build" — this is a precondition, not a mitigation. The spec leaves the single biggest schedule risk (CG-001) unresolved and dependent on planning. No contingency if resolution slips. |
| Multi-service blast radius | Yes (NFR-013 kill-switch, NFR-010, NFR-003) | PASS | Adequately addressed. |
| Billing/spend data handling | Yes (NFR-011, FR-006, NFR-006, NFR-004, NFR-012) | PASS | Well covered. |
| Legacy/reused report-generation path (NFR-015) | Partial | **FAIL** | NFR-015 says existing consumers "must remain unchanged" but then defers *which* consumers are at risk entirely to engineering. No concrete regression requirement or named existing report to protect. A risk acknowledged without a testable mitigation. |
| Reporting window unconfirmed (BR-3 working assumption) | No | **FAIL** | BR-3 and NFR-012 retention (12 months) are both flagged as unconfirmed working assumptions with no requirement to confirm them before go-live. Customer-facing spend scoped to an unconfirmed window is a correctness risk. |

---

## 4. Missed Edge Cases

1. **Partial-channel data (missing vs. zero):** See CG-003 — no defined handling for a channel absent from the source without an incomplete flag.
2. **DST transition on send day:** NFR-001 says "observing daylight-saving transitions," but no FR/test defines behavior on the two Mondays-adjacent-to-DST (e.g., the 08:00 ET that doesn't exist or occurs twice). Untested.
3. **Logo image fails to load:** FR-009a requires correct logo rendering, but no fallback behavior if the hosted image is unreachable at render/open time (broken-image icon in recipients' clients). Alt text is mentioned but not required to preserve layout.
4. **Email accepted by provider but bounced later:** FR-012/FR-013 treat "accepted for delivery" as success. A hard bounce or distribution-list rejection *after* acceptance is not covered — the audit records success while the team never received it.
5. **Distribution list empty / misconfigured:** FR-011 delivers to the configured list; no behavior if the list resolves to zero valid recipients.
6. **Currency / negative spend / refunds:** BR-2 assumes USD two-decimal positive totals. No handling for negative totals (refunds/adjustments) or non-USD figures in the source.
7. **Retry straddles the delivery window:** FR-007/FR-012 allow up to 3 backoff attempts; combined they could push dispatch past NFR-001's 08:00 ET target. No requirement reconciling the retry budget against the timeliness SLA.
8. **Kill-switch flipped mid-compose after data fetched:** FR-016 covers "in flight completes under prior behavior," but not whether a run that already fetched spend but hasn't sent should send during an active incident.
9. **Entitlement source returns ambiguous/degraded (not clean reachable/unreachable):** FR-014/NFR-005 cover reachable vs. unreachable, but not a slow/partial/malformed entitlement response.
10. **Audit write fails AND alert routing fails:** NFR-012 raises an alert on audit-write failure, but no fallback if the alerting path itself is down (silent loss of the audit guarantee).
11. **Duplicate-suppression window boundary:** FR-002/FR-003 key on (customer + reporting window). If the window computation differs by even a day across two triggers (e.g., DST), the idempotency key diverges and a duplicate could slip through.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-017 are all in Gherkin code blocks. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"Appendix: Engineering Reference"** — a multi-paragraph narrative section with its own prose findings ("Affected Services," "Lookup A/B," "Discrepancy to resolve"). This is a stray analysis appendix beyond Sections 0–6, not a sanctioned trailing Revision Summary or a one-line labeled metadata blockquote, so it is correctly flagged. (The trailing "## Revision Summary" itself is EXPECTED and is **not** flagged.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list with no attribute tables or type declarations. |

Note: Section 0 (Overview/metadata) is not present as a distinct numbered section; the document begins at Section 1. Minor — flag for author to confirm Section 0 intentionally omitted.

---

## KG Cross-Check (grounding note, folded into §1 per instructions)

The requirement-agent's independent fresh KG lookup **matches Lookup B** in the spec's appendix almost exactly (k4a-sam-backend, sam-report-scheduler, sam-performance-reporter, reporting-subscriptions-worker, notifications-api, sam-campaign-data-sync, cost-v2, frontend-mos). It does **not** re-find Lookup A's `reporting-api` or `accounts-api`. This reinforces CG-001: the fresh evidence tilts toward the SAM-specific stack, but the spec correctly refuses to pick a side. No contradiction with the spec's claims — the spec's §1 caveat and appendix accurately represent the ambiguity. The spec's entitlement claims (FR-014/BR-6) reference an "authoritative subscription/entitlement source" generically; note that `accounts-api` (the Lookup-A entitlement service) was NOT re-found in the fresh lookup, so the entitlement data source is also unconfirmed — fold into the CG-001 resolution scope.

---

## 6. Recommendations

1. **Resolve CG-001 before build:** pin the single authoritative spend data source and report-generation path (and the entitlement source — `accounts-api` was not re-found). The deadline "mitigation" is a precondition, not a plan; add a contingency if resolution slips.
2. **Resolve CG-002:** reconcile BR-3 (completed prior Mon–Sun week) against NFR-016 (latest live aggregate). A customer-facing billing figure cannot be both.
3. **Add the missing-channel FR (CG-003):** a scoped channel absent from the source (not explicitly zero) must be treated as data-incomplete and block the send.
4. **Fill NFR-002's p95 and concurrency numbers** so the latency AC is testable, and fix NFR-003's failure-rate clause for single-customer weekly scope.
5. **Harden NFR-015:** name the existing report(s)/consumers to protect and add a concrete regression requirement, rather than deferring entirely to planning.
6. **Confirm BR-3 window and NFR-012 retention** with the customer before go-live, or add requirements that force confirmation.
7. **Remove or relocate the "Appendix: Engineering Reference"** section to satisfy the Sections 0–6 constraint (it belongs in an engineering planning doc, not the spec).
8. **Cover post-acceptance delivery failures** (bounces), DST-boundary sends, and logo-image failure fallback with explicit FRs.