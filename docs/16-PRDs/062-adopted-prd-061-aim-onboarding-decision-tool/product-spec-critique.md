# Spec Critique: AIM Onboarding Decision Tool (Part 1)

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-27

## Summary

- **Critical Issues:** 3
- **High Issues:** 7
- **Medium Issues:** 8
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Approval anchoring vs. Timeline is circular / unreachable in client self-serve.** Flow A gates final SoW approval behind "CSM activates CSM Approval control" (step P) which only happens after an offline meeting (step O), which only fires after "Book my onboarding meeting" (step M). But FR-020 says approval is what unlocks/anchors the Timeline, and NFR-002/AIM X's whole premise is *zero-touch, no per-client CSM effort*. The AIM X "zero-touch" path structurally **cannot reach approval without a CSM**, directly contradicting the stated AIM X mandate (Risk Mitigation bullet 2). Either AIM X approval does not require CSM activation (then FR-030 must carve out AIM X), or AIM X is not actually zero-touch. The spec asserts both and defines neither resolution. | Critical | FR-020, FR-030, Flow A, §1 Business Rules | Define explicitly whether AIM X approval is self-serve (no CSM Approval gate) or whether AIM X is not zero-touch. Add an FR covering the AIM X approval path. |
| CG-002 | **Integration ordering with existing onboarding-init is unresolved AND load-bearing.** DEC-001 (whether the tool runs before/during/after `aim-customers-onboarding-initialization`) is flagged but not decided. NFR-011 requires existing downstream steps to "continue to function unchanged" while admitting the integration point is unknown. You cannot write a regression test, nor guarantee backward compatibility, against an unknown integration point. This blocks implementation of the single highest-regression-risk service. | Critical | §1 Scope note, DEC-001, NFR-011 | Resolve DEC-001 before engineering. No FR currently specifies ordering behaviour; one must be added. |
| CG-003 | **Audit-write failure behaviour is undefined for the blocking case.** NFR-007/FR-024 require an append-only immutable Change Log and say an audit-write failure "must surface an operational signal rather than fail silently" — but never say whether the *user's milestone action itself* is allowed to commit when the audit write fails. If the status change persists but the audit entry does not, the immutability guarantee is violated and you have silent data divergence. This is a data-integrity gap. | Critical | FR-024, NFR-007 | Specify fail-closed semantics: if the audit entry cannot be written, the milestone mutation must be rejected/rolled back, not committed. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-002 | "...must update **without perceptible lag** on each state change." | "Perceptible lag" is subjective and not measurable. | Given a user changes any answer, When the live Setup Summary recomputes, Then the DOM reflects the new value within 100ms and input focus is never blocked during recompute. |
| FR-006 | "...reflects the change **in real time**..." | "Real time" is unbounded; no threshold. | Given a user changes a summary-affecting answer on steps 2–6, When the change is committed, Then the Setup Summary panel reflects it within 100ms. |
| NFR-010 | "...must **degrade gracefully** with a clear message when offline..." | "Gracefully" is not testable; what state is the action left in? | Given the user triggers notification/approval while offline, When the network call cannot complete, Then the action is not marked complete, a retry-on-reconnect message is shown, and no partial state is persisted. |
| FR-018 | "...a **rationale** for the recommendation is shown to the user" | "A rationale" is unverifiable — which rule(s) fired must be asserted. | Given tier is computed, When AIM Pro is recommended, Then the displayed rationale names each specific trigger rule that evaluated true (spend+history / data-source count / UA-UE flag). |
| NFR-001 | "...under 500ms for all client-side computation" | Testable, but "all" is undefined for the largest input (max platforms × max regions × max funnel events). | Given the maximum valid input set (3 platforms, all funnel events, max regions), When the output screen computes, Then tier + schema + timeline complete within 500ms. |

*All Section 3 FRs are already in Gherkin; no prose FRs were found. Issues above are threshold/measurability gaps, not format violations.*

---

## 3. Risk Mitigation Audit

