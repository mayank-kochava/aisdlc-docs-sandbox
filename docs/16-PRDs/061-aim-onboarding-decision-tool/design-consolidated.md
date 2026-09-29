---
id: design-consolidated
title: "Consolidated Design: AIM Onboarding Decision Tool (Phase 1)"
---

## Consolidated Design: AIM Onboarding Decision Tool (Phase 1)

**Date**: 2026-06-11
**Status**: Approved — input for writing-plans
**Synthesizes**: product-spec-v2.md, product-spec-v2-reconciliation.md (wins on conflict), eng-context-frontend.md, eng-context-backend.md, eng-research.md, eng-tasks.md

This is the single design of record for Phase 1 implementation planning. It carries the exact contracts (model, routes, state keys) the per-workstream plans reference, so they stay consistent.

---

## 1. Goal

Replace the Excel onboarding questionnaire with an 8-step guided wizard embedded in the K4A console that collects platform/business/data inputs and produces three Phase 1 outputs — a draft Scope of Work, a delivery Timeline, and a filtered Data Schema — persisted to MongoDB, with CSM-only actions enforced server-side.

## 2. Architecture (3 repositories)

- **frontend-mos** (Vue 3.5 / Vuetify 3.12) — wizard + outputs as a **3rd tab** in `MmmInsightsConfiguration` (`packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/`). No new route (`/advertisertools/mmmconfigurations` exists). `v-stepper` wizard (pattern: `views/MmmOptimization/ScenarioCreateView.vue`); `v-tabs`/`v-window` outputs. Pure-TS logic modules + `useAimOnboarding` composable + `onboarding` service on the shared `mmmPortalApi` axios instance (`views/Analytics/services/mmm.ts`).
- **mmm-portal-api** (C# .NET 7, MongoDB.Driver) — persistence mirroring the **SavedView** stack (model + data layer + service + controller + DTOs); CSM-action endpoints with an embedded audit log; SES email; `onboarding_status` on `app_config`.
- **ko-k8s-apps** — one Oathkeeper access-rule exposing `…/onboarding-sessions<.*>` (mirror `mmm-portal-advertiser-saved-views`). Role enforcement lives in portal-api, not the gateway.

## 3. Components & responsibilities

### Frontend (`AimOnboardingTab/`)

- `AimOnboardingTab.vue` — wrapper; status routing (wizard vs read-only record); CSM mode via `useOrganizationStore().isASAdmin`.
- `steps/Step1Team … Step8Review.vue` — 8 wizard steps on a `v-stepper`; progressive disclosure; cross-step resets (business-model change clears LTV/`webFunnel`).
- `output/SowTab.vue`, `output/TimelineTab.vue`, `output/DataSchemaTab.vue` — the 3 output tabs.
- `composables/useAimOnboarding.ts` — flat state, computed flags, step nav, autosave.
- `logic/recommendTier.ts`, `logic/buildProvisionPlan.ts`, `logic/validation.ts` — framework-agnostic, unit-tested, with the 8 prototype bug-fixes.
- `services/onboarding.ts`, `interfaces/aimOnboarding.ts`.

### Backend (mirror SavedView)

- `DataAccess/Mongo/Models/OnboardingSession.cs` (+ embedded types), `CollectionNames.cs` entry.
- `DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs` (+ interface) — advertiser-scoped, Autofac auto-registered.
- `Services/OnboardingSessionService.cs` (+ interface), `Api/Areas/App/Controllers/OnboardingSessionsController.cs`, DTOs in `Kochava.Aim.Portal.Models`.
- mos-iam admin-resolution helper + `EmailService` templates.

## 4. Data flow

1. Portal opens the tab → `GET latest` → `status` routes wizard vs read-only record.
2. Edit within a step → localStorage buffer.
3. Step/tab switch + manual save → debounced **full-doc `PUT`** to Mongo (`updatedAt` stamped server-side).
4. Step 8 "Generate" → client-side compute (tier, SoW, Timeline, Data Schema) + 1.6s animation. **No backend call.**
5. "Book my meeting" → `POST /book-meeting` → SES; UI shows sent/retry.
6. CSM Approval (super-admin) → `POST /approve` → `csmApproved=true` → client SoW signoff (`PUT /sow-approval`) → `status=complete`, `approvedAt` set, timeline anchored.
7. Timeline edits (client **or** CSM) → `PUT /timeline` → status/delay (new date → preview → confirm) + cascade recompute + server-stamped `changeLog`. Tier override → `POST /tier-override` (CSM-only, asymmetric, logged).
8. Approval is terminal — no unlock/re-edit of answers (see §8a).

## 5. Persistence contract

One collection **`aim_onboarding_sessions`**, one document per `advertiserId` (latest-wins). Embeds everything (single aggregate); read/written as a unit, bounded size. Audit `changeLog` embedded; appended **server-side only** (author = session email). Data Schema is computed client-side (not stored). **`onboarding_status` is NOT a stored field — derived from this doc's `status`** (portal entry queries latest session by `advertiserId`). **Field naming: PascalCase per `SavedView`** (C# POCO). Verified vs live DB 2026-06-11: snake_case collection names, no existing onboarding collection, `app_config` is the per-app model-config doc (not an onboarding home), `config_audit_log` is the platform's separate-audit precedent (only split the changeLog out if volume grows).

```jsonc
{
  "_id": ObjectId,
  "advertiserId": "string",
  "status": "not_started | in_progress | complete",
  "createdAt": ISODate, "updatedAt": ISODate,
  "createdByUserId": "string", "updatedByUserId": "string",
  "wizard": {
    "companyName": "", "projectLeads": [{ "name": "", "email": "" }], "dataLeads": [],
    "appName": "", "platforms": [], "platformShares": {}, "modelling": "", "region": "", "regionOther": "",
    "budgetMonthly": 0, "budgetAnnual": 0, "budgetPeriod": "monthly|annual",
    "usesOffline": false, "offlineSplitPct": 20, "digitalMediaTypes": [],
    "hasAttrGaps": "", "attrGapCategories": [], "coverageConfidence": "",
    "paidSplit": { "iOS": 65, "Android": 80, "Web": 50 },
    "ua": false, "uaShare": "", "ue": false, "ueShare": "", "brand": false, "brandShare": "",
    "campaignGrouping": false, "campaignGroupingChoice": "",
    "business": "", "funnel": { "items": [], "kpi": "", "kpiConfirmed": false, "names": {} },
    "webFunnel": {}, "wantsLtv": false, "ltvCohortAvail": "", "ltvPartialChoice": "",
    "mmp": "", "mmpCollection": "", "mmpFileStorage": "", "appsflyerCohortAccess": false,
    "webAttrSources": [], "spendCollection": "", "adSpendSources": [], "history": {},
    "externalFactors": [], "externalFactorsNotes": "",
    "objectivesGoal": "", "objectivesSuccess": "", "objectivesMarketing": "",
    "uaUeRoutingFlag": "", "brandRoutingFlag": false
  },
  "tier": { "recommended": "aim_x|aim_pro", "override": null, "effective": "aim_x|aim_pro" },
  "scopeOfWork": { "sectionNotes": {}, "approverName": "", "approverJobTitle": "", "approvedAt": null, "csmApproved": false },
  "timeline": { "anchorDate": null, "milestones": [
    { "key": "", "name": "", "responsible": "", "weekOffset": 0, "duration": "", "status": "", "delayedToDate": null, "notes": "" } ] },
  "changeLog": [ { "at": ISODate, "userEmail": "", "type": "", "field": "", "oldValue": null, "newValue": null } ]
}
```

**State-key note:** use `campaignGrouping` / `campaignGroupingChoice` (verified in mockup-k4a.html), NOT `campaignMerge`.

## 6. API contract

Base: `/mmm/advertisers/{advertiserId}/onboarding-sessions`

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/` (latest) | advertiser | Portal entry; drives status |
| POST | `/` | advertiser | Create session (blocked once `approvedAt` set) |
| PUT | `/{id}` | advertiser | Autosave (full-doc replace). **Rejected 409 once `approvedAt` set — answers are locked after approval.** |
| POST | `/{id}/book-meeting` | advertiser | SES email to CSM team |
| POST | `/{id}/approve` | **CSM** | Activate SoW signoff (CSM-only) |
| PUT | `/{id}/sow-approval` | advertiser | name+title+timestamp → `status=complete`, **terminal lock** |
| PUT | `/{id}/timeline` | advertiser **+ CSM** | Milestone status / delay (new date) / notes + cascade recompute; audit append (author = session email) |
| POST | `/{id}/tier-override` | **CSM** | Asymmetric override (X→Pro only if auto=Pro); mandatory audit append |

> **Removed:** the spec's `POST /unlock` + re-approval flow — approval is terminal (decision 2026-06-11). No endpoint reopens a completed session's answers.

## 7. Auth (CSM mode)

Frontend gates UI on `isASAdmin` (from identity/Kratos via `/my-details`). Server enforces by **portal-api resolving the caller's `is_as_admin` from mos-iam** (Oathkeeper cannot carry it — `nng-sprinkler-auth` is status-only and `remote_json` can't forward a claim; api-key routes have no Subject). Service-layer gate returns `DataUpdateResult.AccessDenied` → 403 via `HandleDataUpdateResult`/`ForbidWithReason`. Cache the lookup. (Confirm `is_as_admin` vs `ROLE_CSM` — open item.)

**CSM-only (403-gated) set:** `approve`, `tier-override`. **Both client + CSM:** wizard autosave (pre-approval), `book-meeting`, `sow-approval`, and **Timeline status/delay/notes edits** (decision 2026-06-11 — client + CSM both edit the timeline; author always = session email regardless of role, for audit integrity).

## 8. Error handling

- "Book my meeting": clear 200 vs error → frontend sent/retry (spec mermaid).
- CSM actions: 403 on non-admin (enforced regardless of UI).
- Wizard validation: non-blocking; output-screen banner with jump-back links.
- Autosave: last-write-wins on `updatedAt`; guard overlapping in-flight PUTs; surface saving/saved/failed.

## 8a. State, Validation & Persistence Rules (added 2026-06-11 after deep review)

These are the stateful rules the data model implies but the earlier spec/eng-research left underspecified. Locked decisions below.

**Approval is terminal.** `PUT /sow-approval` sets `approverName`/`approverJobTitle`/`approvedAt` + `status=complete`. After that, **wizard answers are immutable** — autosave `PUT /{id}` returns 409, there is no Unlock/re-approval. Post-approval the wizard renders as a **read-only onboarding record**; only the SoW **inline notes** (annotations) and the **Timeline** (delivery tracking) stay editable.

**Validation / completeness (computed, never stored).**

- A step is **done** when every *active* required field is valid; **error** when a visited step has an empty/invalid active required field; **dimmed** = a conditional question whose parent isn't answered yet — excluded from validation until revealed (this is what `.qcard.dim` means).
- Optional steps (External Factors, Objectives) never block and don't count toward required-completeness.
- Output **Approve** is blocked until all required steps are valid; the validation summary lists each offending step and deep-links to its first invalid field (sets that step's rail state to error).

**Tier (computed, not stored except override).** `Estimating…` until budget + campaign types exist (Step 3); then the summary shows the recommended tier + drivers. The authoritative tier is computed at the output screen. `tier.override` (CSM-only, asymmetric: X→Pro only when auto=Pro) is the only stored tier field; every override writes a `changeLog` entry.

**Timeline dates (computed).** `est.start = anchorDate + weekOffset` where `anchorDate = approvedAt ?? today`, rounded to the week-commencing Monday; `est.end = est.start + duration`. **Delay:** editor enters the milestone's new date → UI previews how all subsequent est. dates shift → confirm → store the delta on the milestone (`delayedToDate`); downstream est. dates recompute from the cumulative delta. One `changeLog` entry per edit. Both client + CSM may edit (status/delay/notes).

**Audit author.** Every `changeLog.userEmail` is **server-stamped from the authenticated session** — never client-typed. (Corrects the mockup's free-text "name for notes" field; the spec §4.2 "no free-text, session email" is authoritative.)

**Portal entry (status derived from the session, not a separate flag).** GET-latest → none ⇒ `not_started` (wizard, "Start"); session with no `approvedAt` ⇒ `in_progress` (wizard, "Continue"); `approvedAt` set ⇒ `complete` (read-only record + output tabs, "View your onboarding record"). The session doc's `status` field is the single source of truth — no `onboarding_status` on `app_config` (verified: `app_config` is the per-app model-config doc).

**Stored vs computed (authoritative list).** *Stored:* wizard answers, `tier.override`, `scopeOfWork` (notes/approval/csmApproved), `timeline.milestones` (status/delayedToDate/notes), `changeLog[]`, `status`. *Computed (never stored):* tier recommended/effective + drivers, est. start/end dates, Data Schema provision plan, per-step validation/completeness, the "estimating" tier state.

## 8b. Component & accessibility build notes (from deep review 2026-06-11)

**Render real Vuetify components, not styled divs.** frontend-mos has no `createVuetify` `defaults` block — 8px radius / capitalize / tab underline / field treatment come from global SCSS keyed to `.v-btn`/`.v-field`/`.v-tabs`. A div that merely looks right inherits none of it. All colors via `rgb(var(--v-theme-*))`, never hardcoded hex.

**Component mapping (reuse vs build-fresh):**

- Inline banners → **`v-alert variant="tonal"`** (established precedent: Incrementality Create/List). Do NOT reuse `NotificationBanner` — it's CMS/store-bound, solid-fill.
- Inputs → `InputText`/`InputSelect` (`v-text-field`/`v-select`, 8px, **no focus-halo** — remove the mock's `box-shadow` ring).
- Buttons → `v-btn` (flat/outlined/tonal). Monthly-Annual + CSM tier → `v-btn-toggle`.
- Toggle cards → **build `ToggleCard`** on `v-card`; **checkbox affordance for multi-select** (platforms, campaign types) — the mock's round radio wrongly implies single-select.
- Sliders → **build `ShareAllocator` + `BudgetSlider`** on `v-slider` (the existing `InputSlider` is discrete index-based — can't do % allocation / currency range / total-100 validation).
- Collapsible summary → `v-expansion-panels` (not a JS chevron). Tier chips → `v-chip` with `chips-orange` (Pro) / `chips-blue` (X) tokens.
- **Timeline → bare `v-data-table`** (not the `List` wrapper — its viewport-height + infinite-scroll fight a static 6-row table). Status cell → `v-select` density compact.
- Step rail → **custom** (native `v-stepper` vertical interleaves content — incompatible with the rail|content|summary layout).
- SoW mono document (JetBrains Mono) → intentional, keep; form controls inside stay Inter.

**Accessibility (WCAG 2.1 AA — spec §10):**

- Adopt spec §6 semantic tokens — `--alert-green #427900`, `--alert-orange #BA4E00` — the mock's lighter `#4D840B`/`#D56428` fail AA on tint; warn banners also need a darker/denser background to clear 4.5:1.
- Avoid `text3 #9A9B9D` on white for any text (2.78:1 fail) — use `text2`/darker, especially data-table values.
- Color is never the sole signal: required = asterisk **+** `aria-required` + legend; rail error step gets an icon (not red only); field errors via `aria-invalid`/`aria-describedby`.
- Roles/keyboard: tabs = `role=tablist/tab/tabpanel` + arrow nav; rail = `aria-current="step"`, focusable; multi-select cards/chips = `role=checkbox`/`aria-pressed`; summary collapse = `<button aria-expanded>`; sliders = real `role=slider` with `aria-valuemin/max/now` + arrow keys; focus moves to step heading on step change; loader honors `prefers-reduced-motion`.

**Completeness flag:** the preview visualizes only Steps 2–3 + outputs. The other 6 step UIs (1 Team, 4 Business&Funnel, 5 Data Sources, 6 External Factors, 7 Objectives, 8 Review) are specified in `product-spec-v2.md §4.1` — plan authors build those from the spec, not the mock. All their fields already exist in §5 `wizard{}`.

## 8c. Resolved inputs (investigated 2026-06-11 — not blockers)

**CSM / RBAC (was open #1).** CSM = super-admin = `is_as_admin` — mos-iam code equates "CSM users" with `is_as_admin == true` (`consumer.go:591/599`); there is no separate `"CSM"` role string in the frontend role list.

- **Frontend:** show CSM mode when `useOrganizationStore().isASAdmin` is true; otherwise normal/advertiser mode (a user with `userHasPrivilege("mmm", …)`). Mirror the precedent `ReportsList.vue` (`admin_only ? isASAdmin : true`) / `ApiKeysTab.vue:123`.
- **Backend enforcement:** portal-api resolves the caller via mos-iam **`GetUserByEmail`** (because `x-user-id` is the **email**, not the UUID) and reads `is_as_admin`. ⚠️ The onboarding axios service MUST attach `x-user-id` (+`x-user-name`) on its requests — `mmm.ts` today only adds them for `/incremental-reports`; extend the interceptor to also match `…/onboarding-sessions`. The UI gate is convenience; the server lookup is the real 403 gate. (B5 uses `GetUserByEmail`, not the UUID path.)

**Data Schema columns (was open #2).** AUTHORITATIVE source = Anupam's client-facing doc *"CSV format and file arrangement requirements"* (the doc shared with clients today), cross-confirmed by the live DB (`advertiser_schema_mappings` global doc + `validation_rules`). The Data Schema tab (O6) is the in-app version of this doc, filtered to what the client's answers imply.

Column groups (recommended names — clients may use their own + map on a mapping page; **not all mandatory**):

- **Dimensions:** `full_date, app_id, operating_system, country_code, network_name, campaign_id, campaign_name, campaign_type, publisher_name, channel`.
- **Non-cohort metrics:** `installs, skad_installs, clicks, impressions, total_cost, views, revenue, ad_revenue, new_orders, total_orders, registrations, retentions, daus`.
- **Cohort metrics (flattened):** per cohort day `{0,1,7,30,90,360}` — `views_Xd, revenue_Xd, ad_revenue_Xd, new_orders_Xd, total_orders_Xd, registrations_Xd, retentions_Xd`. **Flattened-cohort format is required** — one row with a column per cohort day, NOT a long `cohort` column.
- **`_incr_` / `_lift_` variants are Incremental-Study ONLY** — exclude from AIM MMM onboarding (base + cohort only).

**File & folder arrangement** (show in the Data Schema tab / template guidance): two supported layouts — (1) hierarchical `YYYY-MM/YYYY-MM-DD/1.csv,2.csv` (filename order only); (2) flat `YYYY-MM/YYYY_MM_DD_n.csv` (filename must encode date+index). Dedicated AIM folder, only the data to share.

**`buildProvisionPlan` (F6)** selects the dimension columns + the metric columns implied by the client's chosen funnel events/sources (e.g. Subscription → installs/registrations/revenue + cohort revenue_Xd; e-commerce → new_orders/total_orders/revenue; ad-spend file → dimensions + `total_cost`). The example dataset (Anupam's xlsx + the Google-sheet link in the doc) is the downloadable CSV-template sample. F6/O6 use this schema verbatim — no placeholder columns.

**Book-meeting email (was open #3).**

- Recipients: env/config `CsmTeamEmail` (comma-list); Phase-1 default `kade@kochava.com, jacob@kochava.com`. Sent via existing `EmailService` (AWS SES).
- Subject: `New AIM onboarding request — {advertiserName} ({tier})`
- Body: `{requesterName} at {advertiserName} ({region}) has completed their AIM onboarding setup and requested a kickoff meeting. Recommended tier: {tier}. Platforms: {platforms}. Review their Scope of Work in K4A: {recordUrl}. We'll reply within 1 business day.`
- From: existing SES Source (`{CompanyName} <{SupportEmail}>`); **Reply-To: the requester's email** so the CSM can respond directly. (B9 template + X2 sent-state copy use this.)

## 9. Testing

- Frontend: Vitest + @vue/test-utils — pure logic (incl. annual→monthly budget, 12-month history threshold, model-change reset), composable autosave, representative step/output components. `npm run test:ci`.
- Backend: NUnit + Moq + FluentAssertions — data-layer scoping, service create/replace, 403 on each privileged op, audit append, cascade math. `dotnet test`.
- Optional Playwright e2e on the vertical slice.

## 10. Plan decomposition (16 workstreams → writing-plans)

Backend: **B1** model+collection+datalayer · **B2** service+controller+DTOs (GET/POST/PUT) · **B3** CSM enforcement (mos-iam lookup→context→403) · **B4** CSM-action endpoints + audit changeLog · **B5** book-meeting SES · **B6** onboarding_status on app_config.
Infra: **I1** Oathkeeper route rule.
Frontend: **F1** tab embed + service + types + status routing (vertical slice) · **F2** pure-logic TS modules · **F3** useAimOnboarding autosave · **F4** wizard steps 1–4 · **F5** wizard steps 5–8 · **F6** SoW tab (+approval, notes, PDF) · **F7** Timeline tab · **F8** Data Schema tab · **F9** CSM-mode UI + book-meeting button.

Maps to eng-tasks IDs (B→T0xx, F→T1xx, I→T201). Each plan = one Asana parent; its TDD steps = subtasks.

## 11. Build order

Vertical slice first: F1 + B1/B2 + I1 + minimal F3 → embed → 1 step → Mongo round-trip → stub output. Locks the persistence contract before breadth. Then F2 logic, wizard (F4/F5), outputs (F6–F8), CSM (B3/B4 + F9), then B5/B6.

## 12. Known-open (content/config, not design)

| Item | Owner |
|------|-------|
| (none currently blocking) | — |

*(Resolved 2026-06-11 — see §8c: CSM signal = `is_as_admin` via mos-iam `GetUserByEmail`; Data Schema columns from `advertiser_schema_mappings` + `validation_rules`; email recipients/copy specified. And `onboarding_status` derived from `session.status`, not `app_config` — all confirmed against the live DB.)*

These do not block planning or the frontend; only the CSM-enforcement wiring (B3) and Data Schema content (F8) depend on them.
