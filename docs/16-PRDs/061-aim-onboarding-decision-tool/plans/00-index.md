---
id: plans-index
title: "Implementation Plans — Index"
---

## AIM Onboarding — Implementation Plans Index

**Date:** 2026-06-11
**Status:** Plan set for Phase 1. One file per plan; one Asana parent per plan; TDD steps = subtasks.

### How to use

Each plan is a self-contained, TDD, bite-sized implementation plan (writing-plans format: exact paths, real code, no placeholders, frequent commits). Every plan references the **shared contracts** below verbatim so the set stays consistent.

### Shared contracts (authoritative — do not re-invent)

- **`design-consolidated.md`** — §5 Mongo doc shape, §6 API contract, §7 auth, **§8a state/validation/persistence rules**, **§8b component + a11y build notes**. This wins on every conflict.
- **`product-spec-v2.md` §4.1** — field lists for all 8 wizard steps (the mock only draws Steps 2–3).
- **`product-spec-v2-reconciliation.md`** — Vue/Mongo/`campaignGrouping`/demo-dropped reconciliation.
- **`eng-research.md`** — repo patterns + exact file paths (SavedView stack, `mmmPortalApi`, `v-stepper` example, theme tokens).
- **`ui-preview.html`** — the agreed K4A-native visual reference (v5, Gary-flavored, embedded). Frontend plans target visual parity with this.

### Locked decisions (2026-06-11) every plan must honor

