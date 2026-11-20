# Spec Critique: K4A Android Install Reporting — Same-Day Sync

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-20

## Summary

- **Critical Issues:** 0
- **High Issues:** 6
- **Medium Issues:** 7
- **Blocking:** No

The spec is unusually disciplined for a change whose root cause is admittedly unlocated. It correctly refuses to presuppose a fix location, defines guardrails that hold regardless of where the lag lives, and surfaces the KG discrepancy rather than hiding it. There are no Critical gaps that block engineering from *starting* discovery. However, several target-behavior FRs rest on quantities the spec never bounds (the parity "variance," the reconciliation window for late-arriving events, the definition of "same-day" itself), and two decision points (DEC-001, DEC-002) are still open. These must be closed before the *implementation* phase, not before discovery.

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | FR-004 requires late-arriving events to be "reconciled such that the final same-day total matches Orion Mobile's total," but defines no reconciliation window. If K4A publishes same-day totals at hour N and Orion Mobile keeps accepting late events until hour N+K, the two systems will *necessarily* disagree at any instant between N and N+K — violating BR-1/FR-002. The spec treats parity as a point check but the accelerated path makes it a moving target. | High | §3 FR-004, §5 NFR-005 | Define the reconciliation cutoff and state whether parity is asserted only at a settled end-of-day checkpoint or continuously. Tie NFR-005's "agreed variance" to that window. |
| CG-002 | NFR-012 gates decommissioning on parity "within an agreed variance," but FR-002/NFR-005 demand "zero divergence" / "identical" totals. These are contradictory: one allows a tolerance band, the other allows none. Engineering cannot build a gate against two incompatible thresholds. | High | §5 NFR-005 vs NFR-012, §3 FR-002 | Reconcile: either parity is exact (variance = 0) at the settled checkpoint, or a named tolerance applies during soak only. State which. |
| CG-003 | "Same-day" (BR-2, NFR-001) is undefined pending DEC-001 — intraday continuous vs. single end-of-day refresh. Every freshness and parity FR (FR-001, FR-005, FR-006, NFR-009 latency alert) depends on this. The alert in NFR-009 ("exceeds the same-day freshness target") cannot be configured against an undefined target. | High | §1 BR-2, §5 NFR-001/NFR-009 | Resolve DEC-001 before implementation. The hourly working default is sensible but is explicitly provisional; the latency SLO must be a concrete number before alerts can exist. |

No issue here rises to data-loss/security Critical — FR-011/NFR-014 protect already-published data, and NFR-010/NFR-011 cover access and log safety. Hence Blocking = No.

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "...target intraday refresh at hourly aggregation granularity, which satisfies either interpretation." | "Satisfies either interpretation" is not a testable SLO; no max staleness bound is stated. | Given an Android install is recorded at time T, When the pipeline is operating normally, Then its count is reflected in the dashboard no later than T + 60 minutes, And the as-of timestamp advances at least once per hour. |
| NFR-004 | "...must not add perceptible load time." | "Perceptible" is subjective and untestable. | Given representative concurrent dashboard load, When the Android install view with the freshness indicator renders, Then p95 render time does not exceed the measured pre-change p95 baseline by more than [X] ms. |
| NFR-005 / NFR-012 | "...within an agreed variance" | The variance is never agreed in the spec, so no pass/fail can be coded. | Given the cutover soak, When K4A and Orion Mobile daily totals are compared at the settled checkpoint, Then absolute difference is 0 (or ≤ [agreed N installs]) for [M] consecutive days before decommission. |
| FR-010 / NFR-013 | "revert...within 5 minutes" (good) BUT "runs already in flight...complete under the prior behavior" | "In flight at the moment of the flip" is undefined at the boundary — a run that started 1s before the flip vs. a run mid-dispatch may be treated differently. Ambiguous for an idempotency/rollback test. | Add: Given a reporting run whose processing has begun but not committed at flip time, Then it is [allowed to complete | aborted and re-run], defining the exact cut line. |

Note: FR-001 through FR-012 are all authored in proper Gherkin (Given/When/Then). The vagueness above is concentrated in the NFR prose, which is acceptable as prose but still needs measurable thresholds.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard month-end deadline + unlocated root cause | Yes — NFR-012 discovery-first, BR-3 scope boundary | PASS | Good: explicitly surfaces the small-fix-vs-rearchitecture spread to the deadline owner. |
| Multi-service blast radius | Yes — NFR-002, NFR-003, FR-008, FR-009, NFR-013/014 | PASS | Shared-resource rate cap (NFR-003) with 80%/5min alert is concrete. |
| Unknown latency source | Yes — NFR-008 named failure conditions, NFR-012 per-hop trace | PASS | Distinct named conditions is the right call for locating the lag. |
| KG cross-check discrepancy (two lookups disagree on services) | Partially — discovery trace to reconcile both sets | PASS (with caveat) | Mitigation is "reconcile during discovery," which is appropriate, but see §4 edge cases: the trace has no stated exit criterion for *what counts as reconciled*. |
| Late-arriving-event accuracy vs. speed (BR-4) | Partially — FR-004, NFR-006 | FAIL | The risk is acknowledged but the mitigation is incomplete: no reconciliation window/cutoff is specified (see CG-001). A mitigation that cannot be measured is not a mitigation. |
| Accuracy of parity claim itself (BR-1) | Contradicted — NFR-005 "zero divergence" vs NFR-012 "agreed variance" | FAIL | The two mitigations for the same risk contradict each other (see CG-002). |

---

