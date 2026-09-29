---
id: eng-context-frontend
title: Engineering Notes - Frontend
---

## Engineering Notes: AIM Onboarding Decision Tool - Frontend

**Engineer:** Mayank Ukey
**Date:** 2026-06-11
**Based on:** product-spec-v2.md + product-spec-v2-reconciliation.md (reconciliation wins on conflict)

---

## Initial Thoughts

- Not a new app/route — it's a **tab inside the existing `MmmInsightsConfiguration` view** in `frontend-mos` (Vue 3 / Vuetify). Route `/advertisertools/mmmconfigurations` already exists; today it has Marketplaces + Validate Onboarding tabs (`v-tabs` + `v-window`). We add a 3rd.
- The hard part isn't the UI chrome — Vuetify gives us `v-stepper` (already used in `MmmOptimization/ScenarioCreateView.vue`) and tabs for free. The hard part is **state correctness**: one flat state object drives the wizard, the tier rec, and all three output documents. Get the state shape + computed flags right and the outputs fall out; get them wrong and every output is subtly wrong.
- spec-v2 says React — ignore that. We're Vue. The React prototype is reference content only.
- Phase 1 now has a real backend (Mongo persistence), so this isn't a pure client-side toy. The frontend ↔ backend persistence contract is a first-class concern, not an afterthought.
- Goal: **complete, working end-to-end.** Not a happy-path demo.

---

## Concerns & Risks

- **Mongo autosave races.** localStorage buffer within a step, then PUT to Mongo on step/tab switch + manual save. Rapid step clicks, slow network, or two tabs open can stomp each other. Need: debounce, a dirty flag, last-write-wins (or a version/`updatedAt` check), and a clear "saving / saved / failed-retry" indicator. Don't lose a client's 8-step answers.
- **Output generation from state.** SoW, Timeline, and Data Schema are all derived from the same flat state. `recommendTier()` and `buildProvisionPlan()` must be ported faithfully **with the 8 prototype bug-fixes** (spec-v2 §9) or the outputs lie. This is where correctness lives.
- **PDF print fidelity.** No PDF lib in the repo → print stylesheet (`@media print` + `window.print()`), SoW only. Risk: page breaks, inline-notes rendering, long free-text, cross-browser. Needs real testing on Chrome/Safari/Firefox/Edge.
- **Stepper + conditional fields.** Heavy progressive disclosure — sub-questions reveal on parent answers, locked sections, and cross-step resets (e.g. changing business model in Step 4 must clear `wantsLtv`/`ltvCohortAvail`/`ltvPartialChoice`/`webFunnel`). Easy to leave stale state that poisons the Data Schema. Vue reactivity makes this cleaner than the prototype but the reset rules must be explicit.
- **Completeness.** Want all of it done properly — wizard, all 3 output tabs, persistence, PDF, CSV, CSM mode — not a subset.

---

## Decisions (frontend, my call)

- **Logic port:** lift the prototype's pure functions (`recommendTier`, `buildProvisionPlan`, validation, tier/flag computation) into **framework-agnostic `.ts` modules** under the tab dir, fix the 8 bugs there, and **unit-test them in isolation**. The Vue composable (`useAimOnboarding.ts`) calls them. Rationale: this logic is the correctness-critical part and the riskiest to port — keeping it pure + tested decouples it from UI churn and makes "works end to end properly" verifiable. Not rewriting inline in the composable.
- **Build order:** **vertical slice first** — tab embed → 1–2 wizard steps → Mongo save/load round-trip → stubbed output tab. Proves the embed + the persistence contract with backend *before* building all 8 steps. Then fill in steps, then output generators, then PDF/CSV polish.

---

## Questions & Answers

### Q1: How does the backend enforce "Super Admin / CSM" server-side?

- **For:** Backend (Satish)
- **Status:** 🔴 Open
- **Answer:** *Pending response*
- **Impact:** Blocks wiring of all CSM-only actions (CSM Approval, Unlock to Edit, timeline status edits, tier-override logging). Frontend has `isASAdmin` (from identity/Kratos) for UI gating, but portal-api has no server-side super-admin today. UI gate alone is insufficient per spec §4.6.

### Q2: Persistence contract for the session?

