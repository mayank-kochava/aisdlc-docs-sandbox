# Spec Critique: Meridian Travel Weekly Campaign-Spend Email

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
| CG-001 | The "data available" branch is binary, but FR-010 only handles the fully-unavailable case. There is no defined behavior for **partial data** — e.g. the spend source returns rows for 3 of 5 channels, or returns a 200 with a truncated/incomplete payload. FR-011 would then render a summary that silently under-reports spend, violating NFR-011's "no misleading summary" intent. | High | FR-010 / FR-011 | Add an FR defining a completeness check: if the fetched spend set is incomplete/partial against an expected channel-set or record-count signal, treat it as a data-unavailable failure (fail closed) rather than sending. |
| CG-002 | FR-009/NFR-009 prevent overlapping concurrent runs and double-summaries, but nothing defines **what happens to a missed week once the data source recovers**. NFR-003 alerts on a missed run, but there is no requirement to backfill or explicitly abandon the skipped period. A week can silently vanish with only an alert. | High | FR-009, NFR-003 | Add an FR specifying whether a missed/failed reporting period is backfilled on recovery or permanently abandoned, and that the chosen behavior is recorded against the reporting period. |
| CG-003 | FR-013 retries delivery and de-dupes successful recipients, but there is no bound on **how a permanently-bouncing recipient** affects the batch, nor what "affected recipients" state persists. A single hard-bouncing address could repeatedly trigger alerts every week forever with no suppression/removal path. | Medium | FR-013 | Define handling for persistently failing recipients (e.g. mark recipient as undeliverable, surface in config, suppress repeat alerts after N weeks). |
| CG-004 | The config save flow (FR-002/FR-003) has no concurrency/lost-update handling. Two admins editing the recipient list simultaneously could silently clobber each other's changes; the audit record (FR-006) would show both as succeeding. | Medium | FR-002, FR-003 | Add optimistic-concurrency / last-writer detection on subscription save, or explicitly declare last-write-wins as intended. |

---

## 2. Testability Review

The FRs are, unusually, almost entirely well-formed Gherkin with measurable assertions. The following still contain non-testable vagueness.

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-007 | "one row per channel in scope" | "Channel set in scope" is marked [DECISION REQUIRED] in Section 1 — the row count cannot be asserted because the expected channel set is undefined. Untestable until resolved. | Given the in-scope channel set is {defined set}, When aggregation runs for a period with spend across those channels, Then the result contains exactly one row per in-scope channel and no rows for out-of-scope channels. |
| FR-008 | "a single, consistently-applied attribution rule" | The rule itself is explicitly undefined (blockquote under FR-008). "Consistently applied" is testable only as internal invariance (total-of-channels == total), not as correctness. Cannot test attribution correctness. | Given attribution rule R is {defined}, When a multi-channel campaign's spend is aggregated, Then each unit is assigned to exactly one channel per R AND sum(channel rows) == total spend for the period. |
| NFR-001 | "representative concurrent weekly-generation load" / "expected weekly concurrency" | "Representative" and "expected" are undefined — no row count, no account count, no concurrency number. The p95 ≤ 5s target is unanchored to a load profile, so it is not reproducibly testable. | Given N accounts each with M campaign-spend records generating concurrently, When the aggregation query runs, Then p95 completion ≤ 5s. (Define N, M.) |
| NFR-002 | "spreading/queuing per-account generation rather than issuing all account queries simultaneously" | "Spreading/queuing" with no cap number, no max-concurrency, no QPS ceiling is not assertable. | Given the shared spend source has concurrency cap C, When weekly generation runs across all accounts, Then concurrent queries against the source never exceed C. (Define C.) |
| NFR-003 | "run reliably on its configured schedule" | "Reliably" is not measurable. The alert-on-miss clause IS testable; "reliably" is not. | Given a scheduled run is due at time T, When more than {grace window} elapses past T with no run recorded, Then a missed-run alert fires. (Define grace window.) |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard December deadline / limited course-correction | Yes — scope held to Phase 1 MVP + decisions resolved early | PASS | Mitigation is procedural (depends on PO closing 7 open decisions), not a requirement; acceptable but contingent. |
| Multi-service blast radius (5 services) | Yes — kill-switch (FR-015/NFR-013), distinct failure statuses + alerting (FR-010/FR-013/NFR-004), idempotency (FR-009) | PASS | Well mitigated with concrete FRs. |
| Ambiguity over which backend stack owns scheduled emails + spend data | Partial — deferred to engineering planning, recorded in Section 1 + appendix | PASS (with caveat) | Correctly surfaced rather than silently resolved. See Section 4 / KG cross-check — the discrepancy is real and still open; acceptable as an engineering-planning item but must be closed before build. |
| Customer-facing console UI change / new UX | Yes — Phase 1 UI minimized to enable/disable + recipients | PASS | Deferral table (Section 6) backs this. |
| Shared data-source starvation | Yes — NFR-002 rate cap | PASS (weak) | Mitigation exists but cap is unquantified (see NFR-002 testability flag). |
| Dropped audit record (compliance) | Yes — NFR-005 immediate alert, FR-006 non-blocking write + alert | PASS | Good. |
| PII in logs | Yes — NFR-007 | PASS | Masking allowed for alerts; acceptable. |

No risk is listed-without-mitigation. The weakness is that several mitigations (NFR-001/002/003 thresholds, attribution rule) are placeholders pending decisions, not hard failures.

