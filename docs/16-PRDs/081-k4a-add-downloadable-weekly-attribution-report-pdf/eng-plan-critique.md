# Engineering Plan Critique: Weekly Last-Quarter Attribution PDF

## Part 1 — Coverage Matrix

| Spec Item | Covered By | Status |
|---|---|---|
| BR-1 | DEC-001/DEC-007 (on-demand model) in Decision Log; Confirmed Assumptions row 1 | covered |
| BR-2 | Corrected Assumptions "last quarter" row; Phase 2 quarter-window computation; QuarterWindow entity | covered |
| BR-3 | Generation worker `buildPDF` renderer (Affected Systems); FR-003 throughout | covered |
| BR-4 | Confirmed Assumptions "K4A download button" row; frontend-mos change | covered |
| BR-5 | Corrected Assumptions "standard-plan" row; reporting-api `feature_rollout` (optional data) | covered |
| BR-6 | Risks table (5-week deadline); overall phasing | covered |
| FR-001 | **See CG-001** — plan resolves to on-demand (no weekly cadence); "weekly generation cycle" not reconciled | deferred |
| FR-002 | Phase 2 quarter-window computation; QuarterWindow entity | covered |
| FR-003 | Generation worker `buildPDF` + logo embed; delivery_format `pdf` enum | covered |
| FR-004 | Corrected Assumptions final row (N/A under on-demand per DEC-001/DEC-007) | deferred |
| FR-005 | Corrected Assumptions final row (N/A under on-demand per DEC-001/DEC-007) | deferred |
| FR-006 | Dependencies/Open items — incomplete-week warning, flagged [DECISION REQUIRED]; Phase 3 "(if in scope)" | deferred |
| FR-007 | Generation worker "Named failure statuses" change; GenerationOutcome entity; Phase 3 | covered |
| FR-008 | Confirmed Assumptions download-button row; Phase 2/3 | covered |
| FR-009 | **See CG-002** — report-unavailable state with reason | gap |
| FR-010 | Corrected Assumptions "403" row; Phase 3 server-side enforcement; [DECISION REQUIRED] 403-vs-convention | deferred |
| FR-011 | Confirmed Assumptions signed-URL row; Phase 1 TTL override; GenerationOutcome entity | covered |
| NFR-001 | Technical Context Performance Goals (serve pre-generated only) | covered |
| NFR-002 | Generation worker "Read-rate cap" change; Phase 4 tuning | covered |
| NFR-003 | Generation worker "Generation monitoring" change; Phase 4 alerts | covered |
| NFR-004 | Generation worker "Named failure statuses + bounded retry" change; Phase 3 | covered |
| NFR-005 | Phase 3 server-side enforcement (same as FR-010) | deferred |
| NFR-006 | Phase 1 signed-URL TTL 15min; Confirmed Assumptions row | covered |
| NFR-007 | Technical Context Constraints (no sensitive data in logs) | covered |
| NFR-008 | Generation worker "Audit entries" change; Phase 3; [DECISION REQUIRED] audit sink | covered |
| NFR-009 | Phase 1 kill-switch config | covered |
| NFR-010 | Standards Check; Risks table; Phase 0/4 regression | covered |
| NFR-011 | **See CG-003** — maps to FR-004/FR-005, treated as N/A under on-demand | deferred |
| Rollback (HIGH-risk) | Risks table (partial); kill-switch (NFR-009) | covered |

## Part 2 — Findings

