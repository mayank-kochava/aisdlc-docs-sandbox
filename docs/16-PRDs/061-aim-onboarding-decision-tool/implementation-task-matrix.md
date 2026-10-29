---
id: implementation-task-matrix
title: Implementation Task Matrix
---

## Implementation Task Matrix — wi_prd061_47cf2c

**Matrix ID:** `wi_prd061_47cf2c-r4` · **Generated:** 2026-10-29T16:03:37.563709+00:00 · **Source:** `tasks` · **Verdict:** **READY with 9 warnings**

### Summary

| Metric | Value |
| :--- | :--- |
| Tasks total | 45 |
| AI-allocated | 44 |
| Manual-allocated | 1 |
| Waves | 12 |
| Conflicts | 0 |
| Requirements uncovered | 4 |
| Low-confidence allocations (<0.6) | 0 |

### Wave 0

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T001 | Register `AimOnboardingSessions = 'aim_onboarding_sessions'` in CollectionNames… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T005 | Add NUnit tests for the data layer (advertiser scoping, latest-wins) in `Kochav… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T101 | Create session and wizard state types (use `campaignGrouping`/`campaignGrouping… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T201 | Check connectivity: confirm the onboarding-sessions gateway rule is deployed in… | Kochava/ko-k8s-apps | manual-developer | manual | 0.90 | planned |

