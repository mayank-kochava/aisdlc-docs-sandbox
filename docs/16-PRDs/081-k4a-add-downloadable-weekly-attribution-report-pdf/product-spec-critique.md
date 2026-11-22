# Spec Critique: Brightway Retail Weekly Attribution PDF Report

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
| CG-001 | FR-008 requires the delivered file be "a valid PDF containing the last-quarter attribution data," but no FR defines behavior when a report was generated successfully but is later found corrupt/truncated at download time (e.g. storage partial read). FR-009 only covers "not generated / generation failed," not "generated-but-unreadable." Download path dead-ends. | High | 3 / FR-008, FR-009 | Add an FR for integrity validation at serve time, falling back to the report-unavailable state (FR-009) with a distinct reason if the stored artifact fails validation. |
| CG-002 | FR-011 serves via a time-limited secure link. No FR defines what happens when a user clicks an EXPIRED link (the exact `[DECISION REQUIRED]`-adjacent happy-path break). User hits a dead end with no defined re-request flow. | High | 3 / FR-011 | Add a Gherkin scenario: expired link → user is returned to the report area to re-initiate a fresh download, not a raw error. |
| CG-003 | The KG cross-check could not re-confirm `pubsuite-attribution` as the authoritative data source, yet FR-007/FR-002/FR-006 all depend on "the attribution data source" as a single known dependency. If the real source is `summary-report-cloud` or `aaas-reporting`, the retry/freshness semantics may not hold. This is routed to engineering but no FR is conditional on resolving it. | High | 1 KG note / 3 | Keep as design blocker but add an explicit acceptance gate: generation FRs are not testable until the authoritative source is confirmed. |
| CG-004 | NFR-009 kill-switch says in-flight requests "complete under the prior behavior." A download in flight when generation is killed is fine, but a generation run in flight when the switch flips has undefined artifact disposition (partial PDF persisted? FR-004 overwrite-in-place could leave a half-written report as "current"). | High | 5 / NFR-009, FR-004 | Define that a generation interrupted by kill-switch leaves the prior successful report as current and never promotes a partial artifact. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-006 | "the report applies the defined data-freshness handling" | "The defined handling" is undefined in the FR itself (deferred to a note). Not independently testable — a tester cannot assert on "the defined handling." | Given a week in the window has incomplete data, When the report is generated, Then that week is stamped with a data-freshness timestamp AND a visible warning label, And complete weeks carry no warning. (Bind the default into the FR, not a note.) |
| FR-001 | "the weekly generation cycle runs" | "Weekly" cadence trigger is unspecified (which day/time, which timezone) — directly collides with the unresolved Delivery model decision. Cannot write a deterministic scheduled test. | Given the configured weekly trigger time arrives in the account's reporting timezone, When the cycle runs, Then ... (parameterize trigger + timezone explicitly). |
| NFR-001 | "measured under expected concurrent K4A finance-user load" | "Expected concurrent load" is not quantified — p95 ≤ 3s is meaningless without a stated concurrency figure. | Given N concurrent authorized finance users (specify N), When each initiates a download of a pre-generated PDF, Then serving begins at p95 ≤ 3s. |
| NFR-002 | "conservative default that engineering can tune" | No numeric read-rate cap. Not testable as written; "conservative" is subjective. | State a concrete default cap value (e.g. ≤ X reads/sec) as the testable baseline, tunable thereafter. |

Note: All Section 3 items ARE in Gherkin — this is a prose/vagueness issue inside otherwise well-formed Gherkin (FR-006), not a format failure.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard 5-week year-end deadline | Yes — scope held to MVP, decisions carry working defaults | PASS | Reasonable; defaults keep engineering unblocked. |
| Multi-service blast radius | Yes — NFR-010 backward compat + NFR-009 kill-switch + rollback | PASS | Well covered. |
| Unresolved delivery architecture (on-demand vs scheduled) | Partial — FR-001–003 "hold for either model" | FAIL | Claiming FRs hold for both models hides a testability gap: a fixed-schedule model vs on-demand model produce different acceptance criteria (freshness, which week is "current"). Risk is acknowledged but no FR actually resolves the user-visible difference; it is deferred, not mitigated. |
| Data freshness / completeness uncertainty | Yes — FR-006 with default | PASS (with FR-006 testability caveat above) | Mitigation exists; wording needs tightening. |
| Authoritative data source unconfirmed (KG cross-check) | No | FAIL | The cross-check could not re-confirm `pubsuite-attribution`. No risk row or mitigation exists for "we may be building against the wrong data source." This is the single highest-value missing mitigation. Add a risk row + design gate. |
| Audit retention beyond 12 months for year-end financial close | Open | FAIL | NFR-008 flags retention as `[DECISION REQUIRED]` but a financial-close artifact with unresolved retention is a compliance risk with no mitigation/default. Supply a safe default (e.g. align to finance audit cycle) pending the decision. |

