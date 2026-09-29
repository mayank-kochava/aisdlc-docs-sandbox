---
id: plan-f2
title: "F2 — Onboarding Service + Interfaces"
---

## F2 — Onboarding Service + Interfaces

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the TypeScript wire-types (`interfaces/aimOnboarding.ts`) and the
`onboardingService` object (`services/onboarding.ts`) that every other frontend workstream
imports — no UI, no composable, just types + HTTP calls.

**Architecture:** Mirrors the `incrementalityService` pattern exactly — a `basePath`
closure and a plain exported object whose methods call the shared `mmmPortalApi` axios
instance. Types are **PascalCase** to match the C# SavedView POCO they mirror (the
incremental-reports types are camelCase but belong to a different backend stack). F1, F3,
F5–F7 all depend on F2; F2 has no runtime dependencies.

**Tech Stack:** Vue 3.5 / TypeScript 5, Vitest, Axios (`mmmPortalApi`).

---

## Files

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/__tests__/onboarding.spec.ts` |

**Casing note:** Wire DTO fields are **PascalCase** — the server is a C# POCO serialised
with `MongoDB.Driver` defaults (matching the existing `SavedView` collection). String
literal *values* remain exactly as shown in §5 (e.g. `'not_started'`, `'aim_x'`,
`'monthly'`). The F3 composable's flat state keys (`campaignGrouping`,
`campaignGroupingChoice`) are camelCase internally — those become `CampaignGrouping` /
`CampaignGroupingChoice` on the wire.

**Import path for `mmmPortalApi`:** from
`packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/`
the relative import is `../../../../Analytics/services/mmm`.

**No dependencies** (F1, F3, F5–F7 depend on F2 — not the other way around).

---

## Task 1: Wire types (`interfaces/aimOnboarding.ts`)

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts`

- [ ] **Step 1: Write the file**

```ts
// interfaces/aimOnboarding.ts
// PascalCase throughout — mirrors the C# SavedView POCO serialised by MongoDB.Driver.
// String literal VALUES match §5 exactly.

export interface Lead {
  Name: string;
  Email: string;
}

export interface PaidSplit {
  iOS: number;
  Android: number;
  Web: number;
}

export interface FunnelState {
  Items: string[];
  Kpi: string;
  KpiConfirmed: boolean;
  Names: Record<string, string>;
}

export interface WizardState {
  CompanyName: string;
  ProjectLeads: Lead[];
  DataLeads: Lead[];
  AppName: string;
  Platforms: string[];
  PlatformShares: Record<string, number>;
  Modelling: string;
  Region: string;
  RegionOther: string;
  BudgetMonthly: number;
  BudgetAnnual: number;
  BudgetPeriod: 'monthly' | 'annual';
  UsesOffline: boolean;
  OfflineSplitPct: number;
  DigitalMediaTypes: string[];
  HasAttrGaps: string;
  AttrGapCategories: string[];
  CoverageConfidence: string;
  PaidSplit: PaidSplit;
  Ua: boolean;
  UaShare: string;
  Ue: boolean;
  UeShare: string;
  Brand: boolean;
  BrandShare: string;
  CampaignGrouping: boolean;
  CampaignGroupingChoice: string;
  Business: string;
  Funnel: FunnelState;
  WebFunnel: Record<string, unknown>;
  WantsLtv: boolean;
  LtvCohortAvail: string;
  LtvPartialChoice: string;
  Mmp: string;
  MmpCollection: string;
  MmpFileStorage: string;
  AppsflyerCohortAccess: boolean;
  WebAttrSources: string[];
  SpendCollection: string;
  AdSpendSources: string[];
  History: Record<string, unknown>;
  ExternalFactors: string[];
  ExternalFactorsNotes: string;
  ObjectivesGoal: string;
  ObjectivesSuccess: string;
  ObjectivesMarketing: string;
  UaUeRoutingFlag: string;
  BrandRoutingFlag: boolean;
}

export interface TierState {
  Recommended: 'aim_x' | 'aim_pro' | null;
  Override: 'aim_x' | 'aim_pro' | null;
  Effective: 'aim_x' | 'aim_pro' | null;
}

export interface ScopeOfWork {
  SectionNotes: Record<string, string>;
  ApproverName: string;
  ApproverJobTitle: string;
  ApprovedAt: string | null;
  CsmApproved: boolean;
}

export interface Milestone {
  Key: string;
  Name: string;
  Responsible: string;
  WeekOffset: number;
  Duration: string;
  Status: string;
  DelayedToDate: string | null;
  Notes: string;
}

export interface TimelineState {
  AnchorDate: string | null;
  Milestones: Milestone[];
}

export interface ChangeLogEntry {
  At: string;
  UserEmail: string;
  Type: string;
  Field: string;
  OldValue: unknown;
  NewValue: unknown;
}

export type OnboardingStatus = 'not_started' | 'in_progress' | 'complete';

export interface OnboardingSession {
  _id?: string;
  AdvertiserId: string;
  Status: OnboardingStatus;
  CreatedAt: string;
  UpdatedAt: string;
  CreatedByUserId: string;
  UpdatedByUserId: string;
  Wizard: WizardState;
  Tier: TierState;
  ScopeOfWork: ScopeOfWork;
  Timeline: TimelineState;
  ChangeLog: ChangeLogEntry[];
}

/** Payload for PUT /{id}/sow-approval */
export interface SowApprovalPayload {
  ApproverName: string;
  ApproverJobTitle: string;
}

/** Payload for PUT /{id}/timeline */
export interface TimelineUpdatePayload {
  Milestones: Milestone[];
}

/** Payload for POST /{id}/tier-override */
export interface TierOverridePayload {
  Override: 'aim_pro';
}
```

