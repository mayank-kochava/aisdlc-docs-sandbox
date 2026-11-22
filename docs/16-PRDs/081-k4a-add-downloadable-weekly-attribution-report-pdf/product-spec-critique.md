# Spec Critique: Brightway Retail Weekly Attribution Report (K4A)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-22

## Summary

- **Critical Issues:** 2
- **High Issues:** 6
- **Medium Issues:** 5
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Delivery model self-contradiction between Section 1 and Section 3.** Section 1 (BR-1, Delivery model) states the PDF is "generated fresh on demand at the moment the finance team downloads it" and is "not pre-generated or stored ahead on a schedule." But FR-001, FR-004, FR-005, FR-008, FR-009, and NFR-001 all describe a **weekly system-triggered generation cycle** that produces a stored "current report" served without synchronous generation on the download path ("Serving a pre-generated file must not trigger synchronous report generation"). These are two mutually exclusive architectures. Engineering cannot build both. This is the core logic of the feature and it is unresolved. | Critical | 1, 3, 5 | Pick ONE model. If on-demand (per BR-1), delete FR-001/FR-004/FR-005 and the weekly-cadence framing, and rewrite NFR-001 to permit the "minute or two" generation latency Section 1 explicitly allows. If pre-generated weekly (per FR-001), fix BR-1 and the entire "Delivery model" subsection. The Risk Mitigation table's claim that "FR-001–FR-003 are written to hold for either model" is false — FR-001 hard-codes a weekly cycle. |
| CG-002 | **Authoritative data source is unverified and the independent KG lookup actively contradicts it.** FR-007, NFR-002, and the Information Requirements all depend on "the attribution data source" (`pubsuite-attribution`). The independent KG cross-check did NOT re-confirm `pubsuite-attribution` as a data source. The spec routes this to "engineering to verify during design" — meaning the entire failure-handling, rate-protection, and data-freshness logic is built on an unconfirmed dependency. If the real source differs, FR-006/FR-007/NFR-002 may need to be rewritten. | Critical | 1, 3, 5, Appendix | Reconcile the data source before engineering starts. A requirement set whose central external dependency is "unconfirmed, verify later" is not implementable with confidence. At minimum, FR-007/NFR-002 must name the verified source or be explicitly gated on that verification. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| FR-006 | "the report applies the defined data-freshness handling for affected weeks" | "Defined data-freshness handling" is indirection to a prose Note. The FR body itself is not independently testable — a tester cannot assert against "the defined handling." | `Given one or more weeks in the last-quarter window have incomplete or still-processing attribution data at generation time, When the report is generated, Then each affected week's row displays a visible data-quality warning label, And complete weeks display no such warning, And no affected week is omitted from the report.` |
| FR-008 | "a current weekly attribution PDF has been generated" | In the on-demand model (BR-1) there is no pre-"generated current" PDF — this precondition only holds under the pre-generated model, exposing CG-001 at the test level. Also "current" is undefined if generation is stale/failed. | Resolve CG-001 first. If on-demand: `Given an authorized finance user requests the report, When generation for the latest completed quarter succeeds within the acceptable latency window, Then a valid PDF for that window is delivered.` |
| NFR-002 | "cap its read load … conservative default that engineering can tune" | No concrete, testable cap value. "Conservative default" is not measurable. Acceptable to defer the number, but the requirement as written has no pass/fail condition for QA. | `Given report generation is reading from the attribution data source, When measured over any 1-minute window, Then generation read throughput must not exceed the configured cap of N req/s (default value TBD-by-eng, MUST be a concrete config value before release).` |
| NFR-006 | "default to a short validity window (e.g. 15 minutes)" | "e.g." makes the 15-minute figure illustrative, not binding → untestable. | `Given a secure download link is issued, When 15 minutes (configurable) have elapsed since issuance, Then any request using that link is rejected as expired.` |
| BR-6 / Value | "available before year-end close (~5 weeks from request)" | A delivery-deadline, not a product requirement — not testable as system behavior, and the "~" is imprecise. Acceptable as context but should not masquerade as a rule. | N/A — reclassify as a project constraint, not a Business Rule. |

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Hard 5-week deadline | Yes — scope held to Phase-1 MVP; decisions flagged with defaults | PASS | Reasonable. |
| Multi-service backend + frontend blast radius | Yes — NFR-010 backward compat + NFR-009 kill-switch + rollback plan | PASS | Well covered. |
| Unresolved delivery architecture (on-demand vs weekly) | Claimed resolved/routed, but **FRs contradict the stated model** | **FAIL** | The table claims "FR-001–FR-003 hold for either model." FR-001 explicitly encodes a weekly cycle; BR-1 explicitly forbids pre-generation. This risk is NOT mitigated — it is unresolved and the spec is internally contradictory (see CG-001). |
| Ambiguous "last quarter" window | Yes — most recent completed calendar quarter | PASS | Clear. |
| Data freshness / completeness uncertainty | Yes — FR-006 + warning | PASS (with FR-006 testability caveat) | OK. |
| Shared data-source dependency (named services may already provide scheduling / overlapping generation) | Routed to eng design, no binding requirement | **FAIL** | KG surfaced `reporting-subscriptions-worker` (existing weekly cadence) and `summary-report-cloud` (overlapping generation). The spec notes these but defines NO requirement preventing duplicate/conflicting generation across services. Given FR-005 (no overlapping runs) only governs *this* feature's runs, a cross-service collision is unmitigated. |
| Unconfirmed authoritative data source | Noted, deferred to eng | **FAIL** | See CG-002. A noted-but-unmitigated dependency risk at the center of the feature. |

