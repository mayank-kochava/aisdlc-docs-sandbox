# Spec Critique

---
id: product-spec-critique
title: Product Specification Critique
---

# Spec Critique: AIM Onboarding Decision Tool

**Spec Version:** product-spec.md
**Reviewed By:** AI QA Architect
**Date:** 2026-11-27

## Summary

- **Critical Issues:** 2
- **High Issues:** 7
- **Medium Issues:** 8
- **Blocking:** Yes

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | **Appendix contradicts the no-implementation-target stance AND the base-rule format.** Section 1's "Affected-services note" explicitly says the spec "does not pick an implementation target." But the document then appends a full `## Appendix: Engineering Reference` with a multi-paragraph narrative "Affected Services" analysis (role guesses, confidence levels, prose reconciliation discussion). This is exactly the kind of multi-paragraph KG-analysis appendix that is a format violation (see Section 5), and it also duplicates the Section 1 content. | Critical | Appendix | Delete the Appendix entirely; the Section 1 KG cross-check note already carries the discrepancy with a `[DECISION REQUIRED]` tag. Engineering-reference KG grounding belongs in eng-plan, not the spec. |
| CG-002 | **PDF/CSV/output-generation failure path is only partially specified.** NFR-003 requires an "explicit error state on failure rather than hanging" for PDF/CSV, but no FR defines the actual user-facing behavior (retry? partial download? error banner copy? what the user does next). FR-101 says "a real export, not a stub" but says nothing about what renders if generation throws. A client clicking "Download Scope of Work (PDF)" on a timeout hits a dead end. | Critical | 3 (FR-101, FR-100), 5 (NFR-003) | Add FRs defining the failure UX for PDF export and CSV generation: Given export fails, Then an inline error with a retry action is shown and no partial/corrupt file is delivered. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| NFR-001 | "respond immediately to user input, with no perceptible delay" | "Immediately" / "no perceptible delay" are not testable; the p95 ≤ 100ms target is good but the prose preceding it is vague. | Given a user changes any Step 3 campaign-share value, When the routing flags and Setup Summary recompute, Then the UI reflects the new state within p95 ≤ 100ms measured over 100 consecutive changes. (Delete the "immediately/no perceptible delay" prose; keep only the numeric target.) |
| NFR-003 | "within a bounded, user-tolerable window" | "User-tolerable" is subjective; only the parenthetical p95 ≤ 3s is testable. | Given the user clicks Download Scope of Work (PDF), When generation runs, Then the file is delivered within p95 ≤ 3s, else an explicit error state is shown. (Remove "user-tolerable window" language.) |
| FR-095 | "spanning approximately 7 weeks minimum" / "approximately 9+ weeks" | "Approximately" is untestable — a test cannot assert an exact span. | Given an AIM X recommendation, When the Timeline renders, Then exactly 7 milestones are shown AND the span from first milestone Monday to last is ≥ 7 weeks (define exact lower bound). |
| FR-012 | "populate the Scope of Work header and signoff block verbatim" | "Verbatim" is good, but "header and signoff block" has no defined field mapping to assert against. | Given Step 1 is complete, When SoW is generated, Then SoW.header.companyName == Step1.companyName AND SoW.signoff.leadNames == the exact list of Step 1 lead names in entry order. |
| FR-091 | "history is at least 12–24 months" | Ambiguous boundary: does the "12–24 months" bucket satisfy "at least 12–24 months"? Is the threshold ≥12 or ≥ the top of the bucket? | Define the exact numeric boundary: Given monthly spend ≥ $250K AND history bucket ∈ {12–24, 24–36, 36+}, Then upgrade to AIM Pro. State explicitly whether the 12–24 bucket qualifies. |

> Note: the great majority of FRs are already in clean Given/When/Then Gherkin — testability issues are concentrated in NFR prose and "approximately"/boundary wording, not FR format.

---

## 3. Risk Mitigation Audit

| Risk (from spec Section 1 / NFRs) | Mitigation in Spec? | Verdict | Notes |
|-----------------------------------|---------------------|---------|-------|
| Scope-creep elimination (core) | Yes — FR-093/094 approval + lock + unlock cycle | PASS | Well-mitigated; lock-on-approval plus explicit re-approval cycle is deterministic. |
| Mixed-network / Brand ad-stock caveat | Yes — FR-092 flags unproven combination when Brand present | PASS | Mitigation is a concrete FR, not just a listed risk. |
| AIM X zero-touch mandate (hard constraint) | Partial — NFR-012 explicitly a release gate for a LATER phase | PASS (scoped) | Correctly scoped out of this phase; acceptable as a stated gate. |
| Commercial-confidential exposure | Yes — NFR-004/005/007/013 | PASS | Covered across logging, auth, session isolation. |
| Validation / incomplete-submission | Yes — FR-002 hard block + FR-003 output validation | PASS | But see CG below re: "critical required field" — the set of critical fields is never enumerated (see Edge Cases). |
| Notification misfire / premature handoff | Yes — NFR-015 / FR-081 / FR-111 | PASS | Clean separation of Generate vs Book. |
| Change Log audit integrity / retention | Partial — NFR-006 append-only + 24mo retention but tagged `[DECISION REQUIRED]` | FAIL | A risk-bearing audit requirement is left as an unresolved decision. Retention and append-only enforcement mechanism must be settled before engineering, since it governs an authoritative audit trail. |
| Multi-region / different approval stages concurrency | No dedicated mitigation | FAIL | FR-112 allows switching between regions at different approval stages mid-session, but no requirement addresses state-bleed or accidental approval against the wrong region (see Edge Cases). |

---

## 4. Missed Edge Cases