- [ ] **Step 2: Verify the file compiles (no test runner needed)**

```bash
cd /path/to/frontend-mos
npx tsc --noEmit --project packages/advertiser/tsconfig.json \
  2>&1 | grep AimOnboarding
```

Expected: no output (no errors for this file).

- [ ] **Step 3: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts
git commit -m "feat(aim-onboarding): add wire types (F2)"
```

---

## Task 2: Failing service tests (`__tests__/onboarding.spec.ts`)

Write all eight tests **before** the service file exists. They should all fail with
"onboardingService is not defined" (or equivalent module-not-found).

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/__tests__/onboarding.spec.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// __tests__/onboarding.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { mmmPortalApi } from '../../../../../Analytics/services/mmm';

vi.mock('../../../../../Analytics/services/mmm', () => ({
  mmmPortalApi: {
    get: vi.fn(),
    post: vi.fn(),
    put: vi.fn(),
  },
}));

import { onboardingService } from '../onboarding';
import type {
  OnboardingSession,
  SowApprovalPayload,
  TimelineUpdatePayload,
  TierOverridePayload,
} from '../../interfaces/aimOnboarding';

const AID = 'adv-001';
const SID = 'sess-abc';
const BASE = `/mmm/advertisers/${AID}/onboarding-sessions`;

beforeEach(() => {
  vi.clearAllMocks();
});

describe('onboardingService', () => {
  it('getLatest — GET basePath', () => {
    onboardingService.getLatest(AID);
    expect(mmmPortalApi.get).toHaveBeenCalledWith(BASE);
  });

  it('create — POST basePath with payload', () => {
    const payload: Partial<OnboardingSession> = { AdvertiserId: AID };
    onboardingService.create(AID, payload);
    expect(mmmPortalApi.post).toHaveBeenCalledWith(BASE, payload);
  });

  it('update — PUT /{id} with payload', () => {
    const payload: Partial<OnboardingSession> = { AdvertiserId: AID };
    onboardingService.update(AID, SID, payload);
    expect(mmmPortalApi.put).toHaveBeenCalledWith(`${BASE}/${SID}`, payload);
  });

  it('bookMeeting — POST /{id}/book-meeting', () => {
    onboardingService.bookMeeting(AID, SID);
    expect(mmmPortalApi.post).toHaveBeenCalledWith(`${BASE}/${SID}/book-meeting`, undefined);
  });

  it('approve — POST /{id}/approve (CSM-only)', () => {
    onboardingService.approve(AID, SID);
    expect(mmmPortalApi.post).toHaveBeenCalledWith(`${BASE}/${SID}/approve`, undefined);
  });

  it('sowApproval — PUT /{id}/sow-approval with payload', () => {
    const payload: SowApprovalPayload = { ApproverName: 'Jane', ApproverJobTitle: 'VP' };
    onboardingService.sowApproval(AID, SID, payload);
    expect(mmmPortalApi.put).toHaveBeenCalledWith(`${BASE}/${SID}/sow-approval`, payload);
  });

  it('updateTimeline — PUT /{id}/timeline with payload', () => {
    const payload: TimelineUpdatePayload = { Milestones: [] };
    onboardingService.updateTimeline(AID, SID, payload);
    expect(mmmPortalApi.put).toHaveBeenCalledWith(`${BASE}/${SID}/timeline`, payload);
  });

  it('tierOverride — POST /{id}/tier-override with payload (CSM-only)', () => {
    const payload: TierOverridePayload = { Override: 'aim_pro' };
    onboardingService.tierOverride(AID, SID, payload);
    expect(mmmPortalApi.post).toHaveBeenCalledWith(`${BASE}/${SID}/tier-override`, payload);
  });
});
```

