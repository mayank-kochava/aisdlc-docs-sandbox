---
id: plan-f1
title: "F1 — Tab Embed + Status Routing"
---

## F1 — Tab Embed + Status Routing

**Date:** 2026-06-11
**Status:** Ready for implementation
**Workstream:** Frontend foundation
**Asana project:** `1215287604489661` · section `1215287743348715`

---

## Goal

Register `AimOnboardingTab` as the **3rd tab** in `MmmInsightsConfiguration.vue` (after Marketplaces and Validate Onboarding Data). The wrapper fetches the latest onboarding session on mount, then routes by `session.status`:

- `not_started` (no session returned) → wizard entry with **Start** CTA
- `in_progress` (session exists, no `approvedAt`) → wizard entry with **Continue** CTA
- `complete` (`approvedAt` set) → read-only onboarding record + output tabs shell

Status is derived from `session.Status` on the session document. There is no separate `onboarding_status` field.

CSM mode is gated on `useOrganizationStore().isASAdmin` (same pattern as `useAdvertiserMenu`).

---

## Files

### New files (create)

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/
  AimOnboardingTab.vue
  __tests__/
    AimOnboardingTab.spec.ts
```

### Modified files

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/MmmInsightsConfiguration.vue
packages/app/src/locales/en.json
```

### Consumed from F2 (do not recreate)

F2 owns these files. F1 imports them — treat them as the seam boundary:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/
  services/onboarding.ts          ← getLatestSession(advertiserId): Promise<OnboardingSession | null>
  interfaces/aimOnboarding.ts     ← OnboardingSession, SessionStatus
```

Until F2 lands, stub the service inside the test file using `vi.mock` (see Task 1). The production import path must match exactly so the stub is transparent.

---

## Dependencies

- **F2** (service + interfaces) — F1 imports `getLatestSession` and `OnboardingSession` / `SessionStatus` from F2's files. Do **not** re-implement the fetch or the type. If F2 is not merged yet, the stubs in the test cover the gap; the `.vue` file uses the real import path.
- **No backend required for F1** — the `onMount` fetch is wired but can 404 gracefully during development (status falls back to `not_started`).

---

## Interface contract (from design-consolidated §5 + §8a)

F1 reads exactly this shape from F2. Do not add fields.

```ts
// From F2: interfaces/aimOnboarding.ts
export type SessionStatus = 'not_started' | 'in_progress' | 'complete';

