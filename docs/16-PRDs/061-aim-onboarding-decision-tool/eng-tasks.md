---
id: eng-tasks
title: Engineering Tasks
---

## Tasks: AIM Onboarding Decision Tool (Phase 1)

**Source**: product-spec-v2.md + product-spec-v2-reconciliation.md, eng-context-frontend.md, eng-context-backend.md, eng-research.md (no eng-plan.md — superpowers owns planning; this is the task inventory / context input)

**Date**: 2026-06-11

**Status**: Draft — context for superpowers plan breakdown

---

## Task Format

```text
- [ ] [ID] [P?] [Story?] Description with file path
```

`[P]` = parallelizable (different files, no dependency). `[USn]` = user story.

User stories: **US1** Client self-serve wizard → outputs · **US2** Mongo persistence (autosave) · **US3** CSM review/approve + Timeline edits · **US4** Data Schema + book-my-meeting.

Task ID ranges: backend `mmm-portal-api` T001–T099 · frontend `frontend-mos` T101–T199 · infra `ko-k8s-apps` T201–T299.

---

## Repository: Kochava/mmm-portal-api

**Technology**: C# .NET 7, MongoDB.Driver, NUnit + Moq. Mirror the SavedView stack.

### Phase 1: Setup

**Blocking**: None

- [ ] T001 Add `AimOnboardingSessions = "aim_onboarding_sessions"` to `Kochava.Aim.Portal.DataAccess/Mongo/CollectionNames.cs`.

**Checkpoint**: Collection name registered.

### Phase 2: Foundational (BLOCKING)

**Purpose**: Model + data layer that all endpoints depend on. Mirror `SavedView`.

- [ ] T002 Create `OnboardingSession` POCO (+ embedded `Wizard`, `Tier`, `ScopeOfWork`, `TimelineState`, `Milestone`, `ChangeLogEntry`) with BSON attrs in `DataAccess/Mongo/Models/OnboardingSession.cs` — shape per eng-research / reconciliation.
- [ ] T003 [P] Create `IOnboardingSessionDataLayer` in `DataAccess/Mongo/DataLayers/Interfaces/`.
- [ ] T004 Create `OnboardingSessionDataLayer` in `DataAccess/Mongo/DataLayers/` — extend `MongoDataLayerBase`, advertiser-scoped filter (`GetUserScopedFilter` pattern), `GetLatestByAdvertiserAsync` / `CreateAsync` / `ReplaceAsync` / `AppendChangeLogAsync`. Auto-registers via Autofac (`IocModule.cs`).
- [ ] T005 [P] Add NUnit tests for the data layer (advertiser scoping, latest-wins) in `Kochava.Aim.Portal.Tests/OnboardingSessionDataLayerTests.cs` (Moq `IMongoCollection`).

**Checkpoint**: Persist + read a session doc, advertiser-scoped.

### Phase 3: US2 — Persistence API (Priority: P1)

**Goal**: Frontend can create/load/autosave a session. **Independent test**: POST then PUT then GET round-trips the doc.

- [ ] T006 [US2] Create DTOs (`OnboardingSessionDto`, `OnboardingSessionUpsertDto`) in `Kochava.Aim.Portal.Models`.
- [ ] T007 [US2] Create `IOnboardingSessionService` + `OnboardingSessionService` in `Services/` — get-latest, create, full-doc replace (bump `updatedAt`, set `updatedByUserId`). No change-log on autosave.
- [ ] T008 [US2] Create `OnboardingSessionsController` in `Api/Areas/App/Controllers/` — `GET` (latest), `POST`, `PUT /{id}` (full-doc replace) under `/mmm/advertisers/{advertiserId}/onboarding-sessions`. Mirror `SavedViewsController`.
- [ ] T009 [P] [US2] NUnit service tests (create/replace, updatedAt stamp) in `Tests/OnboardingSessionServiceTests.cs`.

**Checkpoint**: CRUD round-trip green.

### Phase 4: US3 — CSM enforcement + privileged actions (Priority: P1)

**Goal**: Only CSM (is_as_admin) can approve/unlock/edit timeline; audit logged. **Independent test**: non-admin → 403; admin action → 200 + change-log entry.