- [ ] **Step 2: Run the tests — confirm they all fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/__tests__/onboarding.spec.ts
```

Expected: 8 tests fail. Error should reference missing module `../onboarding`.

- [ ] **Step 3: Commit the failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/__tests__/onboarding.spec.ts
git commit -m "test(aim-onboarding): add failing service tests (F2)"
```

---

## Task 3: Service implementation (`services/onboarding.ts`) + green tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts`

- [ ] **Step 1: Write the service**

```ts
// services/onboarding.ts
// Mirror pattern: incrementalityService in packages/advertiser/src/services/incrementality.ts
import { mmmPortalApi } from '../../../../Analytics/services/mmm';
import type {
  OnboardingSession,
  SowApprovalPayload,
  TimelineUpdatePayload,
  TierOverridePayload,
} from '../interfaces/aimOnboarding';

const basePath = (advertiserId: string) =>
  `/mmm/advertisers/${advertiserId}/onboarding-sessions`;

export const onboardingService = {
  /** GET /mmm/advertisers/{advertiserId}/onboarding-sessions — drives wizard vs read-only status */
  getLatest(advertiserId: string) {
    return mmmPortalApi.get<OnboardingSession>(basePath(advertiserId));
  },

  /** POST /mmm/advertisers/{advertiserId}/onboarding-sessions — create session */
  create(advertiserId: string, payload: Partial<OnboardingSession>) {
    return mmmPortalApi.post<OnboardingSession>(basePath(advertiserId), payload);
  },

  /** PUT /mmm/advertisers/{advertiserId}/onboarding-sessions/{id} — full-doc autosave; returns 409 once approvedAt is set */
  update(advertiserId: string, id: string, payload: Partial<OnboardingSession>) {
    return mmmPortalApi.put<OnboardingSession>(`${basePath(advertiserId)}/${id}`, payload);
  },

  /** POST /mmm/advertisers/{advertiserId}/onboarding-sessions/{id}/book-meeting — triggers SES to CSM team */
  bookMeeting(advertiserId: string, id: string) {
    return mmmPortalApi.post<void>(`${basePath(advertiserId)}/${id}/book-meeting`, undefined);
  },

  /** POST /mmm/advertisers/{advertiserId}/onboarding-sessions/{id}/approve — CSM-only; 403 for non-admin */
  approve(advertiserId: string, id: string) {
    return mmmPortalApi.post<OnboardingSession>(`${basePath(advertiserId)}/${id}/approve`, undefined);
  },

  /** PUT /mmm/advertisers/{advertiserId}/onboarding-sessions/{id}/sow-approval — name+title+timestamp; sets status=complete (terminal) */
  sowApproval(advertiserId: string, id: string, payload: SowApprovalPayload) {
    return mmmPortalApi.put<OnboardingSession>(`${basePath(advertiserId)}/${id}/sow-approval`, payload);
  },

  /** PUT /mmm/advertisers/{advertiserId}/onboarding-sessions/{id}/timeline — status/delay/notes + cascade; author stamped server-side */
  updateTimeline(advertiserId: string, id: string, payload: TimelineUpdatePayload) {
    return mmmPortalApi.put<OnboardingSession>(`${basePath(advertiserId)}/${id}/timeline`, payload);
  },

  /** POST /mmm/advertisers/{advertiserId}/onboarding-sessions/{id}/tier-override — CSM-only, asymmetric (aim_pro only); 403 for non-admin */
  tierOverride(advertiserId: string, id: string, payload: TierOverridePayload) {
    return mmmPortalApi.post<OnboardingSession>(`${basePath(advertiserId)}/${id}/tier-override`, payload);
  },
};
```