---

## 4. Missed Edge Cases

1. **First quarter / no prior data:** FR-002 computes "most recent completed calendar quarter." What happens if the account was onboarded mid-quarter and has zero attribution data for the window? Empty table vs. error vs. "no data" state is undefined.
2. **Quarter boundary race:** Generation computed "at the moment of generation." If generation spans a quarter boundary (e.g., runs at 23:59 on the last day of a quarter), which quarter is reported? FR-002 says "computed at generation" but doesn't fix the boundary.
3. **User clicks "download" while generation in progress (on-demand model):** Section 1 mandates an in-progress state, but NO FR defines behavior when a second download request arrives during generation, or when the user navigates away ("Back") mid-generation.
4. **Download interrupted / resumed:** FR-008 ends at "delivered." Partial download, network drop, and whether the time-limited link (FR-011) permits a retry within its window are undefined.
5. **Link expiry mid-download:** NFR-006 says link is "unusable after its validity window." If a large PDF download is in flight when the 15-min window lapses, is the in-flight transfer killed? Undefined.
6. **Generation succeeds but PDF is corrupt/zero-byte:** FR-009 covers "generation failed," but a run that reports success yet produces an invalid PDF is not addressed. FR-008 asserts "valid PDF" but defines no validation gate.
7. **All 3 retries fail AND no prior report exists:** FR-007 preserves "the prior successfully generated report (if any)." If there is no prior report (first run fails), the user state is FR-009's "unavailable" — but the hand-off between FR-007 failure and FR-009 state is implicit, not specified.
8. **Partial data-source response:** FR-007 handles "unavailable or returns an error." A source that returns a 200 with truncated/partial data (not an error) is not covered and would silently produce a wrong report.
9. **Authorization changes between link issuance and download:** FR-010 re-derives auth "on every request," but a time-limited link (FR-011) could be reused after the user loses permission. Is auth re-checked on the link-redemption request specifically? Not stated.
10. **Kill-switch flipped mid-generation:** NFR-009 says in-flight requests "complete under prior behavior," but a long-running generation (minutes) during an incident is exactly what an operator wants to stop. Conflict between NFR-009's in-flight guarantee and the rollback trigger intent.
11. **Standard-plan downgrade/eligibility change:** BR-5 requires standard plan. If Brightway downgrades mid-cycle, is the current report revoked? Undefined.
12. **Concurrent downloads by multiple finance users:** NFR-001 mentions "concurrent K4A finance-user load" but no FR defines multi-user concurrent download behavior or per-user link isolation.
13. **Audit-write failure path (NFR-008):** Fires an alert but "must not block the primary operation" — so a report can be downloaded with NO audit trail. For a financial-close artifact this may be an unacceptable compliance gap; no FR reconciles this.
14. **Timezone of "calendar quarter":** Quarter boundaries are timezone-dependent. No timezone is specified for the window computation (Brightway's TZ vs. UTC vs. K4A's).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-011 are all in Gherkin code blocks. Supplementary `> Note` lines are acceptable annotations, not prose FRs. |
| Document contains only Sections 0–6 | **FAIL** | The document contains an **"Appendix: Engineering Reference"** with multi-paragraph narrative content ("Affected Services (grounded context)", confidence-level discussion, cross-check analysis). This is a stray appendix beyond Sections 0–6 and is NOT the sanctioned trailing Revision Summary (which also exists and is correctly placed last). A multi-paragraph KG-analysis appendix with its own findings is exactly the kind of extra content the format rule catches. The trailing `## Revision Summary` itself is EXPECTED and is NOT flagged. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is a pure bullet list of business concepts with no attribute tables or type definitions. |

> Note on the Appendix: It should be removed from the spec and relocated to eng-plan. The spec even labels it "not required reading for product/requirement review," which is a tacit admission it does not belong in the product spec.

---

## 6. Recommendations

1. **Resolve CG-001 (delivery model contradiction) immediately.** Choose on-demand (BR-1) OR pre-generated weekly (FR-001) and make Section 1, Section 3, and NFR-001 internally consistent. This blocks all download/generation requirements and is the single highest-priority fix.
2. **Resolve CG-002 (unverified data source).** Confirm the authoritative attribution data source before engineering; FR-006/FR-007/NFR-002 all hang on it.
3. **Define cross-service generation collision handling.** The KG surfaced `reporting-subscriptions-worker` (existing weekly cadence) and `summary-report-cloud` (overlapping generation). Add a requirement ensuring no duplicate/conflicting generation across existing services, or explicitly mandate reuse of the existing scheduler.
4. **Remove the "Appendix: Engineering Reference"** from the product spec (format violation) and relocate to eng-plan.
5. **Make FR-006 self-contained and testable** (fold the warning behavior into the Gherkin), and convert NFR-002/NFR-006's "e.g." / "conservative default" language into concrete config-backed, testable values (value may be TBD but the requirement must assert against a configured value).
6. **Add FRs for the missed edge cases**, prioritizing: in-flight download/link expiry (CG-adjacent), corrupt-but-"successful" PDF validation, auth re-check at link redemption, audit-write-failure vs. compliance, and quarter-boundary/timezone definition.
7. **Reclassify BR-6 (5-week deadline)** as a project constraint rather than a testable Business Rule.