- **For:** Backend (Satish)
- **Status:** 🔴 Open
- **Answer:** *Pending response*
- **Impact:** Blocks autosave. Need: endpoint shapes, whether PUT takes the full session doc or a patch, expected save cadence/debounce, and conflict/version handling (`updatedAt` check?). Drives how the composable batches saves.

### Q3: `onboarding_status` on `app_config` — exact field + shape?

- **For:** Backend (Satish)
- **Status:** 🔴 Open
- **Answer:** *Pending response*
- **Impact:** Blocks portal-entry gating (wizard vs read-only record). We read this on load to pick the view. Values: not_started / in_progress / complete.

### Q4: Data Schema downloadable templates — which schema types and exact columns?

- **For:** PM (Gary) / Data
- **Status:** 🔴 Open
- **Answer:** *Pending response*
- **Impact:** Direction resolved (Option A, downloadable, we author it). But the actual per-source schema rows + CSV column names need confirming so the templates are correct. Blocks Data Schema tab content, not the shell.

### Q5: CSM guidance-note content per field?

- **For:** PM (Gary)
- **Status:** 🟢 Answered
- **Answer:** When unspecified, follow `mockup-k4a.html` CSM tabConfig (Mayank, 11 June).
- **Impact:** Low — fallback defined.

---

## Rough Scope

New dir `MmmInsightsConfiguration/AimOnboardingTab/`:

- `AimOnboardingTab.vue` — wrapper; reads status → wizard vs read-only output; reads `isASAdmin` for CSM mode.
- `steps/` — 8 step components on a `v-stepper`: 1 Your Team · 2 About Your Product · 3 Marketing Setup (incl. §3.9 `campaignGrouping`) · 4 Business Model & Funnel · 5 Data Sources · 6 External Factors · 7 Objectives · 8 Review.
- `output/` — `SowTab.vue`, `TimelineTab.vue`, `DataSchemaTab.vue` (`v-tabs` + `v-window`).
- `composables/useAimOnboarding.ts` — state + autosave (localStorage buffer + Mongo PUT on step-switch/manual save) + save-status indicator.
- `logic/` — pure TS: `recommendTier.ts`, `buildProvisionPlan.ts`, validation, flag computation (+ unit tests). Bake in the 8 bug fixes.
- `services/onboarding.ts` — CRUD on shared `mmmPortalApi` (mirror `services/incrementality.ts`).
- `interfaces/aimOnboarding.ts` — types; `campaignGrouping`/`campaignGroupingChoice` (not `campaignMerge`).

Touched: `MmmInsightsConfiguration.vue` (register tab), i18n (`mmm_config.tabs.aim_onboarding`), endpoints map. Single-region only. Print stylesheet for SoW PDF. Blob pattern for CSV downloads.

---

## Dependencies on Other Roles

- **Backend (Satish):** session persistence API; server-side super-admin enforcement; `onboarding_status` on `app_config`; "Book my meeting" email endpoint. (Q1–Q3.)
- **Anupam:** Validate Onboarding tab — the Data Schema CSV download feeds into the existing Validate Onboarding upload (spec §4.2 Tab 3 / B8). Confirm hand-off contract.
- **PM (Gary):** Data Schema template content (Q4); CSM note content fallback = mockup (Q5).
- **Design:** AIM X / MOS tokens (`colors_and_type.css`); remove bridge aliases.

---

## Notes

- Reference prototype: `mockup-k4a.html` (K4A-embedded; authoritative for `campaignGrouping`, CSM tabConfig). `mockup-standalone.html` is Gary's standalone (binary-ish, not the build target).
- Stepper pattern to copy: `packages/advertiser/src/views/Analytics/.../` → see `views/MmmOptimization/ScenarioCreateView.vue`.
- API instance + auth interceptor: `views/Analytics/services/mmm.ts`.
- Role flag: `packages/core/src/stores/organization.ts` → `isASAdmin`.
- 8 prototype bugs to fix (spec-v2 §9): tier reads `state.spend`; 24mo history (→12mo); paidSplit 50/50; seasonality always "Included"; tier always AIM X; objectives missing from SoW; channels missing from output; `campaignGrouping` had no UI.

---

*Captured by `/kochava:eng-context frontend` on 2026-06-11*