- [ ] **Step 2: Run the tests — confirm all 8 pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/__tests__/onboarding.spec.ts
```

Expected output:

```text
 ✓ onboardingService > getLatest — GET basePath
 ✓ onboardingService > create — POST basePath with payload
 ✓ onboardingService > update — PUT /{id} with payload
 ✓ onboardingService > bookMeeting — POST /{id}/book-meeting
 ✓ onboardingService > approve — POST /{id}/approve (CSM-only)
 ✓ onboardingService > sowApproval — PUT /{id}/sow-approval with payload
 ✓ onboardingService > updateTimeline — PUT /{id}/timeline with payload
 ✓ onboardingService > tierOverride — POST /{id}/tier-override with payload (CSM-only)

Test Files  1 passed (1)
Tests       8 passed (8)
```

- [ ] **Step 3: Verify no TypeScript errors**

```bash
npx tsc --noEmit --project packages/advertiser/tsconfig.json \
  2>&1 | grep -i "aim\|onboarding"
```

Expected: no output.

- [ ] **Step 4: Commit the implementation**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts
git commit -m "feat(aim-onboarding): add onboardingService (F2)"
```

---

## Dependencies

None — F2 has no runtime dependencies within the onboarding workstream.

**Downstream dependents** (these import from F2):

| Plan | Imports |
|------|---------|
| F1 — Tab embed + routing | `onboardingService.getLatest` |
| F3 — useAimOnboarding composable | `onboardingService.*`, `OnboardingSession`, `WizardState` |
| F5 — logic/recommendTier | `TierState`, `WizardState` |
| F6 — logic/buildProvisionPlan | `WizardState` |
| F7 — logic/validation + milestones | `WizardState`, `Milestone`, `TimelineState` |

---

## Self-review

**Spec coverage:**

- §5 Mongo doc shape → all fields typed in `WizardState` / embedded interfaces; PascalCase throughout; `CampaignGrouping`/`CampaignGroupingChoice` present; `BudgetPeriod` union; `OnboardingStatus` union; `TierState` with `Recommended`/`Override`/`Effective`; `ScopeOfWork` with `SectionNotes` map; `Milestone` with all 8 fields; `ChangeLogEntry` with all 6 fields.
- §6 API contract → 8 methods, 1:1 with the table: `getLatest`/`create`/`update`/`bookMeeting`/`approve`/`sowApproval`/`updateTimeline`/`tierOverride`. Verbs match exactly (GET/POST/PUT). Paths include trailing sub-paths verbatim.
- No unlock endpoint (removed per 2026-06-11 decision).
- Test coverage: one test per method, asserting verb + exact URL + payload.

**Placeholder scan:** No TBD / TODO / "fill in" / "similar to" language. All code blocks are complete.

**Type consistency:** `OnboardingSession`, `SowApprovalPayload`, `TimelineUpdatePayload`, `TierOverridePayload` defined in Task 1 and imported identically in Tasks 2 and 3. Method names match between spec and implementation (`onboardingService.getLatest`, not `getLatestSession`).
