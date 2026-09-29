---
id: product-spec-v3-critique
title: Product Specification Critique (v3 — Data Mapping Tab)
---

## Spec Critique: AIM Onboarding Decision Tool — Data Mapping Tab (v3)

**Spec Version:** product-spec-v3.md
**Reviewed By:** AI QA Architect
**Date:** 2026-08-06

## Summary

- **Critical Issues:** 3
- **High Issues:** 4
- **Medium Issues:** 4
- **Blocking:** Yes

**Scope note:** This critique focuses primarily on the new Data Mapping tab content (§4.2 Tab 4 and everything it touches), since the already-shipped scope (Steps 1–8, Tabs 1–3) already went through a prior critique/revision cycle (v1→v2, see `product-spec-critique.md`). Format Compliance findings apply to the whole document.

---

## 1. Critical Logic Gaps

| ID | Issue | Severity | Section | Suggested Fix |
|----|-------|----------|---------|---------------|
| CG-001 | The spec never states whether the Data Mapping tables (Conversion Events, Media Metrics & Dimensions) are editable **before** Scope of Work is approved, or whether the entire tab is locked/read-only until then. §4.2 only says the *Approve control* is disabled pre-SoW-approval; §4.6's User Modes table lists "Data Mapping edits + approval (after SoW approved)" for the client row, which reads ambiguously as either "(edits + approval), both gated on SoW" or "edits [ungated] + approval (gated on SoW)." An engineer cannot build the gating logic without this being resolved. | Critical | §4.2, §4.6 | State explicitly: are cells editable pre-SoW-approval (so a client can get a head start) or does the whole tab stay non-interactive until SoW is approved? Update both sections to say the same thing unambiguously. |
| CG-002 | The `AIM Metric` and `AIM Schema Element` dropdowns have no "Other" / custom-value escape hatch. Every other selectable field in this wizard that constrains input to a fixed list (MMP: `Other/none`, Web attribution sources: `Other`, Ad spend sources: `Other` + free text) has one — because a fixed list can't cover every client's reality. If a client's actual mapping doesn't fit any option, they are forced to either pick a **wrong** value (defeating the entire purpose of the tab) or the client is blocked with no path forward. This is a data-integrity risk, not just a UX gap. | Critical | §4.2 Tab 4 | Add an "Other" option (with a free-text field, same pattern as the rest of the wizard) to both dropdowns, or explicitly justify why this tab is the one place in the wizard where a closed list is safe. |
| CG-003 | Approval is terminal (§4.2, §9) and there is explicitly no re-opening mechanism (§4.7: "a future decision if a real need emerges"). Combined with OQ-13 (the `AIM Metric`/`AIM Schema Element` option lists are engineering-authored placeholders pending Data Engineering sign-off): a client who approves Data Mapping **today**, using the placeholder list, has **no defined recovery path** once Data Engineering finalizes the real list — even if their approved mapping turns out to reference an option that no longer exists or was wrong all along. The spec that exists specifically to prevent "field-mapping mismatches discovered after ingestion begins" (§2 Problem Statement) has no answer for the exact failure mode it describes, once compounded with its own known-placeholder dependency. | Critical | §4.2, §11 OQ-13, §4.7 | Either (a) block Data Mapping approval entirely until OQ-13 is resolved, or (b) define a correction path (e.g., a CSM-triggered re-open, scoped narrowly to this one field) for mappings approved against a placeholder list that later changes. Silence on this is not an acceptable answer given the terminal-approval design. |

---

## 2. Testability Review

### Vague Requirements Found