export interface OnboardingSession {
  _id: string;
  advertiserId: string;
  status: SessionStatus;
  approvedAt: string | null;
  // ... other wizard fields omitted — F1 does not read them
}
```

Status routing logic (§8a — authoritative):

| GET /onboarding-sessions result | `approvedAt` | Derived status | CTA |
|---|---|---|---|
| null (no session) | — | `not_started` | Start |
| session | null | `in_progress` | Continue |
| session | non-null | `complete` | View record |

---

## Tasks

### Task 1 — Write failing component test

**File:** `AimOnboardingTab/__tests__/AimOnboardingTab.spec.ts`

Write the full spec before creating the component. Run `npm run test:unit` — it must fail with "Cannot find module" or similar.

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { mount, flushPromises } from '@vue/test-utils';

// ---------------------------------------------------------------------------
// Hoisted mocks (must be before any vi.mock calls)
// ---------------------------------------------------------------------------
const { mockGetLatestSession, mockOrganizationStore } = vi.hoisted(() => ({
  mockGetLatestSession: vi.fn(),
  mockOrganizationStore: { isASAdmin: false, accountEntityId: 'adv-123' },
}));

vi.mock('../services/onboarding', () => ({
  getLatestSession: mockGetLatestSession,
}));

vi.mock('@mos/core', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@mos/core')>();
  return {
    ...actual,
    useOrganizationStore: () => mockOrganizationStore,
  };
});

import AimOnboardingTab from '../AimOnboardingTab.vue';

// ---------------------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------------------
const mountTab = () =>
  mount(AimOnboardingTab, {
    // global plugins (vuetify, pinia, i18n) come from tests.config.ts setup file
  });

describe('AimOnboardingTab', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mockOrganizationStore.isASAdmin = false;
    mockOrganizationStore.accountEntityId = 'adv-123';
  });

  // -------------------------------------------------------------------------
  // Loading state
  // -------------------------------------------------------------------------
  describe('loading state', () => {
    it('shows a loader while the session fetch is in flight', async () => {
      // Never resolves during this test
      mockGetLatestSession.mockReturnValue(new Promise(() => {}));

      const wrapper = mountTab();
      // Loader present before fetch resolves
      expect(wrapper.find('[data-testid="aim-onboarding-loading"]').exists()).toBe(true);
    });
  });

  // -------------------------------------------------------------------------
  // not_started: no session returned
  // -------------------------------------------------------------------------
  describe('when session is null (not_started)', () => {
    it('renders the Start CTA and hides the Continue and View-record views', async () => {
      mockGetLatestSession.mockResolvedValue(null);

      const wrapper = mountTab();
      await flushPromises();

      expect(wrapper.find('[data-testid="aim-onboarding-start"]').exists()).toBe(true);
      expect(wrapper.find('[data-testid="aim-onboarding-continue"]').exists()).toBe(false);
      expect(wrapper.find('[data-testid="aim-onboarding-record"]').exists()).toBe(false);
    });

    it('calls getLatestSession with the active advertiser id', async () => {
      mockGetLatestSession.mockResolvedValue(null);

      mountTab();
      await flushPromises();

      expect(mockGetLatestSession).toHaveBeenCalledWith('adv-123');
    });
  });

  // -------------------------------------------------------------------------
  // in_progress: session exists, approvedAt is null
  // -------------------------------------------------------------------------
  describe('when session is in_progress', () => {
    it('renders the Continue CTA and hides Start and View-record views', async () => {
      mockGetLatestSession.mockResolvedValue({
        _id: 'sess-1',
        advertiserId: 'adv-123',
        status: 'in_progress',
        approvedAt: null,
      });

      const wrapper = mountTab();
      await flushPromises();

      expect(wrapper.find('[data-testid="aim-onboarding-continue"]').exists()).toBe(true);
      expect(wrapper.find('[data-testid="aim-onboarding-start"]').exists()).toBe(false);
      expect(wrapper.find('[data-testid="aim-onboarding-record"]').exists()).toBe(false);
    });
  });

  // -------------------------------------------------------------------------
  // complete: session.approvedAt is set
  // -------------------------------------------------------------------------
  describe('when session is complete', () => {
    it('renders the read-only record view and hides wizard CTAs', async () => {
      mockGetLatestSession.mockResolvedValue({
        _id: 'sess-2',
        advertiserId: 'adv-123',
        status: 'complete',
        approvedAt: '2026-06-01T00:00:00Z',
      });

      const wrapper = mountTab();
      await flushPromises();

      expect(wrapper.find('[data-testid="aim-onboarding-record"]').exists()).toBe(true);
      expect(wrapper.find('[data-testid="aim-onboarding-start"]').exists()).toBe(false);
      expect(wrapper.find('[data-testid="aim-onboarding-continue"]').exists()).toBe(false);
    });
  });

  // -------------------------------------------------------------------------
  // Fetch error: falls back to not_started, no unhandled rejection
  // -------------------------------------------------------------------------
  describe('when the fetch rejects', () => {
    it('falls back to not_started view without throwing', async () => {
      mockGetLatestSession.mockRejectedValue(new Error('Network error'));

      const wrapper = mountTab();
      await flushPromises();

      // Graceful degradation: treat missing session as not_started
      expect(wrapper.find('[data-testid="aim-onboarding-start"]').exists()).toBe(true);
    });
  });

  // -------------------------------------------------------------------------
  // CSM mode: isASAdmin flag is passed to child slots
  // -------------------------------------------------------------------------
  describe('CSM mode', () => {
    it('exposes isCSM=true when isASAdmin is true', async () => {
      mockOrganizationStore.isASAdmin = true;
      mockGetLatestSession.mockResolvedValue(null);

      const wrapper = mountTab();
      await flushPromises();

      // The wrapper root carries data-is-csm so child stubs can assert it
      expect(wrapper.find('[data-testid="aim-onboarding-tab"]').attributes('data-is-csm')).toBe('true');
    });

    it('exposes isCSM=false for non-admin users', async () => {
      mockOrganizationStore.isASAdmin = false;
      mockGetLatestSession.mockResolvedValue(null);

      const wrapper = mountTab();
      await flushPromises();

      expect(wrapper.find('[data-testid="aim-onboarding-tab"]').attributes('data-is-csm')).toBe('false');
    });
  });
});
```

**Verify:** `cd packages/advertiser && npm run test:unit -- --reporter=verbose AimOnboardingTab.spec.ts` exits non-zero.