### Wave 1

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T002 | Create `OnboardingSession` POCO and embedded types with BSON attributes | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T102 | Create `onboardingService` on shared `mmmPortalApi` and attach x-user-id for on… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T104 | `recommendTier.ts` with the prototype bug-fixes (budget monthly/annual with ann… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T105 | `buildProvisionPlan.ts` (Data Schema rows from state, per Anupam's CSV format d… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T106 | `validation.ts` (per-step and output validation) plus routing-flag computation | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T119 | SoW PDF export button and print stylesheet (`@media print` + `window.print()`) | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 2

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T003 | Create `IOnboardingSessionDataLayer` interface | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T006 | Create onboarding session DTOs and AutoMapper profile | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T018 | Derive onboarding status (not_started/in_progress/complete) from the session do… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T107 | Cross-module Vitest tests for the logic modules (annual to monthly, 12-24mo qua… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T122 | CSM Approval button and 'Book my onboarding meeting' (sent/retry) component; no… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 3

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T004 | Create `OnboardingSessionDataLayer` (advertiser-scoped Mongo CRUD + change-log… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T108 | `useAimOnboarding` composable: flat state, computed flags, step nav, business-m… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 4

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T007 | Create `IOnboardingSessionService` + `OnboardingSessionService` (get-latest, cr… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T017 | SES templates, `CsmTeamEmail` config and `POST /{id}/book-meeting` (clear 200 /… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T109 | `useOnboardingAutosave` composable: localStorage buffer, debounced full-doc PUT… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T112 | Steps 1-2: Your Team, About Your Product components with progressive disclosure | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T113 | Step 3 Marketing Setup including `campaignGrouping` | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T114 | Step 4 Business Model & Funnel (model change clears LTV/webFunnel; internal eve… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T115 | Steps 5-6: Data Sources, External Factors | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T116 | Steps 7-8: Objectives, Review with Generate (1.6s loader, no backend call) | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T117 | Setup Summary panel (visible steps 2-6) with component tests | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T118 | SoW tab: generated sections, inline notes (editable post-approval), approval bl… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T120 | Timeline tab: 6 milestones, computed dates, client+CSM editing, cascade, audit-… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T121 | Data Schema tab: split from `buildProvisionPlan`, CSV template download (Blob),… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 5

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T008 | Create `OnboardingSessionsController` (GET latest, POST, PUT /{id}) | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T009 | NUnit service tests (create/replace, updatedAt stamp) in `Tests/OnboardingSessi… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T019 | NUnit test for book-meeting success/failure controller response | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T110 | Additional Vitest autosave tests (load-on-mount round trip, localStorage fallba… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T111 | `AimOnboardingTab.vue` wrapper: status routing, v-stepper shell, isASAdmin CSM… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T123 | Cross-component tests for output tabs (SoW render, schema rows, CSM gating) | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 6

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T010 | Resolve caller `is_as_admin` via mos-iam (cached) and expose `IsSuperAdmin` on… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T103 | Register the 3rd tab in `MmmInsightsConfiguration.vue` and add i18n key `mmm_co… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 7

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T011 | Super-admin gate service returning DataUpdateResult.AccessDenied (403) | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T015 | `PUT /{id}/timeline` (client and CSM): milestone status/delay/notes, cascade, s… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| TG1 | Run tests & validate build | Kochava/frontend-mos | test-change-gate | ai | 0.90 | planned |

### Wave 8

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T012 | `POST /{id}/approve` (CSM only) sets ScopeOfWork.CsmApproved with audit entry | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |
| T014 | `POST /{id}/tier-override` (CSM only, asymmetric) with mandatory audit entry (r… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |

### Wave 9

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T013 | `PUT /{id}/sow-approval` (client): approver fields, ApprovedAt, status=complete… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |

### Wave 10

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T016 | NUnit cross-cutting tests: 403 on every privileged op, change-log appended, cas… | Kochava/mmm-portal-api | coding-agent | ai | 0.90 | planned |

### Wave 11

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| TG2 | Run tests & validate build | Kochava/mmm-portal-api | test-change-gate | ai | 0.90 | planned |

### Dependency Graph

```mermaid
graph LR
    T001["T001: Register `AimOnboardingSessions = 'aim_onboarding_sessions'…"]
    T002["T002: Create `OnboardingSession` POCO and embedded types with BSO…"]
    T003["T003: Create `IOnboardingSessionDataLayer` interface"]
    T004["T004: Create `OnboardingSessionDataLayer` (advertiser-scoped Mong…"]
    T005["T005: Add NUnit tests for the data layer (advertiser scoping, lat…"]
    T006["T006: Create onboarding session DTOs and AutoMapper profile"]
    T007["T007: Create `IOnboardingSessionService` + `OnboardingSessionServ…"]
    T008["T008: Create `OnboardingSessionsController` (GET latest, POST, PU…"]
    T009["T009: NUnit service tests (create/replace, updatedAt stamp) in `T…"]
    T010["T010: Resolve caller `is_as_admin` via mos-iam (cached) and expos…"]
    T011["T011: Super-admin gate service returning DataUpdateResult.AccessD…"]
    T012["T012: `POST /{id}/approve` (CSM only) sets ScopeOfWork.CsmApprove…"]
    T013["T013: `PUT /{id}/sow-approval` (client): approver fields, Approve…"]
    T014["T014: `POST /{id}/tier-override` (CSM only, asymmetric) with mand…"]
    T015["T015: `PUT /{id}/timeline` (client and CSM): milestone status/del…"]
    T016["T016: NUnit cross-cutting tests: 403 on every privileged op, chan…"]
    T017["T017: SES templates, `CsmTeamEmail` config and `POST /{id}/book-m…"]
    T018["T018: Derive onboarding status (not_started/in_progress/complete)…"]
    T019["T019: NUnit test for book-meeting success/failure controller resp…"]
    T101["T101: Create session and wizard state types (use `campaignGroupin…"]
    T102["T102: Create `onboardingService` on shared `mmmPortalApi` and att…"]
    T103["T103: Register the 3rd tab in `MmmInsightsConfiguration.vue` and…"]
    T104["T104: `recommendTier.ts` with the prototype bug-fixes (budget mon…"]
    T105["T105: `buildProvisionPlan.ts` (Data Schema rows from state, per A…"]
    T106["T106: `validation.ts` (per-step and output validation) plus routi…"]
    T107["T107: Cross-module Vitest tests for the logic modules (annual to…"]
    T108["T108: `useAimOnboarding` composable: flat state, computed flags,…"]
    T109["T109: `useOnboardingAutosave` composable: localStorage buffer, de…"]
    T110["T110: Additional Vitest autosave tests (load-on-mount round trip,…"]
    T111["T111: `AimOnboardingTab.vue` wrapper: status routing, v-stepper s…"]
    T112["T112: Steps 1-2: Your Team, About Your Product components with pr…"]
    T113["T113: Step 3 Marketing Setup including `campaignGrouping`"]
    T114["T114: Step 4 Business Model & Funnel (model change clears LTV/web…"]
    T115["T115: Steps 5-6: Data Sources, External Factors"]
    T116["T116: Steps 7-8: Objectives, Review with Generate (1.6s loader, n…"]
    T117["T117: Setup Summary panel (visible steps 2-6) with component tests"]
    T118["T118: SoW tab: generated sections, inline notes (editable post-ap…"]
    T119["T119: SoW PDF export button and print stylesheet (`@media print`…"]
    T120["T120: Timeline tab: 6 milestones, computed dates, client+CSM edit…"]
    T121["T121: Data Schema tab: split from `buildProvisionPlan`, CSV templ…"]
    T122["T122: CSM Approval button and 'Book my onboarding meeting' (sent/…"]
    T123["T123: Cross-component tests for output tabs (SoW render, schema r…"]
    T201["T201: Check connectivity: confirm the onboarding-sessions gateway…"]
    TG1["TG1: Run tests & validate build"]
    TG2["TG2: Run tests & validate build"]
    T001 --> T002
    T002 --> T003
    T003 --> T004
    T002 --> T006
    T004 --> T007
    T006 --> T007
    T007 --> T008
    T007 --> T009
    T008 --> T010
    T010 --> T011
    T011 --> T012
    T012 --> T013
    T011 --> T014
    T010 --> T015
    T013 --> T016
    T014 --> T016
    T015 --> T016
    T004 --> T017
    T002 --> T018
    T017 --> T019
    T101 --> T102
    T111 --> T103
    T101 --> T104
    T101 --> T105
    T101 --> T106
    T104 --> T107
    T105 --> T107
    T106 --> T107
    T107 --> T108
    T108 --> T109
    T102 --> T109
    T109 --> T110
    T109 --> T111
    T112 --> T111
    T113 --> T111
    T114 --> T111
    T115 --> T111
    T116 --> T111
    T117 --> T111
    T118 --> T111
    T120 --> T111
    T121 --> T111
    T122 --> T111
    T108 --> T112
    T108 --> T113
    T108 --> T114
    T108 --> T115
    T108 --> T116
    T108 --> T117
    T108 --> T118
    T119 --> T118
    T102 --> T118
    T101 --> T119
    T102 --> T120
    T108 --> T120
    T108 --> T121
    T102 --> T122
    T118 --> T123
    T120 --> T123
    T121 --> T123
    T122 --> T123
    T103 --> TG1
    T110 --> TG1
    T123 --> TG1
    T005 --> TG2
    T009 --> TG2
    T016 --> TG2
    T018 --> TG2
    T019 --> TG2
```

### Open Questions

- `is_as_admin` vs `ROLE_CSM`; mos-iam endpoint for T010 (Satish + Identity)
- `onboarding_status` field shape on `app_config` (T018) (Satish)
- Data Schema template columns per source (T121) (Gary / Data)
- SES recipient config + copy (T017) (Gary / Backend)

### Uncovered Requirements

`US1`, `US2`, `US3`, `US4`

### Warnings

- anchored 1 partial file path to full paths from the PRD plan docs
- grounding pinned to db6f837c; knowledge graph skipped for Kochava/mmm-portal-api
- grounding pinned to 66bc506c; knowledge graph skipped for Kochava/frontend-mos
- T010: task too large (6 files (max 5)); consider splitting
- T017: task too large (8 files (max 5)); consider splitting
- Open item: `is_as_admin` vs `ROLE_CSM`; mos-iam endpoint for T010
- Open item: `onboarding_status` field shape on `app_config` (T018)
- Open item: Data Schema template columns per source (T121)
- Open item: SES recipient config + copy (T017)

---
> Generated by: taskmatrix 0.1.0
