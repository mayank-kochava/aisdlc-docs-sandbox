---
id: eng-research-v3
title: Engineering Research (v3 — Data Mapping Tab)
---

## Research: AIM Onboarding Decision Tool — Data Mapping Tab (v3)

**Feature**: 061-aim-onboarding-decision-tool (Data Mapping tab enhancement)
**Date**: 2026-08-06
**Phase**: 0 — Research & Technology Decisions

## Overview

Validates `product-spec-v3.md`'s assumptions against the real `mmm-portal-api` (C# .NET, MongoDB) and `frontend-mos` (Vue 3 / Vuetify) codebases on `main`, and against `data-mapping-tab-implementation-plan.md`'s design. This is a delta research pass — the original `eng-research.md` (2026-06-11) already validated the shipped 3-tab scope; this one covers only what's new: the `FieldMapping` persistence model, its two endpoints, and the new AIM Metric/Schema Element reference-data collection.

---

## Architectural Patterns

### Decision: `FieldMapping` follows the `Timeline`/`SoW` targeted-update pattern, exact shape now confirmed

**Chosen**: Two new endpoints on `OnboardingSessionsController`, mirroring `EditTimelineAsync`/`ApproveSowAsync` exactly — not a guess, read directly from `main`.

**Rationale**:

- `IOnboardingSessionService` confirms the real method-naming convention: `EditTimelineAsync(id, TimelineEditDto)` for the "stays editable" targeted update, `ApproveSowAsync(id, SowApprovalDto)` for the terminal approval action. `FieldMapping` needs one of each: `EditFieldMappingAsync(id, FieldMappingEditDto)` and `ApproveFieldMappingAsync(id, FieldMappingApprovalDto)`.
- `OnboardingSessionsController`'s real routes are flat, hyphenated, verb-appropriate — `PUT {id}/timeline`, `PUT {id}/sow-approval`, `POST {id}/approve`, `POST {id}/tier-override`, `POST {id}/book-meeting` — never nested sub-paths like `/timeline/edit`. The implementation plan's guessed `field-mapping/approve` (nested, POST) should be **`PUT {id}/field-mapping-approval`** (flat, PUT) — it's modeled on `sow-approval`'s shape (sets approver name/job title/terminal timestamp), not on the CSM `approve` flag-flip action, which is a different concept (`ScopeOfWork.CsmApproved`, a precondition-only boolean).
- Confirmed exact `ChangeLogEntry` shape from `EditTimelineAsync`'s real body: `{ At, UserEmail, Type, Field, OldValue, NewValue }`. `FieldMapping`'s entries should follow the same shape — e.g. `Type: "field_mapping_updated"` / `"field_mapping_approved"`, `UserEmail` from `_currentUser().UserId` (confirmed: `ICurrentUserContext` has no `.Email`, `UserId` **is** the identity/email string).
- Confirmed `ApproveSowAsync`'s real precondition check: it requires `session.ScopeOfWork.CsmApproved == true` *before* the client can approve — a two-step gate (CSM clicks "Approve" first, unlocking the client's own SoW sign-off) that the spec's Flow 1/2 already describe correctly. This is NOT the same gate as `FieldMapping`'s "SoW must be approved first" — that one checks `ScopeOfWork.ApprovedAt != null` (the *client's* SoW approval), a separate, later condition. Confirms the implementation plan's design is right, just needed the real precedent to point at.
- Confirmed the server-stamps-its-own-clock pattern: `SowApprovalDto.ApprovedAt` exists on the DTO but is deliberately unused server-side (comment: "the server always stamps its own clock... never trusts this value... exists on the DTO only so tests can assert the server ignores a stale/malicious client-supplied one"). `FieldMappingApprovalDto` should follow the identical shape/comment, not omit the field.

**Alternatives Considered**:

1. **Nest both under `/field-mapping/*`** (edit at `/field-mapping`, approve at `/field-mapping/approve`)
   - Pros: reads slightly more RESTful
   - Cons: doesn't match this controller's actual convention (every other multi-word action is a flat hyphenated segment, never nested)
   - Rejected: inconsistent with `sow-approval`, `tier-override`, `book-meeting`

**Implementation Notes**:

- `PUT {id}/field-mapping` → `EditFieldMappingAsync` (mirrors `EditTimeline` action method's shape, including its `[FromBody]` DTO validation pattern — though unlike Timeline's "exactly one field" rule, `FieldMappingEditDto` takes the whole `Overrides` dict at once, so that specific validation doesn't port over).
- `PUT {id}/field-mapping-approval` → `ApproveFieldMappingAsync` (mirrors `SowApproval` action method's shape exactly, including its manual 409 handling for `NameConflict` — `HandleDataUpdateResult` alone doesn't cover that case, per the real `SowApproval` action).

---

### Decision: AIM Metric/AIM Schema Element reference data mirrors the existing `GeoLocations` pattern — not a new pattern

**Chosen**: Extend the existing `IReferenceDataService`/`ReferenceDataService` with a new method, backed by a new minimal data layer mirroring `GeoLocationsDataLayer` exactly, exposed via a new thin controller mirroring `GeoLocationsController` exactly.

**Rationale**:

- This codebase already has an established, working pattern for **exactly this kind of data** — small, shared, non-advertiser-scoped, read-mostly reference lists. `GeoLocationsDataLayer.GetAsync()` is the simplest example: `GetCollection<GeoLocation>(...).Find(Builders<GeoLocation>.Filter.Empty)` — no advertiser filter at all, because it's global data every advertiser reads the same copy of.
- `GeoLocationsController` confirms the controller shape for this class of endpoint: route `[area]/[controller]` (**not** nested under `advertisers/{advertiserId}/...` like `OnboardingSessionsController` — this is the single most important correction to the implementation plan, which incorrectly assumed the new reference endpoint would live under the onboarding-sessions controller's advertiser-scoped base route). A single `[HttpGet("")]` action delegates straight to the service.
- `ReferenceDataService` already exists as the shared home for this class of lookup (`IGeoLocationsDataLayer` is injected into it directly) — the new AIM Metric/Schema Element data layer should be injected into this *same* service as a new dependency, not a new standalone service, unless there's a reason to keep onboarding-specific reference data separate (no such reason found — this data isn't onboarding-specific in principle, Model Config Phase 2 will likely want the same list).
- `NetworksDataLayer` shows the soft-delete variant of the same pattern (`Filter.Ne(x => x.Deleted, true)`) — worth considering for the AIM Metric/Schema Element collection too, since Data Engineering removing a value (the OQ-15 scenario) is cleaner as a soft-delete (`Deleted: true`, filtered out of `GetAsync()`) than a hard delete, since it preserves the value for any already-approved `FieldMapping.Overrides` records that reference it, partially informing (but not fully resolving) OQ-15.

**Alternatives Considered**:

1. **New standalone `AimMappingReferenceDataLayer` + own service**, not touching `ReferenceDataService`
   - Pros: keeps onboarding-specific concerns isolated
   - Cons: this data isn't conceptually onboarding-specific (Model Config, Phase 2, will likely need the same list); duplicates a pattern that already exists for exactly this purpose
   - Rejected: no isolation benefit found; `ReferenceDataService` is the established home for "small global lookup list," full stop

**Implementation Notes**:

- New model, e.g. `AimMetric { Value, Label, Deleted }` / `AimSchemaElement { Value, Label, Deleted }` (two small collections, or one collection with a `Type` discriminator — either is consistent with existing conventions; `GeoLocation`/`Network`/`Region` are each their own collection, suggesting two separate collections is the more idiomatic choice here).
- `IReferenceDataService` gains `GetAimMetricsAsync()` / `GetAimSchemaElementsAsync()`, mirroring `GetGeoLocationsAsync()`'s one-line delegation + AutoMapper shape exactly.
- New controller (or a new action on an existing thin reference controller, if one groups several already — check before assuming a 1:1 controller-per-resource split is universal) exposing `GET mmm/aim-metrics` and `GET mmm/aim-schema-elements` — **not** nested under the onboarding sessions route.
- Corrects the implementation plan's assumed endpoint shape (`GET {baseRoute}/aim-mapping-reference`, implicitly under the advertiser-scoped onboarding base route) — the real precedent says this should be its own unscoped, `ReferenceDataService`-backed endpoint(s).

---

## Specification Validation

### Confirmed Assumptions

| Spec/Plan Statement | Verified By | Notes |
|----------------------|-------------|-------|
| `Timeline` stays editable post-SoW-approval via a targeted update, no `ApprovedAt` guard | `OnboardingSessionService.EditTimelineAsync`, `OnboardingSessionDataLayer.UpdateTimelineAsync` (main) | Confirmed exactly as the implementation plan described. |
| `ApproveSowAsync` server-stamps its own approval timestamp, never trusts the client's | `SowApprovalDto.cs` comment + `OnboardingSessionService.ApproveSowAsync` body | Confirmed — `FieldMappingApprovalDto` should follow the identical pattern. |
| Changelog entries are server-constructed with author identity from the session, never free text | `ChangeLogEntry` construction in `EditTimelineAsync`/`ApproveSowAsync` | Confirmed — `UserEmail = _currentUser().UserId`. |

### Corrected Assumptions

| Spec/Plan Statement | Actual Reality | Impact on Implementation |
|----------------------|-----------------|---------------------------|
| New `FieldMapping` endpoints would be `PUT {id}/field-mapping` and `PUT {id}/field-mapping/approve` (nested) | Real convention is flat, hyphenated action segments (`sow-approval`, `tier-override`, `book-meeting`) | Approval endpoint should be `PUT {id}/field-mapping-approval`, not a nested `/approve` sub-path. |
| AIM Metric/Schema Element reference data would live under the onboarding-sessions controller's advertiser-scoped route | The codebase has an established, unscoped reference-data pattern (`GeoLocationsController`, route `[area]/[controller]`, backed by `ReferenceDataService`) for exactly this kind of shared lookup list | Build on the existing `ReferenceDataService`/`GeoLocationsDataLayer` pattern instead of inventing a new one nested under onboarding sessions. |
| `patchFieldMapping` (frontend) would itself call the new backend endpoint, "mirroring `patchTimeline`'s pattern (dedicated endpoint call)" | `patchTimeline` does **not** call the API — it's a pure local-state mirror-sync function. `OutputScreen.vue` calls `onboardingService.updateTimeline(...)` directly, then calls `patchTimeline({ Milestones: updated })` afterward to sync the composable's cached copy once the server confirms success | `patchFieldMapping` should be the same: a local mirror-sync only. `DataMappingTab.vue` (not the composable) should call `onboardingService.updateFieldMapping(...)` directly, then call `patchFieldMapping(...)` on success. |

---

## Testing Strategy

| Test Type | Scope | Tools |
|-----------|-------|-------|
| Backend unit | Data-layer targeted updates, service-layer changelog/precondition logic, reference-data layer | NUnit/Moq/FluentAssertions (existing convention, confirmed in `OnboardingSessionDataLayerTests.cs` et al.) |
| Frontend unit | `buildMapping.ts` (pure, fixture-based reference lists — no network), `validateFieldMapping.ts`, `EditableCell.vue` | Vitest (existing convention) |
| Frontend component | `DataMappingTab.vue` with mocked `useAimOnboarding`, mocked reference-data fetch | Vitest + `@vue/test-utils` (existing convention, confirmed in `OutputScreen.spec.ts`) |

No new test infrastructure needed — everything follows patterns already present in both repos.

---

## Open Questions

| Question | For | Blocking |
|----------|-----|----------|
| Should the new AIM Metric/Schema Element collections use a soft-delete (`Deleted: true`, matching `NetworksDataLayer`'s pattern) rather than hard delete, to partially inform OQ-15 (product-spec-v3.md §11)? | Backend engineer / architect, once eng-plan starts | Not blocking research, but worth deciding before the data-layer implementation task, since it affects the model shape. |
| Does an existing reference-data controller already group multiple small lookups together (rather than one controller per resource, like `GeoLocationsController`)? | Backend engineer, quick codebase check during eng-plan | Not blocking — either shape (new controller vs. added action on an existing one) is consistent with precedent; just needs a quick look before committing to file layout. |

---

## Summary

Confirmed the real backend endpoint/DTO/changelog conventions (correcting the implementation plan's two guessed route/nesting details), and found a strong existing precedent (`GeoLocationsController`/`ReferenceDataService`) for the new AIM Metric/Schema Element reference data — meaning that piece needs *less* new architecture than originally planned, not more. Also corrected a frontend assumption about where the API call actually happens in the `patchTimeline` pattern.

**Decisions Made**: 2 (FieldMapping endpoint shape; reference-data architecture)
**Open Questions**: 2 (both non-blocking, resolvable during eng-plan)
**Status**: Ready for Implementation Planning

---
> Generated by: claude-sonnet-5
