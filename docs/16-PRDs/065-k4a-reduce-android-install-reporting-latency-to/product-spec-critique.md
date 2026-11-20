# Spec Critique: K4A Android Install Reporting Same-Day Sync

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
| CG-001 | **Core behavior gated on two unresolved DECISION REQUIRED flags.** BR-2/NFR-001 (DEC-001) leave "same-day" undefined (continuous intraday vs single end-of-day refresh), and NFR-012 (DEC-002) leaves the parity variance gate undefined. The headline acceptance criterion (FR-001) is therefore untestable as written — "same calendar day" could pass at 23:59 or require hourly refresh. Engineering cannot size, build, or verify against an undecided target. | Critical | 1, 5 | Resolve DEC-001 to a concrete refresh cadence and DEC-002 to a numeric parity variance (e.g. 0 rows or ±N) before engineering starts. Until then this is a blocking gap, not a working default. | 
| CG-002 | **Evidence contradicts a named mitigation dependency.** FR-003 / FR-004 / BR-4 depend on attributing events to a reporting day and reconciling late-arriving/deduplicated events via the ingestion pipeline. The spec names `event-manager-api` as "event ingestion pipeline," but the KG evidence shows it exposes ONLY single-event CRUD endpoints (`DELETE /event/delete`, `DELETE /event/{id}`, `GET /event/{id}`, `GET /events/list`, `PATCH /event/{id}`) — no bulk ingestion or timing endpoint. The spec itself admits this parenthetically yet still relies on this service for timing/attribution. The actual ingest path per the fresh KG lookup is a different, unvalidated cluster (`inboard-*`). Building FR-003/FR-004 against `event-manager-api` would fail; the true attribution/reconciliation surface is unidentified. | Critical | 1, 4 (Appendix) | Do not treat `event-manager-api` as the ingestion/timing authority. The discovery trace (NFR-012) must first identify the real ingest/reconciliation service before FR-003/FR-004 can be implemented or tested. Flag that no bulk-ingest endpoint exists on the named service. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-001 | "...reflected in the K4A dashboard on that same calendar day" | "Same calendar day" has no cutoff until DEC-001 is resolved — not measurable. | `Given an Android install recorded at 00:00 on day D` `When the pipeline processes day D under normal operation` `Then the K4A dashboard reflects that install no later than <agreed cadence, e.g. D 23:59 in Orion Mobile's timezone, or within N hours of record>` |
| FR-002 | "...the totals are identical" | No tolerance defined for the comparison window; "identical" at which point in the day? Conflicts with same-day-in-progress totals that legitimately differ until end of day. | `Given both dashboards have completed their day-D reporting cycle` `When day-D totals are compared after the agreed parity point` `Then K4A's day-D total equals Orion Mobile's day-D total exactly` |
| FR-005 | "...reflecting the actual recency of the displayed data" / "the most recent point to which same-day data has been loaded" | "Actual recency" not bounded — how stale can the as-of be before it is a defect? No max skew between as-of and wall clock. | `Given the dashboard shows an as-of timestamp T` `When compared to the latest data the serving layer holds` `Then T equals the timestamp of the newest loaded datapoint, and T is never shown as more recent than the actual data` |
| NFR-001 | "...target intraday refresh at hourly aggregation granularity, which satisfies either interpretation" | A "working default" for an undecided requirement is not a testable acceptance criterion; it pre-commits engineering to a cadence the business hasn't agreed. | Resolve DEC-001; then: `Given same-day data for day D` `When the hourly refresh runs` `Then dashboard data is no more than 60 minutes behind the serving layer` |
| NFR-004 | "...must not add perceptible load time" | "Perceptible" is subjective; the row then gives a measurable p95 rule — the vague clause should be deleted. | `Given representative concurrent dashboard load` `When the Android install view with the freshness indicator renders` `Then p95 render time does not exceed the current measured baseline` |

All FRs (FR-001 through FR-012) are written in Gherkin — no prose FRs. The vagueness above is in the acceptance *content*, not the format.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard month-end deadline + unlocated root cause | Yes — NFR-012 (discovery trace first), BR-3 (scope boundary) | PASS | Mitigation is a real requirement, not just a listed risk. Caveat: discovery outcome could blow the deadline; spec acknowledges the spread must be surfaced to the deadline owner. |
| Multi-service blast radius | Yes — NFR-002, NFR-003, FR-008, FR-009, NFR-013, NFR-014 | PASS | Collateral-impact and kill-switch requirements are concrete. |
| Unknown latency source | Yes — NFR-008 (named failure conditions), NFR-012 (per-hop timing) | PASS | Observability requirements exist to locate the lag. |
| KG cross-check discrepancy (two lookups disagree on affected services) | Partial — "discovery trace reconciles both sets" | FAIL | The mitigation is a hand-wave: there is no requirement defining *how* reconciliation is decided, what evidence closes the discrepancy, or which set gates the cutover. CG-002 shows one claimed service (`event-manager-api`) is contradicted by its own endpoint evidence — the spec surfaces this but does not require it be resolved before build. Add a requirement: the discovery trace must produce a single reconciled, evidence-backed service map before implementation scope is fixed. |
| Accuracy sacrificed for speed (late-arriving events / dedup) | Yes — FR-004, NFR-005, NFR-006 | PASS | But see EC list — behavior when late events arrive *after* the day is closed/published is underspecified. |
| Accelerated path misbehaves in prod | Yes — FR-010/011/012, NFR-013/014 | PASS | Kill-switch, idempotency, and data-preservation are concrete and testable. |