| Risk (from spec §1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Scope creep (core problem) | Yes — FR-019, FR-020 lock on approval | PASS | Deterministic SoW + lock is a real requirement. |
| AIM X zero-touch breaking under manual effort | Stated as mitigated, but contradicted | **FAIL** | See CG-001. The approval flow *requires* a CSM for every client, which is itself the regression this risk warns against. The mitigation asserts a constraint the FRs violate. |
| Brand ad-stock / mixed-network ambiguity | Yes — FR-019 flags 30-day ad-stock + unconfirmed mixed-network | PASS | Concrete FR backs it. |
| Commercial-confidential data exposure | Yes — NFR-003 | PASS | But see EC-9: no requirement covers the PDF/CSV downloads (FR-021/026) containing the same confidential data — exposure surface is incompletely scoped. |
| Integration ordering with existing onboarding-init | No — flagged only, DEC-001 open | **FAIL** | "Mitigated once it is settled" is not a mitigation; it is a deferred decision. No FR defines the integration point. See CG-002. |
| Unconfirmed downstream consumer (attribution-api) | Partial — "treat output as contract if confirmed" | **FAIL (conditional)** | No requirement defines the output contract shape, and no guard prevents accidental coupling. Mitigation is aspirational, not specified. |
| Audit-write failure | Partial — NFR-007 says "surface a signal" | **FAIL** | Does not specify whether the triggering mutation commits. See CG-003. |

---

## 4. Missed Edge Cases

1. **AIM X approval deadlock (zero-touch):** As CG-001 — a self-serve client has no CSM to activate the CSM Approval control, so the Timeline can never be anchored. No FR covers a CSM-less approval path.
2. **Conversion-share allocator rounding:** FR-008 requires per-platform shares total 100% with proportional redistribution for 3 platforms. Proportional redistribution of integers routinely yields 99% or 101%. No rule defines rounding/residual assignment.
3. **Budget-split warning vs. hard block:** FR-009 warns when a campaign type is <5% or >90% but never blocks. A 100%-single-type split can still satisfy "both UA and UE present at mid share" never firing — fine — but the interaction between warning-only shares and the tier flag's 10–90% band is untested for boundary values (exactly 10%, exactly 90%).
4. **Tier flag boundary ambiguity:** FR-010/FR-018 say "between 10% and 90%" — is the band inclusive or exclusive at the endpoints? A client at exactly 10% UA is undefined. Same for the $250K spend threshold (≥ vs >) — spec says "at least $250K" (inclusive) but history band "12–24 month band or longer" overlaps the 12-month boundary ambiguously.
5. **Manual downgrade then answer change:** FR-018 allows AIM Pro → AIM X manual downgrade. If the user then edits an answer that re-triggers an AIM Pro auto-recommendation, does the manual downgrade persist or get overwritten? Undefined.
6. **Unlock-to-edit after PDF download:** FR-021 PDF includes approver details. FR-020 "Unlock to edit" resets approval. A PDF already downloaded with approval details is now stale/orphaned in the wild. No requirement addresses already-issued approved artifacts.
7. **Business-model change destroying KPI:** FR-012 resets the funnel on business-model change. If the user had set a Primary KPI and LTV config, those are silently destroyed. No warning/confirmation required before destructive reset.
8. **"Same as project lead" sync then unsync:** FR-007 syncs data-lead to project-lead in real time. If the user edits project lead *after* syncing, then unchecks — what is the data-lead state? Undefined (stale synced value vs. empty).
9. **Concurrent CSM edits to one onboarding:** NFR-006 guarantees session isolation, but FR-031's region dropdown and CSM-assisted review imply two CSMs could touch the same onboarding. Last-write-wins vs. conflict is undefined for shared (non-isolated) onboarding records in production.
10. **Timeline delay with negative/past date:** FR-023 shifts subsequent milestones when a delay date is saved. Entering a date *earlier* than the prior milestone (negative delay) is not handled.
11. **"Other/none" MMP injecting milestone + later changed:** FR-013/FR-022 inject a "to be confirmed" milestone for Other/none. If the user later selects a real MMP, is the injected milestone removed? Undefined.
12. **Offline detection mid-submission:** NFR-010 covers offline at action time, but not connectivity dropping *during* the notification/approval network call (partial send).
13. **Super Admin loses role mid-session:** FR-027 re-derives role per request (good), but the UI may still show CSM-only guidance notes from a cached render; no requirement forces client-mode UI downgrade on role loss.
14. **Max funnel events / performance ceiling:** No upper bound on funnel events or lead rows (FR-007 allows unlimited add), feeding NFR-001's "all computation under 500ms" with an unbounded input.
15. **DEC-002 uncollected state-shape fields:** Output tabs reference fields (channels, campaign-merge, no-LTA channels) with no Phase 1 input. Output generation against missing inputs has undefined behaviour (blank vs. error vs. omit).

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | FR-001 through FR-031 are all in Gherkin. No prose FRs. |
| Document contains only Sections 0–6 | **FAIL** | The document ends with a multi-paragraph **"Appendix: Engineering Reference"** containing an *Affected Services* narrative list and a *Reconciliation note* prose finding. This is exactly the "stray KG cross-check appendix with its own prose findings" pattern that is NOT exempt — it is not a Revision Summary and not a one-line `> **Label:**` metadata trailer. It should be removed or folded into Section 1. |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is bullet lists with descriptive prose only; no attribute tables or type declarations. |

---

## KG Cross-Check (reconciliation finding)

The independent KG cross-check re-found **the same six services at the same confidence levels** the spec's Scope note and Appendix claim (aim-portal-ui, aim-portal-api, aim-customers-onboarding-initialization at high; aim-reference-data-api, aim-workflow-manager, aim-attribution-api at low). **No discrepancy** — no service claimed by the spec was un-found, and no evidence service was omitted. The spec's implementation-reality claims (UI → backend → onboarding-init spine, low-confidence adjacencies not to be built around) are consistent with the evidence. The only flagged concern is the *unresolved ordering* between the tool and `aim-customers-onboarding-initialization` (DEC-001 / CG-002), which the evidence corroborates as a genuine open integration risk rather than a contradiction.

---

## 6. Recommendations

1. **Resolve CG-001 (AIM X approval deadlock).** Decide whether AIM X approval is self-serve; add an explicit FR for the zero-touch approval path or remove the "zero-touch" claim. This is the single most spec-breaking contradiction.
2. **Resolve DEC-001 / CG-002 (integration ordering).** Nothing in the highest-regression-risk service can be safely built until before/during/after is decided; add an FR defining the integration point and its backward-compat assertion.
3. **Resolve CG-003 (audit-write failure semantics).** Specify that a failed audit write rejects/rolls back the triggering milestone mutation — immutability is meaningless otherwise.
4. **Remove the Engineering Reference appendix** (or fold it into Section 1) to satisfy the Sections 0–6 constraint.
5. **Pin all threshold boundaries:** inclusive/exclusive for the 10–90% UA/UE band, ≥$250K spend, and the 12-month history overlap; add the NFR-002/FR-006/NFR-010 measurable rewrites from Section 2.
6. **Define destructive-reset confirmations** for FR-012 (funnel/KPI/LTV loss) and the FR-007 sync/unsync state.
7. **Specify stale-artifact handling** for FR-021 PDFs after Unlock-to-edit, and define output behaviour against DEC-002 uncollected fields.
8. **Define rounding/residual rules** for the FR-008 conversion-share allocator and FR-009 campaign-split totals.