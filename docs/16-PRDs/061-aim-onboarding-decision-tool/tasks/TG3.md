# Task Packet: TG3 — Run tests & validate build

**Matrix:** `wi_1eff1d855cc55619bdca-r1` · **Repo:** `Kochava/mmm-portal-api` · **Stack:** `unknown` · **Owner:** `test-change-gate` (confidence 0.90)

## Objective

Run tests & validate build

## Dependencies

- Depends on `T001` — Add `AimOnboardingSessions = "aim_onboarding_sessions"` to `Kochava.Aim.Portal.DataAccess…
- Depends on `T005` — Add NUnit tests (advertiser scoping, latest-wins, replace rejected after approval; Moq `I…
- Depends on `T006` — Create `OnboardingSessionDto` and `OnboardingSessionUpsertDto` (mirror the model, no serv…
- Depends on `T008` — Create `OnboardingSessionsController` with `GET` (latest), `POST`, `PUT /{id}` under `/mm…
- Depends on `T009` — Add NUnit service tests (create, replace stamps `UpdatedAt`, replace after approval confl…
- Depends on `T016` — Add NUnit tests: 403 for non-admin on approve and tier-override, change-log appended with…
- Depends on `T019` — Add NUnit test for book-meeting success and SES-failure responses in `Kochava.Aim.Portal.…

## Allowed Files

Touch only these paths. Anything else is out of scope for this task.

- (none captured from source -- verify against the plan before editing)

## Context

**Technology (mmm-portal-api):** C# .NET 7, MongoDB.Driver, NUnit + Moq. Mirror the SavedViews stack.

From eng-plan:

> ### Kochava/mmm-portal-api
> **Technology**: C# .NET 7, MongoDB.Driver, NUnit + Moq
> **Key Files**:
> - `Kochava.Aim.Portal.DataAccess/Mongo/Models/OnboardingSession.cs` - aggregate POCO
> - `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs` - advertiser-scoped CRUD
> - `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs` - GET/POST/PUT
> **Changes**:
> | Change | Type | Complexity | Notes |
> |--------|------|------------|-------|
> | Collection name, model, data layer + tests | New | M | Mirror SavedView; 409 on replace once approved |
> | DTOs, session service, controller + tests | New | M | No change-log on autosave |
> | mos-iam `is_as_admin` resolver on `ICurrentUserContext` | New | H | Cached; `GetUserByEmail` because `x-user-id` is the email |
> | CSM guard, `approve`, `tier-override` | New | M | `DataUpdateResult.AccessDenied` maps to 403 |
> | `sow-approval` (terminal lock, anchors timeline) | New | M | Sets `Status=complete` |
> | `timeline` edit + cascade + change log | New | H | Client and CSM; author is session email |
> | `book-meeting` SES email | New | L | `CsmTeamEmail` config; Reply-To requester |
>

## Acceptance Criteria

- [ ] Build and existing tests still pass.

## Definition of Done

- Tests pass for every file listed above under Test.
- No file outside Allowed Files is modified.
- Commit uses the conventional-commits format; PR title references `TG3`.

## Escalation

Stop and record an open question instead of guessing if: the files listed above do not match the current checkout, an interface this task consumes does not exist yet, or completing the task requires touching a file outside Allowed Files.