---

## 4. Missed Edge Cases

1. **Late-arriving events after same-day publication:** FR-004 says late events are "reconciled such that the final same-day total matches Orion Mobile." But what if Orion Mobile *also* only reconciles later? The window during which K4A and Orion Mobile legitimately disagree (both still settling) is undefined — FR-002's "identical" would falsely fail during this window.
2. **Orion Mobile is itself delayed or down:** Every parity requirement (FR-002, NFR-005, NFR-012) assumes Orion Mobile is the available, correct baseline. No behavior defined if Orion Mobile's own numbers are late, wrong, or unreachable during the parity check/soak.
3. **Day-boundary / DST transition:** FR-003 aligns to Orion Mobile's convention, but behavior across a DST shift or if Orion Mobile changes its convention is undefined — installs near the boundary on a 23- or 25-hour day.
4. **Backfill / historical correction:** If upstream corrects a prior day's totals, does K4A's same-day path propagate the correction? FR-011 says published data is not retroactively altered on rollback — but legitimate upstream corrections are a different case and are unaddressed.
5. **Parallel-run divergence during soak (NFR-012):** If old and new paths disagree during the parallel run, which is shown to users? No requirement defines the user-facing source of truth during cutover.
6. **Kill-switch flip mid-day (FR-010):** Runs in flight complete under prior behavior, new runs revert — but the dashboard could then show a mix of same-day and prior-day totals within one calendar day. User-facing consistency during the 5-minute transition is undefined.
7. **Freshness indicator when zero installs vs no data:** A legitimate zero-install day is indistinguishable from "data not yet loaded" in the current flow. FR-006 covers stale data but not the zero-vs-null ambiguity.
8. **Shared-resource rate cap (NFR-003) throttles install reporting itself:** The cap protects unrelated reporting, but if it throttles the accelerated path, same-day target (FR-001) may silently miss. No requirement defines precedence when the cap and the freshness target conflict.
9. **Partial-day / mid-processing dashboard load:** User loads the dashboard while aggregation for the current hour is in progress — is a partial total shown with an as-of timestamp, or suppressed? FR-007 covers cache staleness but not partial-aggregation reads.
10. **Idempotency key collision across reporting days (FR-012/NFR-006):** Idempotency is keyed on "reporting-day key." If a reprocessing run spans a day boundary or carries a mis-attributed key, the upsert could overwrite the wrong day. Key-derivation correctness is assumed, not required.
11. **Access-control re-derivation under caching (NFR-010 vs FR-007):** NFR-010 requires per-request server-side access checks; FR-007 permits display-layer caching. A cached view could serve data to a role whose access changed between cache fill and read. Interaction is unspecified.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-012 are all in Gherkin fenced blocks. |
| Document contains only Sections 0–6 | FAIL | The document ends with an **"## Appendix: Engineering Reference"** containing multi-paragraph narrative analysis (affected-services lists, KG cross-check prose, pipeline-shape discussion). This is NOT a sanctioned Revision Summary trailer nor a short one-line `> **Label:** ...` metadata blockquote — it is exactly the "multi-paragraph narrative KG cross-check appendix" the format rule is written to catch. Section 1's "KG Cross-Check Discrepancy" subsection also pushes KG-reconciliation detail into the body; acceptable in-section, but the standalone Appendix must be removed or relocated out of the spec. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a bullet list described at business level; it names no data types/schemas (the "*System-derived*/*External*" tags are sourcing annotations, not type declarations). No attribute tables present. |

---

## 6. Recommendations

1. **Resolve CG-001 (DEC-001 and DEC-002) before any engineering.** Pin "same-day" to a concrete cadence and define the numeric parity variance gate. The entire acceptance suite is untestable until this is fixed. (Blocking)
2. **Resolve CG-002: stop treating `event-manager-api` as the ingestion/timing authority.** Its endpoints (per KG evidence) are single-event CRUD only — no bulk ingest/timing surface. Make the discovery trace identify and validate the real ingest/reconciliation service before FR-003/FR-004 are built. (Blocking)
3. **Add a concrete reconciliation requirement for the KG service-set discrepancy** (Risk Audit FAIL): define what evidence closes the discrepancy and that a single reconciled service map gates implementation scope.
4. **Define behavior when Orion Mobile (the parity baseline) is late/wrong/unavailable**, and define the legitimate "both still settling" window so FR-002 doesn't false-fail (EC-1, EC-2).
5. **Remove or relocate the "Appendix: Engineering Reference"** so the document contains only Sections 0–6 (plus any sanctioned Revision Summary / metadata trailer).
6. **Specify user-facing source-of-truth and consistency rules** during parallel-run soak (EC-5) and during the kill-switch 5-minute transition (EC-6).
7. **Tighten vague acceptance clauses** in FR-005, NFR-001, NFR-004 per Section 2 rewrites; delete "perceptible" and the "working default" cadence once DEC-001 lands.
8. **Address the remaining edge cases** (zero-vs-null freshness, rate-cap vs freshness precedence, cache-vs-access-control interaction, idempotency key derivation).