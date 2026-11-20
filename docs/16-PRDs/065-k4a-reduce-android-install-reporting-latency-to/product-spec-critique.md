# Spec Critique: K4A Android Install Same-Day Reporting

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-20

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The entire acceptance suite is gated on two unresolved decisions. DEC-001 (does "same-day" mean continuous intraday updates or a single end-of-day refresh?) determines whether FR-001/NFR-001 pass or fail, and DEC-002 (numeric parity variance gate) determines when FR-004/NFR-005 are satisfied. With both open, NONE of the same-day FRs are testable — "within the same calendar day" with no defined refresh cadence cannot be asserted against. The Revision Summary acknowledges this and explicitly defers it. A spec whose core behavior cannot be tested until two business decisions land is not implementation-ready. | Critical | 1 (BR-2), 5 (NFR-001, NFR-012) | Resolve DEC-001 and DEC-002 before engineering. At minimum, pin a provisional refresh cadence (e.g. hourly) and a provisional variance (e.g. 0%) so the suite is executable, and mark them as assumptions to revisit. |
| CG-002 | **Contradiction with KG evidence.** The spec (Section 4, Appendix) names `event-manager-api` as the Android install "event ingestion pipeline" that "validates timing of Android install events." But the Appendix's own endpoint enumeration shows `event-manager-api` exposes ONLY single-event CRUD/lookup (`DELETE /event/delete`, `DELETE /event/{id}`, `GET /event/{id}`, `GET /events/list`, `PATCH /event/{id}`) — there is NO bulk ingestion or timing endpoint. The independent KG cross-check does not re-surface `event-manager-api` at all and instead places ingest on `inboard-injector` / `inboard-sync-v2` / `inboard-log-jammer`. The spec therefore names an ingestion authority whose actual interface contradicts that role. Attributing install-reporting latency to this service for the discovery trace risks pointing engineering at the wrong hop. | Critical | 1 (KG Cross-Check), 4, Appendix | Do not treat `event-manager-api` as the install-ingestion/timing authority. The discovery trace must empirically confirm which service ingests Android install events before FR-001 scope is set; reconcile against the `inboard-*` cluster surfaced by the independent lookup. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-001 | "reflected in the K4A dashboard on that same calendar day" | "Same calendar day" without a cadence (DEC-001) is unassertable — a test can't know the deadline within the day to fail against. | `Given an Android install recorded at <time> on day D / When the next scheduled refresh after <time> completes / Then the install is reflected in K4A no later than <agreed cadence boundary> on day D` |
| FR-005 | "a freshness indicator (an 'as-of' timestamp) reflecting the actual recency" | "Actual recency"/"reflecting" is not a bound; no tolerance for how far the displayed as-of may lag the true load watermark. | `Given data is loaded to watermark W / When the dashboard renders / Then the displayed as-of timestamp equals W within <N seconds>` |
| NFR-001 | "within the same calendar day ... engineering should target intraday refresh at hourly aggregation granularity" | "Working default" wording makes the actual requirement ambiguous; a reviewer cannot tell if hourly is the pass bar or a placeholder. | `Given data recorded in hour H / When the hourly refresh for H completes / Then it appears in K4A within <N> minutes of the top of hour H+1` |
| NFR-004 | "within its current performance envelope"; "must not add perceptible load time" | "Current performance envelope" and "perceptible" are undefined. p95 baseline is referenced but never stated numerically. | `Given representative concurrent load L / When the Android install view renders / Then p95 render time ≤ <baseline ms> + <tolerance ms>` |
| NFR-002 | "at or better than current measured values" | "Current measured values" are not captured in the spec; nothing to assert against. | Capture and cite the baseline latency per report, then: `Then measured latency ≤ <cited baseline> for each existing report` |

Note: all FR-001–FR-012 are correctly written in Gherkin blocks. The vagueness above is in the *content* of the Then-clauses, not the format.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard month-end deadline + unlocated root cause | Yes — NFR-012 (discovery-first), BR-3 (scope boundary) | PASS | Mitigation is concrete; discovery precedes implementation. |
| Multi-service blast radius | Yes — NFR-002/003, FR-008/009, NFR-013/014 | PASS | Kill-switch and collateral-impact FRs present. |
| Unknown latency source | Yes — NFR-008 (named failure conditions), NFR-012 (per-hop trace) | PASS | Named conditions make the trace actionable. |
| KG cross-check discrepancy (two lookups disagree on affected services) | Partial — "discovery trace must reconcile both sets" | FAIL | This is a deferral, not a mitigation. No requirement forces reconciliation to a verifiable outcome before scope lock, and the `event-manager-api` endpoint contradiction (CG-002) is exactly the kind of false lead that can survive "reconcile both sets." Add an explicit discovery exit criterion: the ingest hop must be empirically confirmed before FR-001 scope is fixed. |
| Accuracy sacrificed for speed (BR-4) | Yes — FR-004, NFR-005/006 | PASS | Reconciliation and idempotency are specified. |
| Late-arriving events reconciliation under acceleration | Partial — FR-004 asserts final total matches | FAIL (High) | The requirement states the *end state* must match but defines no behavior for the window between first same-day publication and reconciliation — i.e. what the dashboard shows while a day's count is still settling. See EC below. |

