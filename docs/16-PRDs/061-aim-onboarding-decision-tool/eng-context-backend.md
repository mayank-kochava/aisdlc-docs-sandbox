---
id: eng-context-backend
title: Engineering Notes - Backend
---

## Engineering Notes: AIM Onboarding Decision Tool - Backend

**Engineer:** Mayank Ukey (capturing; backend owner Satish Karunanithi)
**Date:** 2026-06-11
**Based on:** product-spec-v2.md + product-spec-v2-reconciliation.md (reconciliation wins on conflict)

---

## Initial Thoughts

- Backend is in Phase 1 now (was localStorage-only in spec-v2; reconciled to Mongo hybrid). Home is **`mmm-portal-api` (C# .NET 7)** — it already owns MongoDB (db `aim`). `mmm-attribution-api` only reads Mongo; not involved.
- This is a textbook CRUD-over-Mongo feature with an audit log and a couple of privileged actions. The patterns already exist — **mirror the SavedView stack** (model + data layer + service + controller, advertiser-scoped). Low conceptual risk except for one thing: server-side role enforcement.
- The one genuinely new/hard piece is **super-admin (CSM) enforcement server-side**. Everything else is wiring.

---

## Concerns & Risks

- **Super-admin enforcement (the real one).** portal-api has NO server-side role concept today. `ICurrentUserContext` carries org scope only ("auth removed"); it reads client-supplied `x-user-id`/`x-user-name` (untrusted). `is_as_admin` lives in mos-iam-server (Spanner) and only reaches the **frontend** via `/organizations/{orgId}/my-details` JSON. Oathkeeper rules for portal use `mutators: noop` — no role header injected. So today a client could call a CSM-only endpoint directly. spec-v2 §4.6 forbids that (must 403). **Good news:** the platform already has the primitive — Keto `role.RootUser` tuples checked by Oathkeeper's `remote_json` authorizer (used on `mos:iam:admin` routes). It's just not wired to portal routes yet. Path forward = inject a trusted role header on the portal Oathkeeper rule (Keto-checked) and read it in portal-api. This pulls **infra** into the loop.
- **Audit log integrity.** Change Log (milestone status/delay/notes, tier override, approval, unlock) must be append-only and server-stamped (timestamp + user email from session) — never trust a client-sent log entry. Append server-side on the dedicated CSM-action endpoints, not on autosave PUT.
- **Advertiser scoping.** Every read/write filtered by `advertiserId` (SavedView `GetUserScopedFilter` pattern) so one advertiser can't read/modify another's session.
- **Email failure path.** "Book my meeting" → SES. Frontend needs a real success/failure signal for its retry UX (spec §4.2 Tab 1 mermaid). Endpoint returns clear 200 vs error.

---

## Decisions (this session)

- **Session model:** one active `aim_onboarding_sessions` doc per `advertiserId` (latest wins). No multi-session history in Phase 1 (that's the Phase 2 dashboard).
- **PUT autosave:** full-doc replace. Frontend holds canonical state; PUT sends the whole session, backend replaces + bumps `updatedAt`. Change-log entries are NOT written on autosave PUT — only on the CSM-action endpoints, server-side.
- **Super-admin mechanism (direction):** Oathkeeper-injected trusted role header (Option 1), Keto-checked — matches the existing `role.RootUser` pattern and Mayank's "Oathkeeper checks role." Final call + the Oathkeeper/Keto wiring is Satish + infra (see Q1).

---

## Questions & Answers

### Q1: Confirm the super-admin enforcement mechanism + who wires Oathkeeper/Keto

- **For:** Backend (Satish) + Infra
- **Status:** 🟡 Direction set, needs confirmation
- **Answer:** Recommended: extend the portal Oathkeeper access rule (`ko-k8s-apps/oathkeeper/.../portal-mmm-kochava-com-api.yaml`) to check Keto for the CSM/admin role and inject a trusted header (e.g. `X-Is-As-Admin`); portal-api reads it into `ICurrentUserContext` (mirror `AdvertiserContextActionFilter` reading `x-user-id`). Service-layer check returns `DataUpdateResult.AccessDenied` → `HandleDataUpdateResult` → 403. Need: (a) Satish's sign-off, (b) what Keto tuple/namespace represents "CSM" (is it `role.RootUser`, or a new tuple?), (c) infra owner for the Oathkeeper rule change.
- **Impact:** Blocks CSM Approval, Unlock to Edit, timeline status edits, and tier-override logging. Everything else can proceed.

### Q2: `onboarding_status` on `app_config` — field name + shape

- **For:** Backend (Satish)
- **Status:** 🔴 Open
- **Answer:** *Pending response*
- **Impact:** Portal entry-point gating. Confirm the exact field on the `app_config` doc (values not_started / in_progress / complete) and who writes it (portal-api on session create/approve?). Verify against Mongo when connection string available.

### Q3: CSM recipient list config

- **For:** PM (Gary) / Backend
- **Status:** 🟡 Partially answered
- **Answer:** Phase 1 = env/config (`CsmTeamEmail` in appsettings), initial values Kade + Jacob. Phase 2 = advertiser's assigned CSM lookup. Confirm appsettings key name + whether it's a list.
- **Impact:** "Book my meeting" email recipients.

### Q4: SES template ownership

- **For:** Backend
- **Status:** 🔴 Open
- **Answer:** *Pending response*
- **Impact:** New SES templates (`OnboardingMeetingRequested.html/.txt`) as embedded resources, mirroring existing `EmailService` templates. Copy/subject content TBD.

---

## Rough Scope

Mirror the SavedView stack in `mmm-portal-api`:

- `DataAccess/Mongo/CollectionNames.cs` — add `AimOnboardingSessions = "aim_onboarding_sessions"`.
- `DataAccess/Mongo/Models/OnboardingSession.cs` — POCO + BSON attrs (wizard, tier, scopeOfWork+approval, timeline.milestones, changeLog).
- `DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs` (+ interface) — advertiser-scoped CRUD; get-latest-by-advertiser; change-log append.
- `Services/OnboardingSessionService.cs` (+ interface) — status transitions, approve/unlock, tier-override, super-admin gate (AccessDenied → 403), email via `IEmailService`.
- `Api/Areas/App/Controllers/OnboardingSessionsController.cs` — routes under `/mmm/advertisers/{advertiserId}/onboarding-sessions`: GET (latest), POST (create), PUT (autosave, full-doc replace), POST `/book-meeting`, POST `/approve` (super-admin), PUT `/sow-approval`, PUT `/timeline` (super-admin), POST `/unlock` (super-admin).
- DTOs in `Kochava.Aim.Portal.Models`.
- `appsettings` — add `CsmTeamEmail`. New SES templates.
- **Infra (ko-k8s-apps):** Oathkeeper rule change to inject the role header (Q1).
- `onboarding_status` write on `app_config` (Q2).

---

## Dependencies on Other Roles

- **Infra:** Oathkeeper access-rule change for portal mmm routes + Keto tuple for the CSM/admin role (Q1). This is the critical-path dependency.
- **Frontend:** persistence contract (endpoint shapes, full-doc PUT, save cadence), and it consumes the 403s for CSM-action gating.
- **PM (Gary):** recipient list (Q3), email copy (Q4).

---

## Notes

- Mongo connection topology in infra memory (tunnels via pyro.mchnad.com). Connection string to be provided to verify `app_config` shape + the new collection.
- Reference patterns: `SavedViewsDataLayer.cs`, `SavedViewsController.cs`, `AdvertiserContextActionFilter.cs` (header → `ICurrentUserContext`), `EmailService.cs` (SES + embedded templates), `DataUpdateResult.AccessDenied` → `ForbidWithReason` (403).
- Existing collections are snake_case (`saved_views`, `incremental_reports`, `app_config`) — `aim_onboarding_sessions` fits.

---

*Captured by `/kochava:eng-context backend` on 2026-06-11*