---

## 4. Missed Edge Cases

1. **Partial/incomplete spend data (not a clean success or failure):** See CG-001. The happy-path assumes data is either fully present or fully absent.
2. **Missed-week backfill vs. abandonment:** See CG-002. Recovery behavior after a failed/missed cycle is undefined.
3. **Persistently bouncing recipient:** See CG-003. Repeat weekly hard-bounces have no suppression/escalation path.
4. **Concurrent config edits:** See CG-004. No lost-update protection on subscription saves.
5. **Recipient removed/deactivated between save and dispatch:** An address valid at save time may be an off-boarded employee by dispatch day; no re-validation at dispatch.
6. **Subscription disabled mid-cycle:** FR-004 says "subsequent cycles," but a disable during an in-flight generation (after query, before dispatch) has undefined behavior — mirror the kill-switch's "in-flight completes under prior behavior" rule, which is not stated for per-subscription disable.
7. **Kill-switch flipped back on:** FR-015/NFR-013 define disabling, but re-enabling and whether the skipped weeks backfill is undefined (overlaps CG-002).
8. **Timezone/DST boundary on the delivery schedule:** Delivery timing is [DECISION REQUIRED]; once set, DST transitions and reporting-period boundary alignment (does the week shift?) are unaddressed.
9. **Empty but valid period (genuine zero spend):** A real $0 week is indistinguishable from the fail-closed zero-value suppression (NFR-011). Spec must distinguish "genuine zero spend" from "no data" or risk suppressing a legitimate report.
10. **Attribution total rounding:** FR-008 asserts sum(channels) == total; floating-point/currency rounding across many campaigns can break exact equality. Define tolerance / integer-cents arithmetic.
11. **Very large recipient list / email size limits:** No upper bound on recipient count; no behavior if the rendered email exceeds provider size limits.
12. **Account deletion / subscription orphaning:** No behavior defined if the Meridian Travel account or its config is deleted while a subscription is active.
13. **Audit write ordering vs. in-flight generation:** FR-006 covers config audit; there is no audit linkage between a config change and the cycle it first takes effect on (traceability gap when debugging "why did last week use the old list?").

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-015 are all well-formed Gherkin. The blockquote under FR-008 is explanatory, not a prose FR. |
| Document contains only Sections 0–6 | **FAIL** | The document ends with an **"Appendix: Engineering Reference"** containing an "Affected Services" subsection — a multi-paragraph narrative analysis with its own prose findings about two competing service stacks and a recommendation that engineering reconcile them. This is NOT the sanctioned trailing "Revision Summary" section, nor a short one-line `> **Label:** ...` metadata blockquote trailer; it is exactly the kind of stray multi-paragraph KG-analysis appendix the format rule is written to catch. Move this content into an engineering-planning doc or collapse the open discrepancy into the Section 1 scope note only. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list items with prose descriptions; no attribute tables or type declarations. (Note: a table does appear in Section 6, which is permitted there.) |

---

## KG Cross-Check (Grounding Verification)

Comparing the spec's implementation claims against the independent KG lookup:

- The spec does **not** over-commit to a specific implementation reality — it explicitly flags that two lookups disagree on whether the `reporting-*` stack or the `sam-*` stack owns scheduled emails and spend-by-channel data, and defers resolution to engineering. This is the correct posture and matches the independent cross-check, which likewise surfaces a SAM-heavy set (`k4a-sam-backend`, `sam-report-scheduler`, `sam-performance-reporter`, `reporting-subscriptions-worker`, `notifications-api`, etc.) rather than the Context Seed's `reporting-api`/`reporting-dispatcher`.
- **No contradiction of implementation reality** was found — the spec claims no specific endpoints, tables, joins, or data flows that the KG evidence refutes. The only overlap both lists agree on is `frontend-mos` (console UI) and `sam-api` (spend data), both of which the spec references only as grounding context.
- The one substantive observation: the spec's Appendix presents the discrepancy accurately, but doing so inside a stray appendix is itself the Section 5 format failure above. The grounding content is sound; its placement is not.

---

## 6. Recommendations

1. **Remove/relocate the "Appendix: Engineering Reference" (format FAIL).** Keep only the short Section 1 scope note recording the unresolved stack-ownership discrepancy; move the services list into engineering planning.
2. **Add a partial-data completeness requirement (CG-001).** Define when an incomplete fetch is treated as fail-closed rather than rendered, and distinguish it from genuine $0 spend (edge case #9).
3. **Define missed-week recovery behavior (CG-002, edge cases #7).** Specify backfill vs. permanent abandonment for failed/skipped/kill-switched periods.
4. **Resolve the 8 open [DECISION REQUIRED] items (incl. NFR-008 retention).** FR-007/FR-008 and NFR-001/002/003 cannot be acceptance-tested until the channel set, attribution rule, timing, and load profile are fixed.
5. **Quantify the placeholder NFR thresholds (NFR-001, NFR-002, NFR-003).** Attach concrete load profile (N accounts, M records), a numeric source concurrency cap, and a missed-run grace window.
6. **Add concurrency protection on config saves (CG-004) and dispatch-time recipient re-validation (edge case #5).**
7. **Define persistent-bounce handling and recipient/email-size bounds (CG-003, edge cases #3, #11).**
8. **Specify attribution rounding/tolerance (edge case #10)** so FR-008's sum-equality invariant is testable in currency arithmetic.