1. **"Critical required field" set is never enumerated.** FR-002 hard-blocks on an "empty critical required field," but no FR or Information Requirement lists *which* fields are critical. This is untestable and under-specified — engineering cannot build the hard block without the list.
2. **Conversion-share / budget-share rounding.** FR-021 and FR-034 require shares to total exactly 100% with proportional redistribution across 3 items. Proportional redistribution of three integers rarely sums to exactly 100 (rounding drift). No rule for where the remainder lands.
3. **Approving the wrong region.** FR-112 permits switching regions mid-session while regions sit at different approval stages. No safeguard prevents a CSM from capturing an approval (name/title/timestamp) against the region they didn't intend while context-switching.
4. **Unlock-to-edit after ETL has begun.** FR-094 lets an approved SoW be unlocked and re-approved. Per the flow, approval unlocks the Data Schema and ETL begins. Nothing defines what happens to in-flight ETL / already-sent client data when scope is unlocked and changed.
5. **Business model change after funnel customization.** FR-040 resets the funnel on business-model change, silently discarding user-renamed events and the chosen KPI. No confirm/warning before destructive reset.
6. **Primary KPI event deleted/toggled off.** FR-042 requires exactly one KPI, but if the user toggles off the event currently set as Primary KPI, behavior is undefined.
7. **Platform deselected after downstream answers given.** Deselecting Web (or iOS) after completing web funnel / web attribution / SKAN-dependent answers — are those orphaned answers purged, hidden, or retained? Undefined.
8. **"Same as project lead" sync then project lead edited/removed.** FR-010 syncs data leads from project leads in real time; behavior when a synced project-lead row is later deleted is undefined (orphaned data lead? auto-remove?).
9. **Demo mode in production / role interaction.** FR-113 pre-fills and jumps to output. No statement on whether demo mode is reachable in production, whether it bypasses NFR-007 client authentication, or whether demo data can contaminate real sessions.
10. **Concurrent CSM edits to the same Timeline.** NFR-007 mentions no cross-session conflicts, but two CSMs editing the same milestone's status/notes simultaneously (last-write-wins? lock?) is unspecified — the Change Log is append-only but the underlying state resolution isn't defined.
11. **Timeline cascade-delay on an already-complete downstream milestone.** FR-096 shifts all subsequent milestones, but if a later milestone is already "Complete," shifting its date is contradictory. Undefined.
12. **Budget below $50K floor on the slider.** FR-030 fixes the slider minimum at $50K, so the floor is enforced by range — but the eligibility-rejection path (what the user sees if they're genuinely sub-floor, since the slider can't even express it) is never defined. A sub-floor client has no path.
13. **Resume with a since-changed master schema.** FR-004 resumes saved answers; if the Data Schema / funnel defaults changed between sessions, reconciliation of stale saved state vs new defaults is unspecified.
14. **Tier auto-recommendation flips Pro→X after a manual Pro selection.** FR-091 allows X→Pro only when auto-rec is Pro. If a user manually upgraded to Pro and then changes an answer so auto-rec drops to X, is their manual Pro preserved or reverted? Undefined.

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | PASS | Every FR (FR-001 through FR-113) is in Gherkin fenced blocks. Explanatory `>` blockquotes accompany some FRs but the requirements themselves are Gherkin. |
| Document contains only Sections 0–6 | **FAIL** | The document ends with a full `## Appendix: Engineering Reference` containing a multi-paragraph narrative "Affected Services" analysis (role guesses, confidence levels, reconciliation prose). This is NOT a sanctioned trailer — it is not a Revision Summary and not a one-line `> **Label:**` metadata blockquote; it is precisely the multi-paragraph KG-analysis appendix the base rule is written to catch. It also duplicates the Section 1 KG cross-check note. (No Revision Summary or blockquote-metadata trailer is present, so the sanctioned-exception clauses do not apply here.) |
| Section 4 is bullet-list only (no tables/types) | PASS | Section 4 is entirely bold-label bullet lists with provenance tags (User-Entered / System-Derived) and no attribute tables or type declarations. The explicit "Types… deliberately omitted" note confirms compliance. |

---

## 6. Recommendations

1. **Remove the `## Appendix: Engineering Reference` section (CG-001 / Format FAIL).** It violates the Sections 0–6 constraint, duplicates Section 1, and asserts an implementation framing the spec elsewhere declines to pick. Move KG grounding to eng-plan.
2. **Specify the PDF/CSV/output failure UX (CG-002).** Add explicit FRs for export/generation failure (error banner + retry, no partial file), satisfying NFR-003 with testable behavior.
3. **Resolve the two `[DECISION REQUIRED]` tags before engineering** — specifically NFR-006's Change Log append-only enforcement and 24-month retention (audit integrity), and the Section 1 delivery-surface discrepancy.
4. **Enumerate the "critical required field" set** referenced by FR-002, so the hard-block is buildable and testable.
5. **Define region-context safety** for FR-112 (prevent approving/editing the wrong region when switching mid-session across approval stages) and define unlock-to-edit behavior after ETL has begun (FR-094 / Edge Case 4).
6. **Add rounding/redistribution rules** for the 100%-total allocators (FR-021, FR-034) and destructive-reset confirmations for business-model change (FR-040) and KPI-event removal (FR-042).
7. **Tighten vague NFR prose** (NFR-001 "no perceptible delay", NFR-003 "user-tolerable") and "approximately" spans in FR-095 to the numeric targets only.
8. **Reconcile against KG evidence:** the independent KG cross-check re-found the same seven services at the same confidence levels, and PRD-061 is confirmed as precedent — so there is no *new* discrepancy between the spec's claimed services and the fresh lookup. The genuine open issue is the framing mismatch the spec already flags (KG candidate services vs. the precedent "embed in K4A console" model); keep it as a `[DECISION REQUIRED]` in Section 1 and delete the redundant Appendix rather than letting two copies drift.