- One collection `aim_onboarding_sessions`, one doc/advertiser, embedded aggregate; **PascalCase** fields (per `SavedView`). PDF + Data Schema not stored (generated/computed).
- **`onboarding_status` derived from `session.status`** — not stored on `app_config`/`advertisers`.
- **Approval is terminal** — no unlock / no re-edit of answers (autosave PUT → 409 once `approvedAt` set).
- **Timeline editable by client + CSM**; audit author = **session email**; cascade = new-date → preview → confirm.
- **CSM-only (403-gated):** `approve`, `tier-override` (asymmetric). Enforcement = portal-api → mos-iam `is_as_admin` lookup (Oathkeeper can't carry it).
- Repos: frontend `frontend-mos`, backend `mmm-portal-api`, infra `ko-k8s-apps`.

---

> **Pending inputs RESOLVED 2026-06-11** — see `design-consolidated.md §8c`. CSM signal = `is_as_admin` (frontend `isASAdmin`; backend mos-iam `GetUserByEmail`, `x-user-id`=email). Data Schema columns from `advertiser_schema_mappings` + `validation_rules` (DB). Email recipients=`CsmTeamEmail` (Kade+Jacob) + copy specified. The 🟡 flags in the tables below are superseded by §8c.

### Backend — `mmm-portal-api` (T001–T099) · mirror the SavedView stack

| ID | Title | Scope | Depends on | Pending |
|----|-------|-------|-----------|---------|
| B1 | OnboardingSession model | POCO + embedded `Wizard/Tier/ScopeOfWork/TimelineState/Milestone/ChangeLogEntry` (BSON, PascalCase) + `CollectionNames` entry | — | — |
| B2 | OnboardingSessionDataLayer | advertiser-scoped CRUD, `GetLatestByAdvertiser`, full-doc replace (409 if approved), changeLog append + NUnit | B1 | — |
| B3 | DTOs + OnboardingSessionService | create / replace, terminal-approval guard + NUnit | B2 | — |
| B4 | OnboardingSessionsController | GET latest (drives status) / POST / PUT autosave (409 post-approval) | B3 | — |
| B5 | CSM enforcement | mos-iam `is_as_admin` resolver (cached) → `ICurrentUserContext` via `AdvertiserContextActionFilter`; service gate → 403 | B3 | 🟡 signal field (Satish) |
| B6 | CSM action endpoints | POST `/approve`, POST `/tier-override` (asymmetric) + audit append + 403 gate | B4,B5 | — |
| B7 | SoW approval endpoint | PUT `/sow-approval` → name+title+timestamp, `status=complete`, anchor timeline, terminal lock + audit | B4 | — |
| B8 | Timeline endpoint | PUT `/timeline` — status/delay(new-date)/notes + cascade recompute + changeLog (client+CSM, author=session email) | B4 | — |
| B9 | Book-meeting email | POST `/book-meeting` → SES template + `CsmTeamEmail` config; 200/error | B4 | 🟡 recipients/copy (Gary) |

### Infra — `ko-k8s-apps` (T201)

| ID | Title | Scope | Depends on |
|----|-------|-------|-----------|
| I1 | Oathkeeper route rule | Expose `…/onboarding-sessions<.*>` (prod+qa), mirror `mmm-portal-advertiser-saved-views`. Role enforcement is in portal-api, not the gateway | — |

### Frontend foundation — `frontend-mos` (T101–T199)

| ID | Title | Scope | Depends on |
|----|-------|-------|-----------|
| F0 | Design-token alignment | Verify MOS theme covers the design; adopt AA tokens (`#427900`/`#BA4E00`); Inter; remove input focus-halo | — |
| F1 | Tab embed + routing | `AimOnboardingTab.vue` wrapper; status routing (not_started/in_progress/complete-record from `session.status`); i18n; register 3rd tab in `MmmInsightsConfiguration.vue` | F2 |
| F2 | Service + interfaces | `services/onboarding.ts` + `interfaces/aimOnboarding.ts` (PascalCase contract, `campaignGrouping`) | — |
| F3 | useAimOnboarding composable | flat state, computed flags, step nav, per-step validation derivation (§8a) | F2 |
| F4 | Autosave | localStorage buffer + debounced Mongo full-doc PUT (step-switch/manual); save-status saving/saved/failed/offline; 409-after-approval; load-on-mount | F3,B4 |
| F5 | logic/recommendTier | tier algorithm + the 8 prototype bug-fixes (§9 spec) + Vitest | F2 |
| F6 | logic/buildProvisionPlan | Data Schema provision plan from state + Vitest | F2 |
| F7 | logic/validation + milestones | per-step validation + est-date formula + cascade recompute + Vitest | F2 |

### Component kit — `frontend-mos` (build-fresh; reuse InputTags/InputSlider/NotificationBanner→v-alert/v-tabs/v-data-table/v-stepper where they exist)

| ID | Title | Scope |
|----|-------|-------|
| C1 | ToggleCard | on `v-card`; **checkbox affordance** for multi-select; selected/locked states + tests |
| C2 | ShareAllocator | `v-slider`s summing to 100% + proportional redistribute + tests |
| C3 | BudgetSlider | `v-slider` Monthly/Annual toggle + `fmtM` + qualify banner + tests |
| C4 | StepRail + SummaryPanel | custom vertical rail (icon+label+number, done/error/active, Output step 9, Save progress) + collapsible `Model configuration` summary (`v-expansion-panels`, estimating→tier) + tests |

### Wizard steps — `frontend-mos` (read `product-spec-v2.md §4.1` for fields)

| ID | Title | Notes |
|----|-------|-------|
| W1 | Step 1 — Your Team | repeating lead rows, "same as project lead" sync |
| W2 | Step 2 — About Your Product | platforms, conversion share (ShareAllocator), spend-separability, model structure, market |
| W3 | Step 3 — Marketing | budget (BudgetSlider), media types, attribution, campaign types + split, **§3.9 `campaignGrouping`** |
| W4 | Step 4 — Business & Funnel | model cards, gaming revenue, funnel checklist + KPI, web funnel, LTV; model-change clears LTV/webFunnel |
| W5 | Step 5 — Data Sources | history per platform, MMP + method, AppsFlyer cohort, web attr, ad spend |
| W6 | Step 6 — External Factors (optional) | auto Seasonality/Holidays + 6-factor grid + notes |
| W7 | Step 7 — Objectives (optional) | 3 textareas + empty placeholder |
| W8 | Step 8 — Review | 7 review sections + Edit + Generate (1.6s animation, no backend call) |

### Outputs — `frontend-mos`

| ID | Title | Scope | Pending |
|----|-------|-------|---------|
| O1 | SoW content | mono document (JetBrains Mono), navy section labels, DRAFT stamp, tier band, objectives | — |
| O2 | SoW approval + notes | name+title+timestamp (terminal) + always-editable inline notes | — |
| O3 | SoW PDF | print stylesheet, SoW only, mono doc | — |
| O4 | Timeline tab | **`v-data-table`** (Milestone/Responsible/Est.Start/Duration/Est.End/Status), delivery-plan header, cascade note | — |
| O5 | Timeline editing | status / delay (new-date→preview→confirm) + cascade recompute + changeLog (client+CSM) | — |
| O6 | Data Schema tab | auto-collected vs you-provide ("DATA PROVISION" framing) + CSV templates | 🟡 columns (Gary) |
| O7 | Output shell | validation summary (deep-link to field), tier header, output `v-tabs`, "generated from setup", completion/read-only-record | — |

### Cross-cutting — `frontend-mos`

| ID | Title | Scope | Pending |
|----|-------|-------|---------|
| X1 | CSM mode UI | `isASAdmin` gating, CSM Approve, tier-override toggle (asymmetric, 403-aware), admin bar | — |
| X2 | Book-my-meeting button | button + sent/retry UX (spec mermaid) | 🟡 copy (Gary) |
| X3 | Entry routing UI | wizard vs "Continue" vs read-only "View your onboarding record" (reads `session.status`) | — |

**Total: 40 plans.**

---

### Build order

Vertical slice first: **B1→B4 + I1 + F0→F2 + minimal F4** → embed → 1 step → Mongo round-trip → stub output. Then logic (F5–F7), component kit (C1–C4), wizard steps (W1–W8), outputs (O1–O7), CSM + cross-cutting (B5/B6 + X1–X3), email (B9).

### Asana mapping

Each plan file → one parent task in project `1215287604489661`, section `1215287743348715`. Its TDD steps → subtasks. Group label = workstream (Backend / Infra / Frontend-foundation / Component-kit / Wizard / Outputs / Cross-cutting).
