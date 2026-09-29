---
id: implementation-task-matrix
title: Implementation Task Matrix
---

## Implementation Task Matrix — wi_c32828e92b97d999618d

**Matrix ID:** `wi_c32828e92b97d999618d-r1` · **Generated:** 2026-09-29T15:55:52.838671+00:00 · **Source:** `tasks` · **Verdict:** **READY**

### Summary

| Metric | Value |
| :--- | :--- |
| Tasks total | 45 |
| AI-allocated | 45 |
| Manual-allocated | 0 |
| Waves | 9 |
| Conflicts | 0 |
| Requirements uncovered | 0 |
| Low-confidence allocations (<0.6) | 0 |

### Wave 0

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T001 | Add `AimOnboardingSessions = 'aim_onboarding_sessions'` to `Kochava.Aim.Portal.DataAccess/Mongo/CollectionNames.cs` | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T002 | Create the `OnboardingSession` aggregate POCO with embedded `Wizard`, `Tier`, `ScopeOfWork`, `TimelineState`, `Mileston… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T010 | Add a mos-iam `GetUserByEmail` helper that resolves the caller's `is_as_admin` (cached, fail closed; `x-user-id` is the… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T101 | Create session and wizard-state types (use `campaignGrouping` and `campaignGroupingChoice`, PascalCase API contract) in… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T103 | Register the third tab (`aimOnboarding` v-tab and v-window-item, i18n key `mmm_config.tabs.aim_onboarding`) in `package… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T105 | Create `buildProvisionPlan` (Data Schema dimension and metric columns from funnel events and sources, flattened cohort-… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T106 | Create per-step and output validation, routing flags (`uaUeRoutingFlag`, `brandRoutingFlag`) and the milestone date and… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T112 | Build Steps 1-2 (Your Team, About Your Product) with progressive disclosure in `packages/advertiser/src/views/Analytics… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T113 | Build Step 3 Marketing Setup including the `campaignGrouping` question in `packages/advertiser/src/views/Analytics/MmmI… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T114 | Build Step 4 Business Model and Funnel (model change clears LTV and `webFunnel` state) in `packages/advertiser/src/view… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T115 | Build Steps 5-6 (Data Sources, External Factors) in `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/A… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T116 | Build Steps 7-8 (Objectives, Review) with Generate (1.6s animation, no backend call) in `packages/advertiser/src/views/… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T119 | Create SoW-only PDF export (print stylesheet plus `window.print()`, excludes empty placeholders) in `packages/advertise… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T124 | Build shared `ToggleCard` (checkbox affordance for multi-select), `ShareAllocator` (sliders summing to 100) and `Budget… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T201 | Add an Oathkeeper access rule for the onboarding-sessions route (GET/POST/PUT, api-key authorizer, `noop` mutators; rol… | Kochava/ko-k8s-apps | sre-agent | ai | 0.90 | planned |

### Wave 1

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T003 | Create `IOnboardingSessionDataLayer` (`GetLatestByAdvertiserAsync`, `CreateAsync`, `ReplaceAsync`, `AppendChangeLogAsyn… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T006 | Create `OnboardingSessionDto` and `OnboardingSessionUpsertDto` (mirror the model, no server-owned fields writable) in `… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T102 | Create `onboardingService` (`getLatest`, `create`, `update`, `bookMeeting`, `approve`, `sowApproval`, `updateTimeline`,… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T117 | Build the collapsible Setup Summary panel (steps 2-6, tier chip) and component tests for representative steps in `packa… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T121 | Build the Data Schema tab (Kochava-collects versus you-provide split from `buildProvisionPlan`, downloadable CSV templa… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| TG2 | Run tests & validate build | Kochava/ko-k8s-apps | test-change-gate | ai | 0.90 | planned |

### Wave 2

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T004 | Create `OnboardingSessionDataLayer` extending `MongoDataLayerBase` with an advertiser-scoped filter (SavedViews data-la… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T104 | Create `recommendTier` with the 8 prototype bug-fixes (budgetMonthly/Annual and annual divided by 12, 12-month history… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T111 | Complete the `AimOnboardingTab` wrapper: status routing from the latest session (wizard, Continue, or read-only record)… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 3

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T005 | Add NUnit tests (advertiser scoping, latest-wins, replace rejected after approval; Moq `IMongoCollection`) in `Kochava.… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T007 | Create `IOnboardingSessionService` and `OnboardingSessionService` (get-latest, create, full-doc replace stamping `Updat… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T107 | Add Vitest unit tests for the three logic modules (annual to monthly, 12 to 24 month qualification, business-model-chan… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T108 | Create `useAimOnboarding` (flat state, computed flags, step navigation, per-step validation derivation) in `packages/ad… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 4

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T008 | Create `OnboardingSessionsController` with `GET` (latest), `POST`, `PUT /{id}` under `/mmm/advertisers/{advertiserId}/o… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T009 | Add NUnit service tests (create, replace stamps `UpdatedAt`, replace after approval conflicts) in `Kochava.Aim.Portal.T… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T011 | Add a service-layer super-admin guard returning `DataUpdateResult.AccessDenied` (maps to 403 through `ForbidWithReason`… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T013 | Create `OnboardingSowApprovalService` and controller `PUT /{id}/sow-approval` (requires `CsmApproved`; sets approver na… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T015 | Create `OnboardingTimelineService` and controller `PUT /{id}/timeline` (milestone status, new-date delay stored as `Del… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T017 | Add `OnboardingMeetingRequested.html` and `.txt` SES templates (subject, body and Reply-To per design-consolidated 8c),… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |

### Wave 5

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T012 | Create `OnboardingCsmService` and `OnboardingCsmController` with `POST /{id}/approve` (sets `CsmApproved`) and `POST /{… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T019 | Add NUnit test for book-meeting success and SES-failure responses in `Kochava.Aim.Portal.Tests/OnboardingMeetingService… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T109 | Add hybrid autosave to the composable (localStorage buffer, debounced full-doc PUT on step or tab switch and manual sav… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T118 | Build the SoW tab (generated sections in mono document style, tier band, inline notes editable after approval, approval… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T120 | Build the Timeline tab as a bare `v-data-table` (6 milestones, computed dates anchored on `approvedAt`, status select,… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 6

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T016 | Add NUnit tests: 403 for non-admin on approve and tier-override, change-log appended with session email, sow-approval t… | Kochava/mmm-portal-api | coding-agent | ai | 0.70 | planned |
| T110 | Add Vitest tests for autosave (debounce, failure retry, load on mount, 409) with a mocked service in `packages/advertis… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| T122 | Build the output shell with validation summary and deep links, output `v-tabs`, CSM Approve button, and the 'Book my on… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |

### Wave 7

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| T123 | Add component tests for output tabs (SoW render, schema rows, CSM gating, book-meeting retry) in `packages/advertiser/s… | Kochava/frontend-mos | coding-agent | ai | 0.80 | planned |
| TG3 | Run tests & validate build | Kochava/mmm-portal-api | test-change-gate | ai | 0.90 | planned |

### Wave 8

| Task | Title | Repo | Owner | Allocation | Confidence | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| TG1 | Run tests & validate build | Kochava/frontend-mos | test-change-gate | ai | 0.90 | planned |

### Dependency Graph

```mermaid
graph LR
    T001["T001: Add `AimOnboardingSessions = 'aim_onboarding_sessions'` to `Kochava.Aim.Portal.DataAccess/Mongo/CollectionNames.cs`"]
    T010["T010: Add a mos-iam `GetUserByEmail` helper that resolves the caller's `is_as_admin` (cached, fail closed; `x-user-id` is the…"]
    T002["T002: Create the `OnboardingSession` aggregate POCO with embedded `Wizard`, `Tier`, `ScopeOfWork`, `TimelineState`, `Mileston…"]
    T003["T003: Create `IOnboardingSessionDataLayer` (`GetLatestByAdvertiserAsync`, `CreateAsync`, `ReplaceAsync`, `AppendChangeLogAsyn…"]
    T004["T004: Create `OnboardingSessionDataLayer` extending `MongoDataLayerBase` with an advertiser-scoped filter (SavedViews data-la…"]
    T005["T005: Add NUnit tests (advertiser scoping, latest-wins, replace rejected after approval; Moq `IMongoCollection`) in `Kochava.…"]
    T006["T006: Create `OnboardingSessionDto` and `OnboardingSessionUpsertDto` (mirror the model, no server-owned fields writable) in `…"]
    T007["T007: Create `IOnboardingSessionService` and `OnboardingSessionService` (get-latest, create, full-doc replace stamping `Updat…"]
    T008["T008: Create `OnboardingSessionsController` with `GET` (latest), `POST`, `PUT /{id}` under `/mmm/advertisers/{advertiserId}/o…"]
    T009["T009: Add NUnit service tests (create, replace stamps `UpdatedAt`, replace after approval conflicts) in `Kochava.Aim.Portal.T…"]
    T011["T011: Add a service-layer super-admin guard returning `DataUpdateResult.AccessDenied` (maps to 403 through `ForbidWithReason`…"]
    T012["T012: Create `OnboardingCsmService` and `OnboardingCsmController` with `POST /{id}/approve` (sets `CsmApproved`) and `POST /{…"]
    T013["T013: Create `OnboardingSowApprovalService` and controller `PUT /{id}/sow-approval` (requires `CsmApproved`; sets approver na…"]
    T015["T015: Create `OnboardingTimelineService` and controller `PUT /{id}/timeline` (milestone status, new-date delay stored as `Del…"]
    T017["T017: Add `OnboardingMeetingRequested.html` and `.txt` SES templates (subject, body and Reply-To per design-consolidated 8c),…"]
    T016["T016: Add NUnit tests: 403 for non-admin on approve and tier-override, change-log appended with session email, sow-approval t…"]
    T019["T019: Add NUnit test for book-meeting success and SES-failure responses in `Kochava.Aim.Portal.Tests/OnboardingMeetingService…"]
    T101["T101: Create session and wizard-state types (use `campaignGrouping` and `campaignGroupingChoice`, PascalCase API contract) in…"]
    T102["T102: Create `onboardingService` (`getLatest`, `create`, `update`, `bookMeeting`, `approve`, `sowApproval`, `updateTimeline`,…"]
    T103["T103: Register the third tab (`aimOnboarding` v-tab and v-window-item, i18n key `mmm_config.tabs.aim_onboarding`) in `package…"]
    T104["T104: Create `recommendTier` with the 8 prototype bug-fixes (budgetMonthly/Annual and annual divided by 12, 12-month history…"]
    T105["T105: Create `buildProvisionPlan` (Data Schema dimension and metric columns from funnel events and sources, flattened cohort-…"]
    T106["T106: Create per-step and output validation, routing flags (`uaUeRoutingFlag`, `brandRoutingFlag`) and the milestone date and…"]
    T107["T107: Add Vitest unit tests for the three logic modules (annual to monthly, 12 to 24 month qualification, business-model-chan…"]
    T108["T108: Create `useAimOnboarding` (flat state, computed flags, step navigation, per-step validation derivation) in `packages/ad…"]
    T124["T124: Build shared `ToggleCard` (checkbox affordance for multi-select), `ShareAllocator` (sliders summing to 100) and `Budget…"]
    T109["T109: Add hybrid autosave to the composable (localStorage buffer, debounced full-doc PUT on step or tab switch and manual sav…"]
    T110["T110: Add Vitest tests for autosave (debounce, failure retry, load on mount, 409) with a mocked service in `packages/advertis…"]
    T112["T112: Build Steps 1-2 (Your Team, About Your Product) with progressive disclosure in `packages/advertiser/src/views/Analytics…"]
    T113["T113: Build Step 3 Marketing Setup including the `campaignGrouping` question in `packages/advertiser/src/views/Analytics/MmmI…"]
    T114["T114: Build Step 4 Business Model and Funnel (model change clears LTV and `webFunnel` state) in `packages/advertiser/src/view…"]
    T115["T115: Build Steps 5-6 (Data Sources, External Factors) in `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/A…"]
    T116["T116: Build Steps 7-8 (Objectives, Review) with Generate (1.6s animation, no backend call) in `packages/advertiser/src/views/…"]
    T117["T117: Build the collapsible Setup Summary panel (steps 2-6, tier chip) and component tests for representative steps in `packa…"]
    T111["T111: Complete the `AimOnboardingTab` wrapper: status routing from the latest session (wizard, Continue, or read-only record)…"]
    T119["T119: Create SoW-only PDF export (print stylesheet plus `window.print()`, excludes empty placeholders) in `packages/advertise…"]
    T118["T118: Build the SoW tab (generated sections in mono document style, tier band, inline notes editable after approval, approval…"]
    T120["T120: Build the Timeline tab as a bare `v-data-table` (6 milestones, computed dates anchored on `approvedAt`, status select,…"]
    T121["T121: Build the Data Schema tab (Kochava-collects versus you-provide split from `buildProvisionPlan`, downloadable CSV templa…"]
    T122["T122: Build the output shell with validation summary and deep links, output `v-tabs`, CSM Approve button, and the 'Book my on…"]
    T123["T123: Add component tests for output tabs (SoW render, schema rows, CSM gating, book-meeting retry) in `packages/advertiser/s…"]
    T201["T201: Add an Oathkeeper access rule for the onboarding-sessions route (GET/POST/PUT, api-key authorizer, `noop` mutators; rol…"]
    TG1["TG1: Run tests & validate build"]
    TG2["TG2: Run tests & validate build"]
    TG3["TG3: Run tests & validate build"]
    T002 --> T003
    T003 --> T004
    T004 --> T005
    T002 --> T006
    T004 --> T007
    T007 --> T008
    T007 --> T009
    T010 --> T011
    T007 --> T011
    T011 --> T012
    T007 --> T013
    T007 --> T015
    T007 --> T017
    T012 --> T016
    T013 --> T016
    T015 --> T016
    T017 --> T019
    T101 --> T102
    T102 --> T104
    T104 --> T107
    T105 --> T107
    T106 --> T107
    T104 --> T108
    T105 --> T108
    T106 --> T108
    T008 --> T109
    T201 --> T109
    T108 --> T109
    T109 --> T110
    T112 --> T117
    T113 --> T117
    T114 --> T111
    T115 --> T111
    T116 --> T111
    T117 --> T111
    T119 --> T118
    T013 --> T118
    T015 --> T120
    T201 --> T120
    T105 --> T121
    T012 --> T122
    T017 --> T122
    T118 --> T122
    T120 --> T122
    T121 --> T122
    T122 --> T123
    T103 --> TG1
    T107 --> TG1
    T124 --> TG1
    T110 --> TG1
    T111 --> TG1
    T123 --> TG1
    T201 --> TG2
    T001 --> TG3
    T005 --> TG3
    T006 --> TG3
    T008 --> TG3
    T009 --> TG3
    T016 --> TG3
    T019 --> TG3
```

### Open Questions

None.

---
> Generated by: taskmatrix 0.1.0