---

### Task 2 — Create `AimOnboardingTab.vue`

**File:** `AimOnboardingTab/AimOnboardingTab.vue`

Mirror `ValidateOnboardingTab.vue` for structure (wrapper div, scoped style). Status routing uses a single `computed` on `session.value?.status` cross-referenced with `session.value?.approvedAt` (§8a rule).

```vue
<template>
  <div
    class="aim-onboarding-tab-content ml-4"
    data-testid="aim-onboarding-tab"
    :data-is-csm="String(isCSM)"
  >
    <!-- Loading -->
    <div v-if="isLoading" class="d-flex justify-center align-center py-12" data-testid="aim-onboarding-loading">
      <v-progress-circular indeterminate color="primary" />
    </div>

    <!-- not_started: no session -->
    <div v-else-if="routedStatus === 'not_started'" data-testid="aim-onboarding-start">
      <p class="body-2 mb-6" style="max-width: 520px; color: rgb(var(--v-theme-grey))">
        {{ $t('mmm_config.aim_onboarding.description_start') }}
      </p>
      <v-btn color="primary" rounded>
        {{ $t('mmm_config.aim_onboarding.cta_start') }}
      </v-btn>
    </div>

    <!-- in_progress: session exists, not approved -->
    <div v-else-if="routedStatus === 'in_progress'" data-testid="aim-onboarding-continue">
      <p class="body-2 mb-6" style="max-width: 520px; color: rgb(var(--v-theme-grey))">
        {{ $t('mmm_config.aim_onboarding.description_continue') }}
      </p>
      <v-btn color="primary" rounded>
        {{ $t('mmm_config.aim_onboarding.cta_continue') }}
      </v-btn>
    </div>

    <!-- complete: approvedAt set -->
    <div v-else-if="routedStatus === 'complete'" data-testid="aim-onboarding-record">
      <p class="body-2 mb-6" style="max-width: 520px; color: rgb(var(--v-theme-grey))">
        {{ $t('mmm_config.aim_onboarding.description_record') }}
      </p>
      <v-btn variant="outlined" color="primary" rounded>
        {{ $t('mmm_config.aim_onboarding.cta_view_record') }}
      </v-btn>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { useOrganizationStore } from '@mos/core';
import { getLatestSession } from './services/onboarding';
import type { OnboardingSession, SessionStatus } from './interfaces/aimOnboarding';

// ---------------------------------------------------------------------------
// Store
// ---------------------------------------------------------------------------
const organizationStore = useOrganizationStore();

const isCSM = computed(() => organizationStore.isASAdmin);

// ---------------------------------------------------------------------------
// Session fetch + status routing (design-consolidated §8a)
//
// Routing rules:
//   null session          → 'not_started'  (CTA: Start)
//   session, no approvedAt → 'in_progress' (CTA: Continue)
//   session, approvedAt set → 'complete'   (read-only record)
// ---------------------------------------------------------------------------
const isLoading = ref(true);
const session = ref<OnboardingSession | null>(null);

const routedStatus = computed<SessionStatus>(() => {
  if (!session.value) return 'not_started';
  if (session.value.approvedAt) return 'complete';
  return 'in_progress';
});

onMounted(async () => {
  try {
    session.value = await getLatestSession(organizationStore.accountEntityId);
  } catch {
    // Network/API error: degrade gracefully to not_started
    session.value = null;
  } finally {
    isLoading.value = false;
  }
});
</script>

<style scoped lang="scss">
.aim-onboarding-tab-content {
  min-height: 200px;
}
</style>
```

**Verify:** `npm run test:unit -- AimOnboardingTab.spec.ts` passes all 8 tests.

---

### Task 3 — Register tab in `MmmInsightsConfiguration.vue`

**File:** `MmmInsightsConfiguration/MmmInsightsConfiguration.vue`

Add the 3rd tab value `aimOnboarding` to the existing `v-tabs` / `v-window`, import the new component. Minimal diff — touch only what is shown below.

**Template diff (add two blocks):**

```vue
<!-- Inside .config-header > v-tabs, after the validateOnboarding v-tab -->
<v-tab value="aimOnboarding">{{ $t("mmm_config.tabs.aim_onboarding") }}</v-tab>
```

```vue
<!-- Inside v-window, after the validateOnboarding v-window-item -->
<v-window-item value="aimOnboarding">
  <aim-onboarding-tab />
</v-window-item>
```