- [ ] T010 [US3] Add a mos-iam client/helper to resolve caller `is_as_admin` (cache) and surface admin flag on `ICurrentUserContext` (`Common/Identity`); populate in `AdvertiserContextActionFilter`. (Confirm `is_as_admin` vs `ROLE_CSM` — eng-research Q2.)
- [ ] T011 [US3] Service-layer super-admin gate → `DataUpdateResult.AccessDenied` (→ 403 via `HandleDataUpdateResult`/`ForbidWithReason`) on privileged ops.
- [ ] T012 [US3] `POST /{id}/approve` (CSM) → set `csmApproved`. Endpoint + service + audit append.
- [ ] T013 [US3] `PUT /{id}/sow-approval` (client) → set `approverName`/`approverJobTitle`/`approvedAt`, `status=complete`, anchor timeline; audit append.
- [ ] T014 [US3] `POST /{id}/unlock` (CSM) → reset `approvedAt`; audit append.
- [ ] T015 [US3] `PUT /{id}/timeline` (CSM) → milestone status/delay/notes + cascade + tier-override; append change-log (timestamp, session email, field, old/new).
- [ ] T016 [P] [US3] NUnit tests: 403 for non-admin on each privileged op; change-log appended; cascade math.

**Checkpoint**: Privileged ops gated + audited.

### Phase 5: US4 — Email + status flag (Priority: P2)

- [ ] T017 [US4] Add `OnboardingMeetingRequested.html/.txt` SES templates (embedded resources) + `CsmTeamEmail` to `appsettings`; `POST /{id}/book-meeting` via `IEmailService`; return clear 200/error.
- [ ] T018 [US4] Write/read `onboarding_status` on the `app_config` collection (not_started/in_progress/complete) on create/approve. (Confirm field shape — eng-research Q3.)
- [ ] T019 [P] [US4] NUnit test for book-meeting success/failure response.

**Checkpoint**: Email fires; status readable for portal entry.

---

## Repository: Kochava/frontend-mos

**Technology**: Vue 3.5 / Vuetify 3.12, Pinia, Vitest + @vue/test-utils. Tab embed in MmmInsightsConfiguration.

### Phase 1: Setup / Foundational (BLOCKING)

- [ ] T101 Create `AimOnboardingTab/interfaces/aimOnboarding.ts` — session + wizard state types (use `campaignGrouping`/`campaignGroupingChoice`).
- [ ] T102 Create `AimOnboardingTab/services/onboarding.ts` — `onboardingService.{getLatest,create,update,bookMeeting,approve,sowApproval,unlock,updateTimeline}` on shared `mmmPortalApi` (mirror `services/incrementality.ts`).
- [ ] T103 Register the 3rd tab in `MmmInsightsConfiguration.vue` (`aimOnboarding` v-tab + v-window-item, `:can-create`/`:can-update`); i18n key `mmm_config.tabs.aim_onboarding`.

**Checkpoint**: Empty tab renders in the existing view.

### Phase 2: US1 — Pure logic + composable (BLOCKING for wizard)

**Goal**: Correct tier/provision/validation logic, framework-agnostic + tested.

- [ ] T104 [US1] `AimOnboardingTab/logic/recommendTier.ts` with the 8 prototype bug-fixes (budgetMonthly/Annual + annual÷12; 12mo history threshold; etc.).
- [ ] T105 [P] [US1] `logic/buildProvisionPlan.ts` (Data Schema rows from state).
- [ ] T106 [P] [US1] `logic/validation.ts` (per-step + output validation) + flag computation (`uaUeRoutingFlag`, `brandRoutingFlag`).
- [ ] T107 [US1] Vitest unit tests for T104–T106 (incl. annual→monthly, 12–24mo qualifies, business-model-change reset).
- [ ] T108 [US1] `composables/useAimOnboarding.ts` — flat state, computed flags, step nav.
- [ ] T109 [US2] Add Mongo hybrid autosave to the composable — localStorage buffer; debounced full-doc PUT on step/tab switch + manual save; save-status (saving/saved/failed); load latest on mount; guard overlapping PUTs.
- [ ] T110 [P] [US2] Vitest tests for autosave (debounce, failure-retry, load-on-mount) with mocked service.

**Checkpoint**: Logic green; state persists round-trip.

### Phase 3: US1 — Wizard UI

