# TG2: Run tests & validate build

## In short

Run the full build and tests for Kochava/mmm-portal-api with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG2 |
| Repo | Kochava/mmm-portal-api |
| Phase | Phase 18 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/mmm-portal-api with every task above merged. Do not edit source.

This task is part of AIM X Investment Dashboard.

### `mmm-portal-api`
**Technology**: C# / ASP.NET Core, .NET 7.0, MongoDB.Driver, Autofac, AutoMapper
**Key Files**:
- `Areas/App/Controllers/CostCurvesController.cs` — structural template for new controllers
- `Attributes/RequireSubscriptionAttribute.cs` — template for future (deferred) account-type gate
- `DataAccess/Mongo/DataLayers/Abstract/MongoDataLayerBase.cs` — base class for new DataLayer
- `DataAccess/Mongo/Models/DateSpend.cs` — reusable raw-spend-record shape
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| `InvestmentDashboardController` — per-network pacing/targets/raw daily spend | New | M | Mirrors `CostCurvesController.cs`'s multi-scope GET pattern. Returns raw time-series (MTD computed client-side per confirmed decision). |
| `NetworkIntelligenceController` — metadata-DB-backed half (model updated, MAPE, confidence, last spend, investment strategy, cost curve, comparison table) | New | M | Single new collection read; no cross-DB complexity since metadata DB is confirmed shared. |
| AI-insight sub-endpoint or pass-through | New | M | **Blocked** pending ML Services confirmation of access mechanism (direct Mongo read now more likely, not yet explicit) — see Open Questions. |
| `SystemHealthController` — aggregate rollup endpoint | New | H | Implements FR-011's multi-factor rule (MAPE ≥20%, ≥2 stale networks >7d, any 3-7d stale, any unconfirmed forward budget) — genuinely new logic, no existing template. |
| `NetworkIntelligenceDataLayer` + collection registration | New | M | Auto-registered via existing Autofac convention scanning — no manual DI wiring. |
| New DTOs (`NetworkPacingDto`, `NetworkIntelligenceDto`, `SystemHealthDto`) + AutoMapper profiles | New | M | Follow existing DTO/profile conventions in `Kochava.Aim.Portal.Models`/`Services/AutoMapperProfiles`. |
| Config Warning Banner / budget-confirmation data source | New/Modify | M | Check whether `OptimizationScenariosService`'s existing budget concepts can be extended, or a new lightweight budget-confirmation model is needed. |
| `RequireAccountType` attribute (Sprinkler-based) | Deferred | — | **Explicitly out of scope for this release** — subsequent-release gate, per phased decision. |
| Integration test infrastructure (`WebApplicationFactory`) | New | M | Recommended Phase 0 addition — see Standards Check. |
| `CLAUDE.md` | New | L | Recommended, not blocking. |

## Summary
Build a new Investment Dashboard page (plus two slide-in drawers, an onboarding tour, and a Help & Definitions panel) inside `frontend-mos`'s existing `@mos/advertiser` package, nested in the existing "AIM MMM" nav section. It's backed by three new/extended endpoints on `mmm-portal-api` (C#/.NET 7) reading from a shared Metadata MongoDB, and gates `AIM X`/`AIM Pro` access via new `feature_flags[]` entries following the existing FAA-vs-paid-tier pattern. Backend API-level account-type enforcement is explicitly deferred to a subsequent release (Sprinkler-based authenticator, not built in this phase).
**Scope correction (9 Jul 2026):** an earlier draft of this plan incorrectly scoped `mmm-attribution-api` changes into this phase for "Attribution/MMP comparison." That feature is explicitly **Out of Scope for Phase 1** per `product-spec-v3.md` Section 6 — it's AIM Pro-only and surfaced only as a static `AimProCard` upsell in this release, with no working comparison feature behind it. `mmm-attribution-api` research in `eng-research.md` remains useful groundwork for whenever that future phase is built, but **no `mmm-attribution-api` changes are part of this plan.**

## Technical Context
**Language/Version**:
- `mmm-portal-api`: C# / ASP.NET Core, .NET 7.0
- `mmm-attribution-api`: TypeScript / Node.js, GraphQL Yoga v5
- `frontend-mos`: Vue 3.5 (Composition API), TypeScript
**Primary Dependencies**:
- Backend: Autofac (DI), AutoMapper, MongoDB.Driver 2.22, FluentValidation, Swashbuckle
- GraphQL: GraphQL Yoga + Envelop, `mysql2`, `mongodb`, `ioredis`, Inversify (DI)
- Frontend: Vuetify 3.12, Pinia, vue-router, Highcharts (via `@mos/core` wrapper only)
**Storage**:
- Metadata MongoDB (shared instance across `mmm-portal-api` and ML Services) — new `network_intelligence` collection, keyed on app ID + region + advertiser ID per FR-039
- Aurora MySQL (Attribution store, read-only, unchanged) — existing `attribution_{appId}`/`potential_{appId}` tables
**Testing**:
- `mmm-portal-api`: NUnit + Moq + FluentAssertions, service-layer unit tests only today (no HTTP integration tests exist — see Standards Check)
- `mmm-attribution-api`: Jasmine (unit + integration specs against Docker DB fixtures)
...

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
- T001 - Scaffold a WebApplicationFactory-based HTTP integration test host in…
- T002 - Run dotnet test across all mmm-portal-api projects to confirm a clean baseline before feature work begins
- T013 - Implement FR-041 no-target-state (targetAmount: null under an active Platform Filter) branch in…
- T017 - Unit tests for InvestmentDashboardService in Kochava.Aim.Portal.Tests/InvestmentDashboardServiceTests.cs…
- T018 - HTTP integration test for the investment-dashboard endpoint using the WebApplicationFactory harness from T001
- T020 - Implement FR-043 Unknown state detection (signal not computable, e.g
- T024 - Unit tests for FR-043 Unknown-state detection as a distinct path from "stale"
- T025 - HTTP integration test for the system-health endpoint using the WebApplicationFactory harness
- T031 - Unit tests for NetworkIntelligenceService in Kochava.Aim.Portal.Tests/NetworkIntelligenceServiceTests.cs, covering…
- T032 - HTTP integration tests for both intelligence endpoints using the WebApplicationFactory harness
- T035 - Implement FR-044's persist-on-failure rule (retain last successful generatedAt/content, never show a "failed" state…
- T036 - Add CLAUDE.md to mmm-portal-api documenting the new domain surface (recommended in eng-plan.md Standards Check)
- T039 - Full dotnet test pass across mmm-portal-api before merge

**Blocks:**
- none

## Testing Notes

```bash
dotnet test
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/mmm-portal-api` tasks in one PR on branch `impl/prd074-backend`.

---

**Parent Task:** AIM X Investment Dashboard — Kochava/mmm-portal-api (40 tasks · 1 PR)
**PRD:** docs/16-PRDs/074-aim-x-investment-dashboard