| FR ID | Original Text | Problem | Suggested Rewrite (Gherkin) |
|-------|---------------|---------|----------------------------|
| §4.2 (Approve gate) | "The Approve control is disabled, **with an explanatory note**, until then." | Copy is never specified. Two different "explanatory note" moments exist in this tab (SoW-not-approved, and required-fields-still-blank) and neither has exact text, unlike the SoW tab's "Book my onboarding meeting" flow, which specifies exact confirmation/error copy. Not testable as written. | `Given Scope of Work has not been approved, When the client views the Data Mapping tab, Then the Approve button is disabled and the text "Approve Scope of Work first to unlock Field Mapping approval." is shown` (exact copy TBD with content design, but a placeholder Gherkin scenario should exist). |
| §4.2 (required-field gate) | "Every ... cell must be filled in before the Approve control can be submitted" | Doesn't specify *how* the client is told which cells are still blank — inline per-cell error state? A summary list (like the wizard's own output-screen validation banner, §4 elsewhere)? Not testable without knowing the feedback mechanism. | `Given at least one Source Event or Source Field cell is blank, When the client attempts to approve, Then the Approve control remains disabled and a list of the specific blank rows is shown, consistent with the existing output-screen validation banner pattern` |
| §4.2 (media rows) | "always exactly 8 individually listed rows (never combined)" | This one is fine — testable as written, flagged here only as a positive example of the format the rest of the tab's requirements should follow. | *(no rewrite needed)* |

---

## 3. Risk Mitigation Audit

