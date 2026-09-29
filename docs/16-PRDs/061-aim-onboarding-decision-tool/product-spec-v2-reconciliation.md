---
id: product-spec-v2-reconciliation
title: "Spec v2.1 Reconciliation: AIM Onboarding Decision Tool"
---

## Spec v2.1 Reconciliation: AIM Onboarding Decision Tool

**PRD:** 061-aim-onboarding-decision-tool
**Author:** Mayank Ukey (lead engineer)
**Date:** 2026-06-11
**Supersedes (where conflicting):** product-spec-v2.md (2026-06-04, merged unchanged in PR #1300)
**Status:** Authoritative for engineering. Read this alongside product-spec-v2.md.

---

## Why this exists

product-spec-v2.md was written 2026-06-04 and merged unchanged on 2026-06-11. Several
engineering decisions were taken in the 10–11 June planning sessions that the merged spec does
not reflect. This document is the delta. **Where this file conflicts with product-spec-v2.md,
this file wins.** Everything in product-spec-v2.md not contradicted here still holds.

---

## 1. Reconciled decisions (override spec-v2)

| # | Topic | product-spec-v2.md says | Canonical (this doc) |
|---|---|---|---|
| R1 | Persistence | localStorage-only; Phase 9 = animation, no backend call except email (§9, §4.4) | **MongoDB hybrid, Phase 1.** localStorage is an in-step buffer only. On wizard step/tab switch and on manual save, the session is written to MongoDB collection `aim_onboarding_sessions`. Backend is in Phase 1 scope. |
| R2 | Timeline (Tab 2) | Full editable: cascade-delay, status edits, notes CRUD, audit log + tier-override logging | **Unchanged — spec-v2 Tab 2 stands in full for Phase 1.** (A June-10 "read-only/deferred" note was reversed.) The audit Change Log persists to MongoDB. |
| R3 | Demo mode | Flow 3 (Demo/Sales) + `BRIGHTFIT_DEMO` seed | **Dropped.** No demo seeding in Phase 1. K4A is the production context. Flow 3 and `BRIGHTFIT_DEMO` are out of scope. |
| R4 | Architecture | "Production: React app with React Router" (§9), React prototype as source (§6) | **Vue 3 / Vuetify**, embedded as a tab inside the existing `MmmInsightsConfiguration` view in `frontend-mos`. No React, no router, no standalone app. The React prototype is reference content only; the navigation chrome is the K4A Vuetify stepper. |
| R5 | Campaign grouping state key | `campaignMerge` / `campaignMergeChoice` (§3.9, §7) | **`campaignGrouping` / `campaignGroupingChoice`.** Verified as the actual key in `mockup-k4a.html`. `campaignMerge` was a documentation-only term and must not appear in code. |

---

## 2. Resolved open questions

These were 🔴 Open in product-spec-v2.md §11. Now resolved:

| OQ | Resolution |
|---|---|
| OQ-1 — Data Schema tab direction | **Option A: downloadable templates.** Engineering authors the Data Schema tab content directly. No API-credential collection in Phase 1. |
| OQ-5 — Multi-region UX | **Dropped. Single-region only** for Phase 1. |
| OQ-11 — "master data schema" | **No external dependency.** There is no separate master schema to wait on; the Data Schema tab is built by engineering (see OQ-1). |
| OQ-12 — `onboarding_status` flag | **Stored as a field on the `app_config` collection** (to be verified against MongoDB). Portal entry reads it to decide wizard vs read-only record. |

D2 (CSM guidance-note content): when content is unspecified, **follow `mockup-k4a.html`** (the CSM tabConfig in the K4A mockup) as the source of truth.

---

## 3. Implementation architecture (Phase 1)

### Repositories

- **Frontend:** `frontend-mos` (Vue 3 / Vuetify). Embedded as a tab in
  `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/MmmInsightsConfiguration.vue`
  (existing route `/advertisertools/mmmconfigurations`). The 8-step wizard uses the Vuetify
  `v-stepper`; the three output tabs (SoW, Timeline, Data Schema) use `v-tabs` + `v-window`.
- **Backend:** `mmm-portal-api` (C# .NET 7) — the MongoDB owner. New CRUD endpoints under
  `/mmm/advertisers/{advertiserId}/onboarding-sessions`, following the existing SavedView
  data-layer / service / controller pattern. Email via the existing AWS SES `EmailService`.

### MongoDB

One new collection: **`aim_onboarding_sessions`** — one document per advertiser onboarding
session, embedding the wizard answers, tier, Scope of Work (with approval block), timeline
milestones, and the audit change log. The Data Schema tab is computed client-side, not stored.
`onboarding_status` lives on the `app_config` collection (OQ-12).

### Known blocker — server-side super-admin enforcement

product-spec-v2.md §4.6 requires server-side enforcement (HTTP 403) of CSM-only actions
(CSM Approval, Unlock to Edit, timeline status edits, tier-override logging). `mmm-portal-api`
currently has **no server-side super-admin concept** — `ICurrentUserContext` carries
organizational scope only, and `is_as_admin` is available only on the frontend (from the
identity service). A mechanism must be chosen before the CSM-action endpoints can satisfy the
spec: verify against the identity service per request, accept a trusted role header injected by
the upstream gateway, or extend `ICurrentUserContext` from a trusted source. A client-supplied
boolean is not acceptable. Owner: backend (Satish).

---

## 4. Unchanged from product-spec-v2.md

Everything not listed above remains as specified in product-spec-v2.md, including: the 8-step
wizard content and field list, the tier recommendation algorithm and budget-conversion formula
(§4.3), the 8 prototype bugs to fix (§9), the milestone list and durations (§4.2 Tab 2), the
"Book my onboarding meeting" email flow with failure/retry (§4.2 Tab 1), PDF export scope
(SoW only, print stylesheet), and the user modes (client / CSM).