---

## 4. Missed Edge Cases

1. **Reconciliation window display:** FR-004 guarantees the final reconciled total matches Orion Mobile, but nothing defines what the dashboard shows *between* early same-day publication and reconciliation. Does a user see a count that later decreases? Is a downward correction allowed, and how is it surfaced against BR-1 ("same totals")?
2. **Orion Mobile unavailable:** FR-002/NFR-005 require parity comparison against Orion Mobile's internal dashboard, but no behavior is defined if Orion Mobile's own feed is delayed or down. Which side is authoritative when the baseline itself is stale?
3. **Day-boundary clock skew / DST:** FR-003 asserts same-day attribution "follows Orion Mobile's convention," but no edge case covers DST transitions, leap-second, or an install recorded exactly at the boundary instant.
4. **Kill-switch flipped mid-reconciliation:** FR-010/FR-011 cover runs "in flight," but not a run that has published an early same-day total and is awaiting reconciliation when the switch flips — is that day left in a half-reconciled published state?
5. **Freshness indicator when NO data has ever loaded for the day:** FR-006 covers "last known totals," but the first-load-of-day cold state (no prior-day fallback exists, e.g. new account) is undefined.
6. **Cache invalidation race (FR-007):** No requirement defines behavior when upstream data updates *during* a dashboard render/refresh — stale-while-revalidate vs. hard block is unspecified.
7. **Partial-day data presented as complete:** If intraday refresh (NFR-001) loads only part of a day, is a partial count visually distinguishable from a complete day's total, or could a mid-day figure be mistaken for the day's final total?
8. **Shared-resource rate cap engaged (NFR-003):** When the 80% cap throttles the accelerated path, which path yields — does Android install reporting degrade, or does unrelated reporting? FR-008 says unrelated must be unaffected, implying Android yields, but the degraded Android behavior under throttle is unspecified.
9. **Idempotency key collision across reporting days:** FR-012/NFR-006 rely on a reporting-day key; no edge case covers re-runs that span a day boundary or a corrected attribution that moves an install between days.
10. **Backward-compat consumer reading mid-acceleration (NFR-015):** No edge case covers an existing consumer that polls at the old cadence and now receives updated totals sooner — could an existing caller see a value change between polls that it previously treated as immutable?
11. **Parallel-run divergence during soak (NFR-012):** No defined behavior if the old and new paths disagree *during* the soak — which path feeds the live dashboard while divergence is being investigated?

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-012 are all in Gherkin code blocks. |
| Document contains only Sections 0–6 | FAIL | A multi-paragraph "## Appendix: Engineering Reference" with its own narrative subsections (Affected Services, independent KG cross-check, pipeline trace prose) appears between Section 6 and the Revision Summary. This is a prose analysis appendix, not a sanctioned one-line metadata trailer and not the Revision Summary — it is exactly the "stray KG cross-check narrative appendix" the base rule targets. Flag and relocate to an external engineering doc. The trailing "## Revision Summary" is EXPECTED per spec-revise's template and is NOT flagged. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet-list only; data types are explicitly deferred to engineering. No tables or type declarations present. |

---

## 6. Recommendations

1. **Resolve DEC-001 and DEC-002 (CG-001).** Pin the same-day refresh cadence and the parity variance gate; without them the acceptance suite is not executable. Most critical — blocks all same-day FR testing.
2. **Re-ground the ingestion authority (CG-002).** Stop treating `event-manager-api` as the install-ingest/timing service given its CRUD-only interface contradicts that role; have the discovery trace empirically confirm the ingest hop (reconcile against the `inboard-*` cluster) before locking FR-001 scope.
3. **Add a discovery exit criterion for the KG discrepancy (Risk Audit FAIL).** Require the trace to confirm the affected-service set to a verifiable outcome before implementation scope is fixed — "reconcile both sets" is a deferral, not a mitigation.
4. **Define the reconciliation-window display behavior (EC-1, Risk Audit).** Specify what the dashboard shows and whether downward corrections are permitted between early publication and reconciliation.
5. **Specify Orion Mobile-unavailable and parallel-run-divergence behavior (EC-2, EC-11).** The baseline is a dependency; define authority when it is stale or when paths disagree during soak.
6. **Tighten the vague Then-clauses (FR-005, NFR-001, NFR-002, NFR-004)** with numeric bounds and cited baselines.
7. **Relocate the Engineering Reference appendix** out of the spec body to restore Section 0–6 compliance.
8. **Add FRs for the remaining edge cases (EC-3 through EC-10).**