- [ ] T111 [US1] `AimOnboardingTab/AimOnboardingTab.vue` wrapper — status routing (wizard vs read-only record), `v-stepper` shell (pattern: `MmmOptimization/ScenarioCreateView.vue`), `isASAdmin` for CSM mode (`useOrganizationStore`).
- [ ] T112 [US1] Steps 1–2: Your Team, About Your Product (`steps/Step1Team.vue`, `steps/Step2Product.vue`) + progressive disclosure.
- [ ] T113 [US1] Step 3 Marketing Setup incl. §3.9 `campaignGrouping` (`steps/Step3Marketing.vue`).
- [ ] T114 [US1] Step 4 Business Model & Funnel (`steps/Step4Funnel.vue`) — model change clears LTV/webFunnel state.
- [ ] T115 [US1] Steps 5–6: Data Sources, External Factors (`steps/Step5Data.vue`, `steps/Step6Factors.vue`).
- [ ] T116 [US1] Steps 7–8: Objectives, Review (`steps/Step7Objectives.vue`, `steps/Step8Review.vue`) + Generate (1.6s animation, no backend call).
- [ ] T117 [P] [US1] Setup Summary panel (steps 2–6) + component tests for representative steps.

**Checkpoint**: 8-step wizard completes; state correct.

### Phase 4: US1/US4 — Output tabs

- [ ] T118 [US1] SoW tab `output/SowTab.vue` — generated sections, inline notes (editable post-approval), approval block (name+job title+timestamp), objectives section.
- [ ] T119 [US1] PDF export — print stylesheet (`@media print` + `window.print()`), SoW only, excludes empty placeholders.
- [ ] T120 [US3] Timeline tab `output/TimelineTab.vue` — 6 milestones + computed dates (anchor from approvedAt), responsible/duration; CSM status/delay/notes editing, cascade, audit-log view, tier-override (gated on isASAdmin + 403-aware).
- [ ] T121 [US4] Data Schema tab `output/DataSchemaTab.vue` — split (Kochava-collects vs you-provide) from `buildProvisionPlan`; downloadable CSV templates (Blob pattern); feeds Validate Onboarding tab.
- [ ] T122 [US3/US4] CSM-mode UI: CSM Approval button, Unlock to Edit, "Book my onboarding meeting" (sent/retry per spec mermaid).
- [ ] T123 [P] Component tests for output tabs (SoW render, schema rows, CSM gating).

**Checkpoint**: All 3 output tabs functional; CSM actions wired.

---

## Repository: Kochava/ko-k8s-apps

**Technology**: Oathkeeper access rules (YAML).

### Phase 1: Foundational

- [ ] T201 [US2] Add Oathkeeper route rule for `…/onboarding-sessions<.*>` (GET/POST/PUT/DELETE) to `oathkeeper/proxy/{prod,qa}/.../portal/portal-mmm-kochava-com-api.yaml`, mirroring `mmm-portal-advertiser-saved-views` (api-key authorizer, `noop` mutators). NOTE: role enforcement is in portal-api (T010), not here.

**Checkpoint**: Onboarding routes reachable through the gateway.

---

## Dependencies & Execution Order

### Phase dependencies

```text
Backend Foundational (T002-T005) ──► Persistence API (T006-T009) ──► CSM/actions (T010-T016) ──► Email/status (T017-T019)
Oathkeeper route (T201) ──► frontend can reach API
FE foundational (T101-T103) ──► logic+composable (T104-T110) ──► wizard (T111-T117) ──► outputs (T118-T123)
```

### Cross-repository dependencies

| Task | Depends on | Reason |
|------|-----------|--------|
| T109 (FE autosave) | T008 (PUT endpoint), T201 (route) | Frontend needs the persistence API reachable |
| T120/T122 (CSM UI) | T010–T016 (server gate + actions) | UI relies on 403 + privileged endpoints |
| T121 (Data Schema) | — | Client-side from state; backend-independent |

### Vertical slice (recommended first)

T001→T004→T008 + T201 + T101→T103 + minimal T108/T109 → embed → 1 step → Mongo round-trip. Locks the persistence contract early (per eng-context-frontend build-order decision).

---

## Open Items

| Item | Blocks | Resolver |
|------|--------|----------|
| `is_as_admin` vs `ROLE_CSM`; mos-iam endpoint for T010 | CSM gate wiring (mechanism decided) | Satish + Identity |
| `onboarding_status` field shape on `app_config` (T018) | Portal entry gating | Satish |
| Data Schema template columns per source (T121) | Schema tab content (not shell) | Gary / Data |
| SES recipient config + copy (T017) | Email content | Gary / Backend |

---

## Task Count Summary

| Category | Count |
|----------|-------|
| Total | 42 |
| mmm-portal-api | 19 |
| frontend-mos | 22 |
| ko-k8s-apps | 1 |

**MVP (vertical slice + US1/US2):** through T117 + backend T001–T009 + T201.

---

## References

- product-spec-v2.md / product-spec-v2-reconciliation.md
- eng-context-frontend.md / eng-context-backend.md
- eng-research.md