| ID | Finding |
|---|---|
| CG-001 | **FR-001 "weekly generation cadence" is not reconciled with the resolved on-demand model.** FR-001's gherkin explicitly requires "When the weekly generation cycle runs / Then a new attribution report is produced." The plan resolves the delivery model to pure on-demand (DEC-001/DEC-007), which structurally eliminates any weekly cycle. The plan handles the *consequences* (FR-004/FR-005 declared N/A) but never explicitly states that FR-001's weekly-cadence requirement is itself superseded/N/A. A reviewer reading FR-001 will find no trace that the plan knows it dropped the weekly trigger. This is the central product-vs-plan tension (the spec title itself says "Weekly") and should be an explicit Open Question to PM, not silently absorbed. Note the spec's own Risk Mitigation table flags the on-demand-vs-weekly decision and the possible reuse of `reporting-subscriptions-worker` — the plan rules that worker out (correctly, per investigation) but doesn't close the loop with the product owner that "weekly" is gone. |
| CG-002 | **FR-009 (no-report-available state) has no trace in the plan.** FR-009 requires the K4A UI to show a report-unavailable state communicating *why* (not-yet-generated vs generation-failed) and to offer no broken/empty download. The plan's only frontend change is `pdf?: boolean` on `OutputFormat`; there is no Affected-System entry, Decision, or Open Question covering the unavailable-state UI or the reason surfacing. Under the on-demand model the user triggers generation and polls, so an "unavailable with reason" state is arguably satisfied by the existing poll/error UX — but that reasoning appears nowhere. Either cite the existing CompletedReports/error-state handling as covering FR-009, or flag it. Currently it is a silent gap. |
| CG-003 | **NFR-011 / FR-004 / FR-005 declared N/A without PM sign-off, and the reasoning is self-referential.** The plan reasons: on-demand ⇒ no "week" key ⇒ idempotency/non-overlap N/A. That is internally sound, but these are explicit Phase-1 in-scope requirements per spec §6 and carry an NFR (NFR-011). Declaring three requirements out of scope is a product decision that should surface as an Open Question to PM, not be buried in a Corrected-Assumptions cell. Worth noting: even under on-demand, concurrent duplicate requests from the same user over the same window is a real scenario — "no overlapping generation runs" (FR-005) may still have a de-dup/debounce interpretation the plan dismisses too quickly. |
| CG-004 | **FR-006 incomplete-week data-quality warning is an unresolved blocking decision for Phase 1 content, but is treated as optional.** The spec lists FR-006 as in-scope Phase 1 (§6) with OQ-005 resolved (warnings *must* be shown). The plan marks it `[DECISION REQUIRED]`, Phase 3 "(if in scope)", and the data-model field is "if in Phase-1 scope." The plan also correctly notes a real tension: "most recent *completed* quarter" (FR-002) vs. per-week incompleteness (FR-006). This is a genuine contradiction between two in-scope requirements that the plan surfaces but does not resolve — it needs a PM ruling before Phase 2/3 content work, and should be a blocking Open Question, not a deferrable one, because it changes the report's data shape (`data_quality_warning` field) and the per-week granularity of the query. |
| CG-005 | **Revenue source + channel mapping is correctly identified as highest-risk but is a true Phase-2 blocker with no working default committed.** The plan flags this as the "single highest-risk unknown" and lists it as an Open Question/Dependency "Needed By Phase 2." Good. However, Phase 2's "channel-grouped revenue + conversions query" task *cannot start* until revenue definition (`event_metadata.price/.sum` vs `summary-report-cloud.event_revenue`), channel field (`partner` vs `network` vs `claim_traffic_type`), and whether existing data covers the last-quarter window are all resolved. This is a blocking gate on Phase 1 report content, not a routine deferral — the plan should state plainly that Phase 2 core content is blocked until Data/PM confirm, and that no working default exists (unlike the other decisions, which all have defaults). The data-model `channel` field already carries the `[DECISION REQUIRED]` forward, which is good. |
| CG-006 | **PDF library is unapproved — Phase 1 Standards BLOCKED item gating Phase 1/2.** The Dependency Standards checkbox is unchecked and marked `[DECISION REQUIRED]`; `maroto`/`gofpdf`/HTML→PDF must pass `/kochava:go-dependency-analysis`. Phase 1's first task ("Add PDF library to go.mod") and all of Phase 2's `buildPDF` work are gated on this approval. This is correctly surfaced in both Standards Check and Dependencies, with a working default (`maroto`) so it is not strictly blocking per the plan's "never blocked" strategy — but it is a real external approval dependency that should be started immediately (Needed By Phase 1). Acceptable as-is; noted for visibility. |
| CG-007 | **Generation-home decision gates Phase 0 and shapes most of the plan, correctly flagged as blocking.** `summary-report-cloud` vs `aaas-reporting` is an open decision "Needed By Phase 0," and Phase 0's test-backfill task is conditional on it (`If aaas-reporting: ...`). The two homes differ materially: `summary-report-cloud` has `event_revenue` (helps CG-005) but a 400-line CSV-coupled path; `aaas-reporting` has the cleanest semantic fit and format switch but *no revenue* and a hardcoded static auth map. This is a genuine Phase-0-blocking architectural decision and is correctly surfaced. Note the interaction with CG-005: picking `aaas-reporting` makes the revenue problem harder (no `event_revenue` to adapt). The plan should make explicit that generation-home and revenue-source decisions are coupled and should be decided together. |
| CG-008 | **403-vs-convention (FR-010/NFR-005) is a blocking Phase-3 decision with existing-client risk, correctly flagged.** Investigation found no clean 403 on the reporting-api path (200+`{status:Error}`); NFR-005 literally requires a 403. Matching "403" on that path is a behavior change risking existing clients (NFR-010). This is correctly surfaced as `[DECISION REQUIRED]`, a Risk, and an Open Question Needed By Phase 3. Acceptable as-is; the plan should ensure the PM ruling clarifies whether "server-side denial in any form" satisfies NFR-005's literal "403." |
| CG-009 | **Rollback is HIGH-risk/multi-service per spec, but the Risks table has no dedicated rollback row and no explicit multi-service coordination risk.** The spec §5 contains a dedicated HIGH-risk Rollback subsection (trigger, owning authority, timing, data treatment). The plan covers the mechanism (kill-switch NFR-009, Phase 1) but the Risks & Mitigations table has no row for the multi-service rollback coordination itself, nor for the "incorrect report content reported by finance" trigger. For a HIGH-risk multi-service change this should be an explicit risk row. Minor relative to CG-001/004/005 but worth adding for completeness given the spec explicitly elevated it.|

