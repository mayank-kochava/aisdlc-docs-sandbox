---
id: eng-research
title: Engineering Research
---

## Research: AIM Onboarding Decision Tool (Phase 1)

**Feature**: 061-aim-onboarding-decision-tool
**Date**: 2026-06-11
**Phase**: 0 — Research & Technology Decisions

## Overview

Validates the spec assumptions in `product-spec-v2.md` (as amended by `product-spec-v2-reconciliation.md`) against the real K4A codebases and records the technology/architecture decisions for Phase 1. Repositories explored: `frontend-mos` (Vue 3 / Vuetify), `mmm-portal-api` (C# .NET 7, MongoDB owner), `ko-k8s-apps` (Oathkeeper/Keto access rules), plus identity-layer references (`mos-iam-server`, Kratos/Keto). Scope: where the wizard lives, how it persists, and how CSM (super-admin) actions are enforced server-side.

---

## Technology Stack Decisions

### Decision: Frontend framework & embedding

**Chosen**: Vue 3.5.30 + Vuetify 3.12.4, embedded as a 3rd tab in the existing `MmmInsightsConfiguration.vue` (`packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/`). Existing route `/advertisertools/mmmconfigurations`.

**Rationale**:

- The target portal is Vue/Vuetify; spec-v2 §9 "React app with React Router" is wrong (reconciliation R4).
- The view already uses `v-tabs` + `v-window` (Marketplaces, Validate Onboarding) — adding a tab is the established extension point. The Validate Onboarding tab is the downstream of the Data Schema CSV (spec §4.2 Tab 3 / B8).
- `v-stepper` already in use for multi-step flows (`views/MmmOptimization/ScenarioCreateView.vue`, numbered `#[item.N]` slots) → reuse for the 8 steps.

**Alternatives Considered**:

1. **Separate route/standalone app** — Rejected: contradicts the embed decision; duplicates shell/auth.

**Implementation Notes**:

- API via shared `mmmPortalApi` axios instance (`views/Analytics/services/mmm.ts`); auth headers auto-injected by interceptor. Service object mirrors `services/incrementality.ts`.
- CSM gate (UI) = `useOrganizationStore().isASAdmin` (`packages/core/src/stores/organization.ts`).

### Decision: Backend stack & home

**Chosen**: `mmm-portal-api` (C# .NET 7), MongoDB.Driver. New CRUD mirroring the **SavedView** stack.

**Rationale**:

- portal-api already owns MongoDB (db `aim`; client singleton in `Startup.cs`); `mmm-attribution-api` only reads Mongo.
- SavedView is the closest analog: per-advertiser, user-scoped document with CRUD + DTOs + controller.
- Data layers auto-register via Autofac assembly scanning (`DataAccess/IocModule.cs`, `Where(typeof(IMongoDataLayer).IsAssignableFrom)`) — a new `OnboardingSessionDataLayer` needs **no manual DI**.

**Implementation Notes**:

- Collection `aim_onboarding_sessions` (snake_case, matches `saved_views`/`incremental_reports`/`app_config`). Add to `CollectionNames.cs`.
- Email via existing `EmailService` (AWS SES + embedded templates).

---

## Architectural Patterns

### Decision: Pure logic in framework-agnostic TS modules

**Chosen**: Lift `recommendTier`, `buildProvisionPlan`, validation, and flag computation into `.ts` modules under the tab dir, with the 8 prototype bug-fixes baked in; unit-test in isolation; `useAimOnboarding.ts` composable calls them.

**Rationale**: This is the correctness-critical, bug-prone part (spec §9). Keeping it pure + tested decouples it from UI churn and makes "works end-to-end" verifiable. Vitest supports plain TS unit tests directly.

**Alternatives Considered**:

1. **Inline in composable** — Rejected: harder to unit-test, couples logic to Vue reactivity.

### Decision: Persistence — Mongo hybrid, one doc per advertiser, full-doc-replace PUT

**Chosen**: localStorage buffer within a step; `PUT` full session doc to Mongo on step/tab switch + manual save. One active `aim_onboarding_sessions` doc per `advertiserId` (latest wins). Audit `changeLog` embedded array, appended **server-side** only on CSM-action endpoints (not on autosave PUT).

**Rationale**:

- Reconciliation R1: backend is in Phase 1. Full-doc replace is simplest (frontend holds canonical state); server stamps `updatedAt`.
- One-per-advertiser matches Phase 1 (multi-session dashboard is Phase 2).
- Server-stamped, append-only audit log prevents client tampering (timestamp + session email).

**Implementation Notes**:

- Advertiser scoping via the `GetUserScopedFilter` pattern in `SavedViewsDataLayer`.
- Endpoints under `/mmm/advertisers/{advertiserId}/onboarding-sessions`: GET(latest), POST, PUT (autosave), POST `/book-meeting`, POST `/approve`, PUT `/sow-approval`, PUT `/timeline`, POST `/unlock`.
- New Oathkeeper rule needed to route `…/onboarding-sessions<.*>` (mirror the existing `mmm-portal-advertiser-saved-views` rule).

### Decision: Server-side super-admin (CSM) enforcement via portal-api → mos-iam lookup

**Chosen**: Enforce CSM-only actions in the service layer (`DataUpdateResult.AccessDenied` → `HandleDataUpdateResult` → 403 `ForbidWithReason`). The admin/CSM signal is obtained by **portal-api calling mos-iam server-side** for the caller's `is_as_admin` (the same signal the frontend gates on via `/organizations/{orgId}/my-details`), keyed by the session user id. **Not** via Oathkeeper header injection — that path is closed (see below).

**Why Oathkeeper injection is ruled out (ko-diagnostic-toolkit research, 11 June):**

- The portal api-key authorizer (`remote_json` → `nng-sprinkler-auth /auth/z/account`) returns **status only** (200/403, no body) — verified in `nng-sprinkler-auth/.../accountauthz/check_handler.go`. Oathkeeper `remote_json` checks status and **cannot forward a role/`.Extra` field**; portal rules use `noop` mutators.
- API-key routes live in the **legacy identity domain** (sprinkler-api, numeric `user_id`, `kochava_accounts`); `is_as_admin` lives in **mos-iam** (Spanner, UUID). No bridge exists for api-key routes. So there is no `.Subject` and no injectable admin claim on these routes.

**Signal options (confirmed available):**

- **`is_as_admin`** (mos-iam, user-level) — matches the frontend's existing CSM gate. **Recommended** for parity.
- **`ROLE_CSM`** keto role (mos-iam `internal/common/constants.go`) — more precise "CSM" primitive; `is_as_admin` ≠ CSM strictly (the "super-admin = CSM" equivalence is NOT code-verified). Confirm with identity team which is authoritative for "CSM mode."
- **Legacy `IsKochavaUser(api_key)`** = api_key belongs to `account_id = 5` (Kochava internal), in `sprinkler-api/.../customerapps/app_get.go` — the in-domain primitive for api-key routes if a mos-iam call is undesirable.

**Implementation Notes**:

- No Oathkeeper change for the role check (only a new route rule mirroring `mmm-portal-advertiser-saved-views` to expose `…/onboarding-sessions<.*>`).
- portal-api: server-side client/cache to mos-iam to resolve `is_as_admin`/`ROLE_CSM` for the caller; surface it on `ICurrentUserContext`; service-layer gate on the CSM actions. Cache to avoid a per-request hop.

---

## Performance Considerations

### Decision: Client-side compute + debounced autosave

**Chosen**: All output generation (SoW, Timeline, Data Schema) is client-side from the flat state. Autosave debounced; save-status indicator (saving/saved/failed). Phase 9 loader is a 1.6s animation (no backend call — data already autosaved).

**Rationale**: Meets spec §10 (step render < 100ms, output < 500ms — all client-side). Debounce avoids PUT-per-keystroke and the autosave race risk flagged in eng-context-frontend.

**Implementation Notes**: last-write-wins with `updatedAt`; guard against overlapping in-flight PUTs (dirty flag / cancel stale).

---

## Error Handling Strategy

### Decision: Explicit failure states

**Chosen**:

- **"Book my meeting"**: endpoint returns clear 200 vs error; frontend renders sent ✓ / try-again per spec §4.2 Tab 1 mermaid.
- **CSM actions**: service returns `AccessDenied` → 403; frontend treats 403 as "not authorized" (shouldn't happen if UI gating correct, but enforced regardless).
- **Wizard validation**: non-blocking; surfaced as a banner with jump-back links at the output screen (spec §4.1).

**Error Categories**:

- Auth/role: 403 (super-admin gate).
- Validation: client-side banner, non-blocking.
- Email/transport: surfaced with retry.

---

## Testing Strategy

### Decision: Vitest (frontend) + NUnit/Moq (backend), unit-first

**Chosen**: Unit-test the pure logic modules + composables + key components (Vitest); unit-test data layer + service (NUnit + Moq, mock `IMongoCollection`/data layer); optional Playwright e2e for the vertical slice.

**Test Types**:

| Test Type | Scope | Tools |
|-----------|-------|-------|
| Unit (FE) | `recommendTier`, `buildProvisionPlan`, validation, composable save logic, step/output components | Vitest 4.1.2, @vue/test-utils 2.2.6, createTestingPinia |
| Unit (BE) | `OnboardingSessionService` (approve/unlock/override/super-admin gate), data-layer filters | NUnit 3.13.3, Moq 4.20.69, FluentAssertions |
| E2E (opt) | Vertical slice: embed → step → Mongo round-trip → output | Playwright 1.50.1 |

**Implementation Notes**:

- FE pattern files: `…/ValidateOnboardingTab/composables/__tests__/useValidateOnboarding.spec.ts` (composable), `packages/core/src/components/inputs/__tests__/InputDateRange.spec.ts` (component). Run `npm run test:ci`.
- BE pattern: `Kochava.Aim.Portal.Tests/AccountsServiceTests.cs` (Moq + NUnit). Run `dotnet test`.
- The 8 prototype bug-fixes each get a unit test (esp. annual→monthly budget conversion, 12mo history threshold).

---

## Specification Validation

### Confirmed Assumptions

| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| Embed in MmmInsightsConfiguration | `MmmInsightsConfiguration.vue` (v-tabs+v-window) | Validate Onboarding tab already present |
| Stepper component exists | `MmmOptimization/ScenarioCreateView.vue` | v-stepper numbered slots |
| portal-api owns Mongo, SavedView pattern | `SavedViewsDataLayer.cs`, `IocModule.cs`, `Startup.cs` | Autofac auto-registration |
| snake_case collections | `CollectionNames.cs` | `aim_onboarding_sessions` fits |
| SES email available | `EmailService.cs`, appsettings AWS:SES | Add template + CsmTeamEmail |
| 403 mechanism exists | `DataUpdateResult.AccessDenied` → `ForbidWithReason` | Service-layer gate |

### Corrected Assumptions

| Spec Statement | Actual Reality | Impact |
|----------------|----------------|--------|
| React app + React Router (§9, §6) | Vue 3 / Vuetify tab embed | Use Vue; prototype is reference only |
| localStorage-only, no backend (§9, §4.4) | Mongo hybrid; backend in Phase 1 | New portal-api CRUD + Oathkeeper route |
| `campaignMerge` state key | `campaignGrouping` (mockup-k4a.html) | Use `campaignGrouping` in code |
| Super Admin "automatic" server-side | No role reaches portal-api today | Oathkeeper/Keto wiring required (Q1) |

---

## Open Questions

| Question | For | Blocking |
|----------|-----|----------|
| Q1: ~~How to obtain a trusted CSM/admin signal on API-key routes?~~ **RESOLVED (toolkit research):** Oathkeeper can't (sprinkler-auth is status-only). Mechanism = portal-api calls mos-iam server-side for `is_as_admin`/`ROLE_CSM`. | — | Resolved |
| Q2: Which signal is authoritative for "CSM mode" — `is_as_admin` (frontend parity) or the `ROLE_CSM` keto role? And confirm the mos-iam endpoint portal-api should call. | Satish + Identity | Wiring the server-side gate (mechanism decided; signal field TBD) |
| Q3: `onboarding_status` field name/shape on `app_config`, and who writes it | Satish | Portal entry-point gating |
| Q4: Data Schema downloadable template — exact schema rows + CSV columns per source | Gary / Data | Data Schema tab content (not shell) |
| Q5: SES recipient config key (list?) + email copy for `OnboardingMeetingRequested` | Gary / Backend | "Book my meeting" email |

---

## Summary

Stack and patterns are confirmed against the codebases; the build is well-precedented (SavedView CRUD + Vuetify stepper/tabs + pure-TS logic). **Server-side CSM enforcement is now mechanism-resolved** (ko-diagnostic-toolkit research): Oathkeeper injection is impossible (sprinkler-auth status-only, `remote_json` can't forward, api-key routes have no Subject), so portal-api verifies server-side by calling mos-iam for the caller's `is_as_admin`/`ROLE_CSM`. The only remaining question is which signal field is authoritative (Q2) — not a blocker for planning. The recommended first move is the vertical slice (embed → step → Mongo round-trip → stub output) to lock the persistence contract early.

**Decisions Made**: 9
**Open Questions**: 4 (CSM mechanism resolved; remaining are signal-field confirmation + content/config)
**Status**: Ready for Implementation Planning