**Script diff (add import after existing imports):**

```ts
import AimOnboardingTab from "./AimOnboardingTab/AimOnboardingTab.vue";
```

The default active tab stays `marketplaces` — do not change `const activeTab = ref("marketplaces")`.

After editing, the full `v-tabs` block looks like:

```vue
<v-tabs v-model="activeTab" class="config-tabs" bg-color="transparent">
  <!-- <v-tab value="metrics">{{ $t("mmm_config.tabs.metrics") }}</v-tab> -->
  <v-tab value="marketplaces">{{ $t("mmm_config.tabs.marketplaces") }}</v-tab>
  <v-tab value="validateOnboarding">{{ $t("mmm_config.tabs.validate_onboarding") }}</v-tab>
  <v-tab value="aimOnboarding">{{ $t("mmm_config.tabs.aim_onboarding") }}</v-tab>
</v-tabs>
```

And the `v-window` block:

```vue
<v-window v-model="activeTab">
  <!-- <v-window-item value="metrics"><metrics-tab /></v-window-item> -->
  <v-window-item value="marketplaces">
    <marketplaces-tab :can-create="canCreate" :can-update="canUpdate" />
  </v-window-item>
  <v-window-item value="validateOnboarding">
    <validate-onboarding-tab :can-create="canCreate" />
  </v-window-item>
  <v-window-item value="aimOnboarding">
    <aim-onboarding-tab />
  </v-window-item>
</v-window>
```

**Verify:** `npm run lint` passes. No type errors. Hot-reload in dev shows a third tab "AIM Onboarding" that renders the loading spinner and then the Start CTA (when no session exists).

---

### Task 4 — Add i18n keys

**File:** `packages/app/src/locales/en.json`

Locate the `mmm_config` → `tabs` object (currently has `metrics`, `marketplaces`, `validate_onboarding`). Add `aim_onboarding`:

```json
"tabs": {
  "metrics": "Metrics",
  "marketplaces": "Marketplaces",
  "validate_onboarding": "Validate Onboarding Data",
  "aim_onboarding": "AIM Onboarding"
}
```

Also add the new `aim_onboarding` namespace under `mmm_config`:

```json
"aim_onboarding": {
  "description_start": "Complete the onboarding questionnaire to generate your Scope of Work, delivery timeline, and data schema.",
  "description_continue": "You have an onboarding session in progress. Pick up where you left off.",
  "description_record": "Your onboarding is complete. Review your Scope of Work, timeline, and data schema below.",
  "cta_start": "Start onboarding",
  "cta_continue": "Continue onboarding",
  "cta_view_record": "View your onboarding record"
}
```

**Verify:** `npm run lint` passes. The tab label renders correctly in the browser.

---

### Task 5 — Run full test suite and commit

```bash
cd /path/to/frontend-mos/packages/advertiser
npm run test:unit
npm run lint
```

Both must pass clean. Then commit:

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/AimOnboardingTab.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/AimOnboardingTab.spec.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/MmmInsightsConfiguration.vue \
  packages/app/src/locales/en.json
git commit -m "feat(aim-onboarding): register AimOnboardingTab as 3rd tab with status routing"
```

---

## Notes for implementation

- `accountEntityId` on `useOrganizationStore()` is the advertiser ID string — confirmed from `tests.config.ts` mock shape (`accountEntityId: 123`) and the `useValidateOnboarding` spec which mocks `getPopulatedEndpoint` replacing `{advertiser_id}` with `"123"`.
- The `data-testid` attributes on the wrapper div (`data-is-csm`) enable the CSM assertions without querying internal structure.
- F2's `getLatestSession` returns `OnboardingSession | null`. Returning `null` is the signal for "no session" (not a 404 exception) — implement the fetch accordingly in F2.
- Do **not** add `canCreate` or `canUpdate` props to `AimOnboardingTab`. Those props gate CSV-upload / metric editing in existing tabs. AIM onboarding access is governed by the `mmm → view` privilege (already checked at the page level by `MmmInsightsConfiguration`) and by `isASAdmin` for CSM-only actions (handled inside the tab itself).
- All colors via `rgb(var(--v-theme-*))` — no hardcoded hex (§8b).
- The loading state uses `v-progress-circular` (Vuetify native, inherits theme) — not a `ThreeBarLoader` or custom spinner.
- `routedStatus` is a computed, not a stored field. `session.value?.approvedAt` is the single truth (§8a).