| Risk (from spec §10 / Problem Statement) | Mitigation in Spec? | Verdict | Notes |
|-------------------------------------------|---------------------|---------|-------|
| Field-mapping mismatches discovered after ingestion begins | Yes — required literal entry + explicit approval | PASS | The base mechanism is sound. |
| Client's in-progress mapping edits lost before approval | Yes — FR requiring persistence across navigation/reload | PASS | |
| Mapping approved out of business-sequence order (before SoW) | Yes — SoW-must-be-approved-first gate | PASS | Undermined in practice by CG-001's ambiguity about what "gate" actually covers. |
| Approved mapping values silently changed after the fact | Yes — terminal lock on approval | PASS (as stated) | But see CG-003 — the terminal lock creates a *new*, unaddressed risk once combined with OQ-13's placeholder list. |
| **(Not in spec's own risk list) Closed-list dropdown forces a wrong selection when no option fits** | No | **FAIL** | See CG-002. This risk isn't even named in §10, let alone mitigated. |
| **(Not in spec's own risk list) Upstream wizard-answer changes after mapping data has been entered** | No | **FAIL** | See CG-004 below. The equivalent risk for LTV state (Step 4 business-model change) *is* explicitly handled elsewhere in this same document — the omission here is inconsistent with the document's own established standard. |

---

## 4. Missed Edge Cases

1. **Upstream wizard-answer changes orphan or misalign Data Mapping rows:** Conversion Events rows are derived from `Funnel.Items`/`WebFunnel.Items` (§4.1 Step 4). If a client returns to Step 4 and changes their business model or funnel selections *after* they've already entered Source Event names for the old row set, what happens to that data? The document already has an explicit precedent for this exact class of problem — Step 4's business-model change "resets the funnel and clears LTV-related answers to prevent stale... rows" — but no equivalent rule is stated for Data Mapping's own entered values.
2. **Multi-platform clients and per-platform event names:** Conversion Events has one row per funnel item, not per platform. A client with both iOS and Android may have genuinely different literal event names per platform for the same logical funnel step (common MMP behavior). The spec's single-row design has no stated way to capture this — is it intentional (one shared name assumed sufficient) or a gap?
3. **CSM edit rights on Data Mapping cells are unstated.** §4.6's User Modes table lists "Data Mapping edits + approval" only for the Client row; the CSM row lists no Data Mapping edit capability, only implicit "all output tabs" access. In the CSM-Assisted flow (Flow 2), can a CSM correct a client's typo in Source Event/Source Field, or is the CSM strictly read-only here even while actively assisting?
4. **No escalation/reminder mechanism for an indefinitely-incomplete Data Mapping tab.** The Scope of Work flow has an explicit progression mechanism ("Book my onboarding meeting"). Data Mapping has none — a client could leave it half-filled forever, silently stalling the ingestion the whole feature exists to unblock, with no defined nudge, deadline, or CSM alert.
5. **Duplicate `Source Field` values across Media rows are not addressed** — is a client allowed to map two different rows (e.g., `campaign` and `country`) to the identical literal source field name? Might be legitimate, might be a client error; the spec doesn't say whether this should be flagged.
6. **No length constraints stated** for `Source Event`, `Source Field`, `Display Label`, or `Rule / Note` free-text fields.
7. **Concurrent-edit race** between two sessions (e.g., client editing in one tab, CSM viewing/attempting a related action in another) is not addressed at the spec level — reasonable to defer to eng-plan, but worth an explicit note rather than silence, given this PRD's own engineering docs elsewhere show real concern for exactly this class of race (session autosave, targeted updates).
8. **What does the client see for the `AIM Metric`/`AIM Schema Element` dropdown while OQ-13 is still unresolved?** If Data Engineering's final list differs from the engineering-authored placeholder before this ships, is there a migration/backfill concern for any sessions that started before the list changed? Related to CG-003 but distinct: this is about pre-launch list changes, not post-approval ones.

*(Per skill instructions, this list is not capped — items 1–8 above represent every meaningful gap found in this pass.)*

---

## 5. Format Compliance

| Check | Result | Details |
|-------|--------|---------|
| All FRs in Gherkin (Given/When/Then) | **FAIL** | The document uses narrative prose/bullet requirements throughout (§4.1 Step descriptions, all of §4.2 Tab 4 including the new Data Mapping content), with only a handful of Gherkin scenarios embedded (§10 NFRs). This is a **deliberate, precedent-matching choice**, not an oversight — `product-spec-v3.md` intentionally mirrors `product-spec-v2.md`'s established structure for this PRD (confirmed by direct comparison), rather than the current `product-spec` skill's strict Gherkin-only format. Flagged per skill instructions; no action needed unless Product specifically wants the document reformatted to the current skill's template. |
| Document contains only Sections 0–6 | **FAIL** | Document has 16 numbered sections (Executive Summary through Revision Summary), matching v2's historical structure. Same deliberate-precedent note as above applies. |
| Section 4 is bullet-list only (no tables/types) | **FAIL** | This document's §4 (Feature Requirements) contains multiple tables (Timeline milestone table, User Modes table, etc.). §7 (Data Requirements, which plays the role the skill calls "Information Requirements") explicitly contains a typed attribute table (State Shape, with a Type column) — direct violation of the current skill's rule if that rule is applied to this section. Same deliberate-precedent note applies. |

---

## 6. Recommendations

1. **Resolve CG-001** — state unambiguously whether Data Mapping cells are editable before Scope of Work approval, or whether the whole tab is non-interactive until then. This blocks engineering from starting.
2. **Resolve CG-002** — add an "Other" escape hatch to both dropdowns, matching the wizard's own established pattern everywhere else a closed list is used, or explicitly justify its absence here.
3. **Resolve CG-003** — decide and document what happens to an approved Data Mapping record if the placeholder `AIM Metric`/`AIM Schema Element` list changes after approval. This is a real data-integrity question, not a documentation nicety.
4. **Resolve edge case #1 (upstream wizard changes)** — add an explicit rule for Data Mapping data when Step 4/5 answers change after mapping values have been entered, mirroring the existing LTV-state-clearing precedent.
5. **Resolve edge case #3 (CSM edit rights)** — state plainly whether CSM can edit Data Mapping cells, not just view them.
6. **Decide on edge case #2 (multi-platform event names)** — confirm whether one shared row per funnel item is intentional, or whether per-platform overrides are needed.
7. **Specify exact copy** for both "explanatory note" moments (§4.2), even as placeholder text pending content design sign-off, so the requirement is testable.
8. **Consider an escalation mechanism** (edge case #4) for indefinitely-incomplete Data Mapping, given it gates the outcome the whole feature exists to protect.
9. **Note the Format Compliance findings are accepted precedent-matching deviations**, not defects — no action required unless Product wants this document reformatted to the current `product-spec` skill's strict template.

---
> Generated by: claude-sonnet-5