---

## 4. Missed Edge Cases

1. **Expired download link reuse:** No defined flow when an authorized user clicks a link after the validity window (see CG-002).
2. **Quarter boundary crossing mid-window:** When the calendar/fiscal quarter rolls over between two weekly cycles, the definition of "last quarter" shifts. No FR defines whether an in-flight or already-current report is recomputed or held. Directly tied to the unresolved Quarter window decision.
3. **Empty quarter / zero attribution data:** No FR covers a report where the last-quarter window legitimately contains no attribution rows (new account, no activity). Is an empty-but-valid PDF produced, or the unavailable state shown?
4. **Partial data-source failure (not total):** FR-007 handles the source being "unavailable or returns an error" wholesale. It does not cover the source returning data for SOME weeks only (partial success) — does generation proceed with freshness flags (FR-006) or fail?
5. **Plan downgrade/eligibility change mid-cycle:** BR-5 requires standard-plan. No FR covers an account losing standard-plan eligibility between generation and download — FR-010 re-derives auth per request, but is a previously generated report still served or blocked?
6. **Concurrent download of a report being overwritten:** FR-004 overwrites in place; FR-005 prevents overlapping generation but not a download racing an overwrite. User could download a mid-overwrite file.
7. **Timezone of "weekly" and "quarter":** No timezone is specified for either cadence or window boundaries. For a year-end financial close this is material.
8. **Audit-write failure during a successful download:** NFR-008 fires an alert on audit-write failure but does not say whether the download still succeeds (async, so presumably yes) — confirm the download is not retroactively invalidated.
9. **Kill-switch flipped mid-generation artifact disposition:** See CG-004.
10. **Retry-After exceeding the generation window:** NFR-004 honors Retry-After, but if Retry-After pushes retries past the weekly cycle, no FR defines whether the cycle is abandoned or deferred.
11. **Multiple finance users downloading simultaneously:** NFR-001 references concurrency but no FR confirms concurrent downloads of the same report are supported and all receive valid files.
12. **Reused scheduler collision (reporting-subscriptions-worker):** If the existing weekly-cadence worker is reused per KG note, its existing schedule may conflict with this feature's cadence — no FR/NFR guards against double-triggering via two schedulers.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-011 are all in `gherkin` fenced blocks with Given/When/Then. Explanatory `>` notes accompany some but the requirement bodies are Gherkin. |
| Document contains only Sections 0–6 | PASS | Sections 1–6 present. The trailing "Appendix: Engineering Reference" is the sanctioned KG-grounding reference material the spec explicitly marks as not-required-reading; it is the engineering reference appendix the spec's own Section 1 routes grounding to, not stray narrative prose, and is permitted. No duplicate or out-of-band sections found. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bullet-list business concepts with no attribute tables or type declarations; it explicitly defers types/storage to eng-plan. |

Note on the KG cross-check: the spec correctly surfaces the data-source discrepancy in Section 1 (KG cross-check note) rather than silently picking a side, consistent with the independent cross-check evidence (which likewise did not re-confirm `pubsuite-attribution` and surfaced `reporting-subscriptions-worker`, `summary-report-cloud`, `aaas-reporting` at low confidence). This is handled correctly — flagged as CG-003 / Risk FAIL only because no acceptance gate binds its resolution, not because the spec contradicts the evidence.

---

## 6. Recommendations

1. **Resolve the authoritative data source (CG-003 / Risk FAIL):** The generation FRs are untestable until it is confirmed whether the source is `pubsuite-attribution` or one of the low-confidence alternatives. Add a design gate.
2. **Define expired-link and corrupt-artifact fallbacks (CG-001, CG-002):** Add Gherkin scenarios routing both to the report-unavailable state / re-request flow so the download path has no dead ends.
3. **Specify kill-switch behavior for in-flight generation (CG-004):** Guarantee no partial artifact is ever promoted to "current."
4. **Bind FR-006's default handling into the FR body**, not a note, so it is independently testable.
5. **Quantify NFR-001 concurrency and NFR-002 read-rate cap** with concrete baseline numbers.
6. **Add quarter-rollover, empty-quarter, and partial-source-failure edge cases** (#2, #3, #4) as FRs — these are year-end-close-material.
7. **Pin timezones** for both the weekly cadence and the quarter-window boundaries (#7).
8. **Supply a safe default for audit retention** pending the NFR-008 decision, given the financial-close compliance context.
9. **Guard against dual scheduling** if `reporting-subscriptions-worker` is reused (#12).