## Blocking Verdict

**Not ready to proceed to G2 as-is.** The plan is unusually thorough, evidence-backed (investigation-verified citations throughout), and honest about its unknowns — but it carries three issues that must be addressed first:

1. **CG-001 (blocking):** The product-vs-plan "weekly" contradiction must be raised explicitly with PM. The spec is titled "Weekly," FR-001 mandates a weekly cadence, and the plan silently resolves to pure on-demand. This is the single largest unflagged divergence and cascades into FR-004/FR-005/NFR-011 being declared N/A.

2. **CG-002 (blocking gap):** FR-009 (report-unavailable state with reason) has zero trace anywhere — neither covered nor deferred. Add an Affected-System/Decision entry (even if just "satisfied by existing poll/error UX") or an Open Question.

3. **CG-004 (blocking decision):** FR-006 is in-scope Phase 1 per spec with OQ-005 resolved, but the plan treats it as optional/"if in scope" and surfaces a genuine FR-002-vs-FR-006 contradiction it does not resolve. Needs a PM ruling before Phase 2/3 content work; it changes the report's data shape.

CG-005 and CG-007 (revenue source + generation home) are correctly identified as the critical-path engineering blockers and are coupled — they should be decided together before Phase 0/2 begin, but they are appropriately flagged and do not require rework of the plan document itself. CG-003, CG-006, CG-008, CG-009 are minor/already-flagged and should be tightened but are not independently blocking.

| CG-010 | **Mitigation doesn't cover every named repo** — Risk ""403" requirement forces a behavior change on reporting-api path" names ['reporting-api'], but its Mitigation only covers (none) — reporting-api left with no stated remediation. |

| CG-011 | **Mitigation doesn't cover every named repo** — Risk "Logo is PNG/JPEG only (no SVG); renderer dependency on mos-iam-server" names ['mos-iam-server'], but its Mitigation only covers (none) — mos-iam-server left with no stated remediation. |