## 4. Missed Edge Cases

1. **Reconciliation window undefined:** Late events arriving after K4A's same-day publish but before Orion Mobile's settle create a guaranteed transient divergence (see CG-001). No FR states how long K4A keeps revising a published day.
2. **Day-boundary clock skew / DST:** FR-003 defers to Orion Mobile's convention, but no requirement covers what happens if Orion Mobile's and K4A's host clocks drift, or across a DST transition, where an install could land in two different days. "Follows Orion's convention" doesn't specify *re-derivation on each run* vs. a cached boundary.
3. **Orion Mobile unavailable during soak:** NFR-012's parity gate assumes Orion totals are queryable. No behavior defined if the comparison baseline itself is down during the soak window — does the gate fail open, fail closed, or stall the cutover?
4. **As-of timestamp staleness ceiling:** FR-006 shows last-known totals with a truthful timestamp indefinitely (flow node K). There is no upper bound at which an extremely stale "as-of" (e.g., >24h old again) escalates to an alert or a visibly degraded state. A truthful-but-ancient timestamp still erodes trust silently.
5. **Negative/corrected totals (install reversals/fraud clawbacks):** If an install is later invalidated upstream, same-day totals may need to *decrease*. No FR addresses whether same-day totals can move downward and how FR-011 ("already-published data remains intact") interacts with a legitimate downward correction.
6. **Partial-day cutover:** If the kill-switch is flipped mid-day (FR-010), the day's total is assembled partly by the accelerated path and partly by the old path. FR-012 idempotency covers retries of the *same* path, not a path change mid-day. Divergence risk unaddressed.
7. **Shared-resource cap backpressure:** NFR-003 caps the accelerated path's load, but if the cap throttles install reporting hard enough, same-day freshness (NFR-001) may be missed — the two NFRs can conflict under load and no priority/precedence is stated.
8. **Timezone of the freshness indicator display:** FR-005's "as-of" timestamp has no stated display timezone; a user in a different TZ than the reporting-day boundary could misread currency.
9. **Dashboard cache invalidation trigger:** FR-007 requires caches not to mask fresh data but defines no invalidation mechanism/TTL; "does not cause data to appear delayed" is an outcome, not a trigger.
10. **Multiple concurrent consumers mid-rollback (NFR-015):** Backward-compat says existing consumers must not break, but doesn't address a consumer reading mid-flip where format is identical but *timing semantics* changed — a consumer that polls expecting day-old data might now receive same-day data earlier than its own logic expects.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-012 are all well-formed Gherkin. NFRs are prose, which is expected for NFRs. |
| Document contains only Sections 0–6 | FAIL | The document ends with an **"Appendix: Engineering Reference"** containing a multi-paragraph narrative analysis — "Affected Services," "Independent KG cross-check," and a pipeline-shape walkthrough with its own findings prose. This is exactly the stray multi-paragraph KG cross-check analysis appendix that is *not* exempt (it is not a Revision Summary and not a one-line `> **Label:**` metadata trailer). Fold its grounding into Section 1's existing "KG Cross-Check Discrepancy" subsection or remove it; it should not be a standalone appendix beyond Sections 0–6. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list; data types/schema are explicitly deferred to engineering, with only business-level descriptors and source tags (System-derived/External), no attribute tables. |

**KG cross-check finding (critique-task grounding, not a format issue):** The spec's own Appendix honestly reports that the independent KG lookup did **not** re-surface the Context Seed's claimed services (k4a-sam-backend, reporting-dispatcher, cost-revenue-workers, frontend-mos, event-manager-api) and instead surfaced a different low-confidence cluster (inboard-*, rollup-*, krep*/summary-report-cloud, reporting-api). This matches the independent evidence provided to me — the spec does not contradict the cross-check; it correctly flags the disagreement as unresolved rather than picking a side. One concrete contradiction worth engineering's attention: the spec names `event-manager-api` as the "event ingestion pipeline" that "validates timing of Android install events," yet its own catalogued endpoints are all single-event CRUD (`DELETE/GET/PATCH /event/{id}`, `GET /events/list`) with **no bulk ingestion or timing endpoint** — the spec notes this honestly, but it weakens the claim that this service is on the high-volume install-ingest latency path. The discovery trace should treat `event-manager-api`'s role as unconfirmed, not assumed.

---

## 6. Recommendations

1. **Resolve DEC-001 (definition of "same-day")** before implementation — every freshness FR and the NFR-009 latency alert depend on a concrete SLO (CG-003).
2. **Reconcile the parity contradiction** between NFR-005 ("zero divergence") and NFR-012 ("agreed variance"), and define the **late-arriving-event reconciliation window** (CG-001, CG-002). State whether parity is exact at a settled checkpoint or tolerance-banded during soak only.
3. **Move the Engineering Reference Appendix content into Section 1's KG Cross-Check subsection** (or drop it) so the document stays within Sections 0–6 — this is the only format FAIL.
4. **Add edge-case FRs** for mid-day kill-switch cutover (partial-day totals), downward corrections/reversals, and the as-of-timestamp staleness ceiling (edge cases 4, 5, 6).
5. **Define precedence** between NFR-003 (shared-resource cap) and NFR-001 (same-day freshness) under contention (edge case 7).
6. **Specify the parity-gate behavior when Orion Mobile is unavailable** during the soak (edge case 3), and the freshness-indicator display timezone (edge case 8).
7. **Flag `event-manager-api`'s role as unconfirmed** for the discovery trace, given its endpoints show no bulk-ingest/timing surface.