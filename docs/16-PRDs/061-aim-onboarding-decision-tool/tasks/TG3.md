# TG3: Run tests & validate build

## In short

Run the full build and tests for Kochava/mmm-portal-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG3 |
| Repo | Kochava/mmm-portal-api |
| Phase | Phase 14 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/mmm-portal-api with every task above merged. Do not edit source.

This task is part of AIM Onboarding Decision Tool.

## 2. Architecture (3 repositories)
- **frontend-mos** (Vue 3.5 / Vuetify 3.12) — wizard + outputs as a **3rd tab** in `MmmInsightsConfiguration` (`packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/`). No new route (`/advertisertools/mmmconfigurations` exists). `v-stepper` wizard (pattern: `views/MmmOptimization/ScenarioCreateView.vue`); `v-tabs`/`v-window` outputs. Pure-TS logic modules + `useAimOnboarding` composable + `onboarding` service on the shared `mmmPortalApi` axios instance (`views/Analytics/services/mmm.ts`).
- **mmm-portal-api** (C# .NET 7, MongoDB.Driver) — persistence mirroring the **SavedView** stack (model + data layer + service + controller + DTOs); CSM-action endpoints with an embedded audit log; SES email; `onboarding_status` on `app_config`.
- **ko-k8s-apps** — one Oathkeeper access-rule exposing `…/onboarding-sessions<.*>` (mirror `mmm-portal-advertiser-saved-views`). Role enforcement lives in portal-api, not the gateway.


## Implementation Guide

1. Implement: Run tests & validate build
2. Add or update the tests listed under Files to Modify
3. Run `dotnet test`

## Files to Modify

| File | Create/Modify/Test | Note |
|---|---|---|
| (none captured, check the plan) | - | - |

## Acceptance Criteria

- Build and existing tests still pass.
- Existing tests pass
- New functionality has test coverage
- Code follows repository conventions (see CLAUDE.md)

## Dependencies

**Blocked by:**
- T002 - Create OnboardingSession POCO (+ embedded Wizard, Tier, ScopeOfWork, TimelineState, Milestone, ChangeLogEntry) with…
- T005 - Add NUnit tests for the data layer (advertiser scoping, latest-wins) in…
- T009 - NUnit service tests (create/replace, updatedAt stamp) in Tests/OnboardingSessionServiceTests.cs
- T016 - NUnit tests: 403 for non-admin on each privileged op; change-log appended; cascade math
- T018 - Write/read onboarding_status on the app_config collection (not_started/in_progress/complete) on create/approve.…
- T019 - NUnit test for book-meeting success/failure response

**Blocks:**
- none

## Testing Notes

```bash
dotnet test
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/mmm-portal-api` tasks in one PR on branch `impl/prd061-backend`.

---

**Parent Task:** AIM Onboarding Decision Tool — Kochava/mmm-portal-api (20 tasks · 1 PR)
**PRD:** docs/16-PRDs/061-aim-onboarding-decision-tool