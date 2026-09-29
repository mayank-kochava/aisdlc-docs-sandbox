---
id: plan-x1
title: "X1 — CSM Mode UI"
---

## X1 — CSM Mode UI

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the CSM-only controls to the output header — gated on
`useOrganizationStore().isASAdmin`. Three additions to the output screen:

1. **Admin bar** (`CsmAdminBar.vue`) — a pill-shaped grouped control in the `.ohead`
   region containing: `Admin` label, AIM X / Pro tier-override toggle
   (asymmetric enable rules, 403-aware), and a CSM Approve button.
2. **Admin info banner** — a `v-alert` below the output header with the copy
   "Admin mode — changes recorded in the change log." Visible in admin bar only.
3. **isASAdmin gate** — client mode shows none of the above; no `is_as_admin`
   or `403` language surfaces in the UI.

**Decision — TierOverridePayload direction:** The asymmetric toggle allows
Pro→X downgrade AND X→Pro upgrade (when auto=Pro). Both directions must reach
the server. The plan widens F2's `TierOverridePayload.Override` type from
`'aim_pro'` to `'aim_x' | 'aim_pro'` (see Task 1 Dependencies note). Send the
selected tier: `{ Override: selectedTier }`.

**No Unlock:** approval is terminal (design-consolidated §8a, 2026-06-11
decision). The mockup's Unlock button is not ported.

**Tech stack:** Vue 3.5, TypeScript 5, Vuetify 3.12, Pinia, Vitest +
@vue/test-utils.

---

## Files

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/CsmAdminBar.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/CsmAdminBar.spec.ts` |

**F2 type amendment** (inline in Task 1):
`packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts`
— widen `TierOverridePayload.Override` from `'aim_pro'` to `'aim_x' | 'aim_pro'`.

**Not touched here:** O7's output shell (`OutputShell.vue` or equivalent) —
X1 owns the component; O7 drops `<CsmAdminBar>` into `.ohead`. That seam is
spec'd in the Dependencies section below.

---

## Dependencies

| Plan | Why |
|------|-----|
| **O7** — Output shell | Owns the `.ohead` row; renders `<CsmAdminBar>` there. The component self-gates on `isASAdmin`, so O7 always renders it — nothing shows in client mode. |
| **F2** — Service + interfaces | `onboardingService.approve` + `onboardingService.tierOverride`; `TierOverridePayload` type (widened in Task 1). |
| **B6** — CSM action endpoints | `POST /approve` + `POST /tier-override` server-side 403 gate; all `changeLog` entries are server-stamped (B6 owns that). |

`isASAdmin` is already a reactive `ref` exported by `useOrganizationStore()`
(`packages/core/src/stores/organization.ts` line 52 / line 296). No new store
work required.

---

## Task 1: Widen `TierOverridePayload` + failing tests

### 1a — Amend `TierOverridePayload` in `interfaces/aimOnboarding.ts`

F2 originally typed `Override: 'aim_pro'` (one direction only). Both downgrade
(Pro→X) and upgrade (X→Pro when auto=Pro) must be sent to the server.

- [ ] **Step 1: Read the interfaces file** (required before edit)

```bash
cat packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts \
  | grep -A3 "TierOverridePayload"
```

- [ ] **Step 2: Widen the type**

Replace:

```ts
/** Payload for POST /{id}/tier-override */
export interface TierOverridePayload {
  Override: 'aim_pro';
}
```

With:

```ts
/** Payload for POST /{id}/tier-override */
export interface TierOverridePayload {
  Override: 'aim_x' | 'aim_pro';
}
```

- [ ] **Step 3: Verify no type regressions**

```bash
cd /path/to/frontend-mos
npx tsc --noEmit --project packages/advertiser/tsconfig.json \
  2>&1 | grep -i "aim\|onboarding"
```

Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts
git commit -m "feat(aim-onboarding): widen TierOverridePayload to include aim_x direction (X1)"
```

### 1b — Failing component tests

Write all tests **before** `CsmAdminBar.vue` exists. All tests must fail
(component import error) when first run.

- [ ] **Step 1: Write the failing tests**

```ts
// components/__tests__/CsmAdminBar.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { mount } from '@vue/test-utils';
import { createTestingPinia } from '@pinia/testing';
import { createVuetify } from 'vuetify';
import 'vuetify/styles';
import CsmAdminBar from '../CsmAdminBar.vue';

vi.mock('../../services/onboarding', () => ({
  onboardingService: {
    approve: vi.fn().mockResolvedValue({}),
    tierOverride: vi.fn().mockResolvedValue({}),
  },
}));

import { onboardingService } from '../../services/onboarding';
import { useOrganizationStore } from '@mos/core';

const vuetify = createVuetify();

const PROPS = {
  advertiserId: 'adv-001',
  sessionId: 'sess-abc',
  autoRecommended: 'aim_x' as 'aim_x' | 'aim_pro',
  effectiveTier: 'aim_x' as 'aim_x' | 'aim_pro',
  isApproved: false,
};

function mountBar(
  isASAdmin: boolean,
  props: Partial<typeof PROPS> = {},
) {
  return mount(CsmAdminBar, {
    props: { ...PROPS, ...props },
    global: {
      plugins: [
        vuetify,
        createTestingPinia({
          initialState: {
            organization: { isASAdmin },
          },
          stubActions: false,
        }),
      ],
    },
  });
}

describe('CsmAdminBar', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  // --- Gating ---

  it('renders nothing when isASAdmin is false', () => {
    const wrapper = mountBar(false);
    expect(wrapper.html()).toBe('');
  });

  it('renders the admin bar when isASAdmin is true', () => {
    const wrapper = mountBar(true);
    expect(wrapper.find('[data-testid="csm-admin-bar"]').exists()).toBe(true);
  });

  it('renders the admin info banner when isASAdmin is true', () => {
    const wrapper = mountBar(true);
    const banner = wrapper.find('[data-testid="csm-info-banner"]');
    expect(banner.exists()).toBe(true);
    expect(banner.text()).toContain('Admin mode');
    expect(banner.text()).toContain('change log');
    // Must not expose internal jargon
    expect(banner.text()).not.toContain('is_as_admin');
    expect(banner.text()).not.toContain('403');
  });

  // --- Asymmetric toggle enable rules ---

  it('Pro v-btn is disabled when auto-recommendation is aim_x', () => {
    const wrapper = mountBar(true, { autoRecommended: 'aim_x' });
    const proBtnWrapper = wrapper.find('[data-testid="tier-btn-pro"]');
    // v-btn with disabled renders as disabled attribute
    expect(proBtnWrapper.attributes('disabled')).toBeDefined();
  });

  it('Pro v-btn is enabled when auto-recommendation is aim_pro', () => {
    const wrapper = mountBar(true, {
      autoRecommended: 'aim_pro',
      effectiveTier: 'aim_pro',
    });
    const proBtnWrapper = wrapper.find('[data-testid="tier-btn-pro"]');
    expect(proBtnWrapper.attributes('disabled')).toBeUndefined();
  });

  it('AIM X v-btn is always enabled when isASAdmin is true', () => {
    const wrapper = mountBar(true, { autoRecommended: 'aim_pro', effectiveTier: 'aim_pro' });
    const xBtnWrapper = wrapper.find('[data-testid="tier-btn-x"]');
    expect(xBtnWrapper.attributes('disabled')).toBeUndefined();
  });

  it('selecting Pro fires tierOverride with Override aim_pro when auto=Pro', async () => {
    const wrapper = mountBar(true, {
      autoRecommended: 'aim_pro',
      effectiveTier: 'aim_x',
    });
    await wrapper.find('[data-testid="tier-btn-pro"]').trigger('click');
    expect(onboardingService.tierOverride).toHaveBeenCalledWith(
      PROPS.advertiserId,
      PROPS.sessionId,
      { Override: 'aim_pro' },
    );
  });

  it('selecting AIM X fires tierOverride with Override aim_x', async () => {
    const wrapper = mountBar(true, {
      autoRecommended: 'aim_pro',
      effectiveTier: 'aim_pro',
    });
    await wrapper.find('[data-testid="tier-btn-x"]').trigger('click');
    expect(onboardingService.tierOverride).toHaveBeenCalledWith(
      PROPS.advertiserId,
      PROPS.sessionId,
      { Override: 'aim_x' },
    );
  });

  // --- Approve button ---

  it('Approve button calls onboardingService.approve and emits approved', async () => {
    const wrapper = mountBar(true);
    await wrapper.find('[data-testid="csm-approve-btn"]').trigger('click');
    expect(onboardingService.approve).toHaveBeenCalledWith(
      PROPS.advertiserId,
      PROPS.sessionId,
    );
    await wrapper.vm.$nextTick();
    expect(wrapper.emitted('approved')).toBeTruthy();
  });

  it('Approve button is disabled once isApproved is true', () => {
    const wrapper = mountBar(true, { isApproved: true });
    const btn = wrapper.find('[data-testid="csm-approve-btn"]');
    expect(btn.attributes('disabled')).toBeDefined();
  });

  // --- 403 handling ---

  it('403 on tierOverride shows human-readable error without 403 jargon', async () => {
    vi.mocked(onboardingService.tierOverride).mockRejectedValueOnce({
      response: { status: 403 },
    });
    const wrapper = mountBar(true, { autoRecommended: 'aim_pro', effectiveTier: 'aim_x' });
    await wrapper.find('[data-testid="tier-btn-pro"]').trigger('click');
    await wrapper.vm.$nextTick();
    const errorEl = wrapper.find('[data-testid="csm-error"]');
    expect(errorEl.exists()).toBe(true);
    expect(errorEl.text()).not.toContain('403');
    expect(errorEl.text()).not.toContain('is_as_admin');
  });

  it('403 on tierOverride reverts the toggle to prior value', async () => {
    vi.mocked(onboardingService.tierOverride).mockRejectedValueOnce({
      response: { status: 403 },
    });
    const wrapper = mountBar(true, {
      autoRecommended: 'aim_pro',
      effectiveTier: 'aim_x',
    });
    await wrapper.find('[data-testid="tier-btn-pro"]').trigger('click');
    await wrapper.vm.$nextTick();
    // The toggle should revert to aim_x (the effectiveTier prop value)
    const toggle = wrapper.find('[data-testid="tier-toggle"]');
    expect(toggle.attributes('modelvalue') ?? toggle.attributes('model-value')).toBe('aim_x');
  });

  it('403 on approve shows human-readable error without 403 jargon', async () => {
    vi.mocked(onboardingService.approve).mockRejectedValueOnce({
      response: { status: 403 },
    });
    const wrapper = mountBar(true);
    await wrapper.find('[data-testid="csm-approve-btn"]').trigger('click');
    await wrapper.vm.$nextTick();
    const errorEl = wrapper.find('[data-testid="csm-error"]');
    expect(errorEl.exists()).toBe(true);
    expect(errorEl.text()).not.toContain('403');
    expect(errorEl.text()).not.toContain('is_as_admin');
  });
});
```

- [ ] **Step 2: Run the tests — confirm they all fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/CsmAdminBar.spec.ts
```

Expected: all tests fail with component import error.

- [ ] **Step 3: Commit the failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/CsmAdminBar.spec.ts
git commit -m "test(aim-onboarding): add failing CsmAdminBar tests (X1)"
```

---

## Task 2: `CsmAdminBar.vue` implementation

- [ ] **Step 1: Write the component**

```vue
<!-- components/CsmAdminBar.vue -->
<template>
  <template v-if="organizationStore.isASAdmin">
    <!-- Admin info banner -->
    <v-alert
      data-testid="csm-info-banner"
      variant="tonal"
      color="info"
      density="compact"
      icon="mdi-shield-account-outline"
      class="mb-4"
      style="max-width: 820px;"
    >
      Admin mode — tier changes and timeline edits are recorded in the change log.
    </v-alert>

    <!-- Admin bar: label + tier toggle + approve -->
    <div
      data-testid="csm-admin-bar"
      class="d-flex align-center ga-2 px-3 py-1 rounded-pill"
      style="
        background: rgb(var(--v-theme-surface-variant));
        border: 1px solid rgba(var(--v-theme-on-surface), 0.12);
        width: fit-content;
      "
    >
      <!-- Label -->
      <span
        class="d-flex align-center ga-1 text-caption font-weight-semibold text-uppercase"
        style="letter-spacing: 0.04em;"
      >
        <v-icon icon="mdi-shield-account-outline" size="16" />
        Admin
      </span>

      <!-- Tier override toggle -->
      <span class="text-caption text-medium-emphasis ml-2">Tier</span>
      <v-btn-toggle
        data-testid="tier-toggle"
        :model-value="localTier"
        mandatory
        density="compact"
        rounded="pill"
        @update:model-value="onTierSelect"
      >
        <v-btn
          data-testid="tier-btn-x"
          value="aim_x"
          size="small"
          variant="text"
        >
          AIM X
        </v-btn>
        <v-btn
          data-testid="tier-btn-pro"
          value="aim_pro"
          size="small"
          variant="text"
          :disabled="props.autoRecommended !== 'aim_pro'"
        >
          Pro
        </v-btn>
      </v-btn-toggle>

      <!-- Approve button -->
      <v-btn
        data-testid="csm-approve-btn"
        variant="outlined"
        color="primary"
        size="small"
        prepend-icon="mdi-check"
        :disabled="props.isApproved || approving"
        :loading="approving"
        @click="onApprove"
      >
        Approve
      </v-btn>
    </div>

    <!-- Error message (human copy only, no jargon) -->
    <v-alert
      v-if="errorMessage"
      data-testid="csm-error"
      variant="tonal"
      color="error"
      density="compact"
      icon="mdi-alert-circle-outline"
      class="mt-2"
      closable
      style="max-width: 820px;"
      @click:close="errorMessage = ''"
    >
      {{ errorMessage }}
    </v-alert>
  </template>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';
import { useOrganizationStore } from '@mos/core';
import { onboardingService } from '../services/onboarding';
import type { TierOverridePayload } from '../interfaces/aimOnboarding';

// ---------------------------------------------------------------------------
// Props & emits
// ---------------------------------------------------------------------------

const props = defineProps<{
  advertiserId: string;
  sessionId: string;
  /** Auto-recommended tier from recommendTier() — drives asymmetric enable rule. */
  autoRecommended: 'aim_x' | 'aim_pro';
  /** Currently effective tier (may differ if an override is already applied). */
  effectiveTier: 'aim_x' | 'aim_pro';
  /** True once the client has completed SoW approval — disables Approve. */
  isApproved: boolean;
}>();

const emit = defineEmits<{
  (e: 'approved'): void;
  (e: 'tier-changed', tier: 'aim_x' | 'aim_pro'): void;
}>();

// ---------------------------------------------------------------------------
// Store
// ---------------------------------------------------------------------------

const organizationStore = useOrganizationStore();

// ---------------------------------------------------------------------------
// Local state
// ---------------------------------------------------------------------------

/** Mirrors effectiveTier; reverted on 403. */
const localTier = ref<'aim_x' | 'aim_pro'>(props.effectiveTier);
const approving = ref(false);
const errorMessage = ref('');

// Keep localTier in sync if parent updates the prop (e.g. after reload).
watch(
  () => props.effectiveTier,
  (val) => { localTier.value = val; },
);

// ---------------------------------------------------------------------------
// Tier override
// ---------------------------------------------------------------------------

async function onTierSelect(selected: 'aim_x' | 'aim_pro') {
  if (selected === localTier.value) return;

  const previous = localTier.value;
  localTier.value = selected; // optimistic
  errorMessage.value = '';

  const payload: TierOverridePayload = { Override: selected };
  try {
    await onboardingService.tierOverride(props.advertiserId, props.sessionId, payload);
    emit('tier-changed', selected);
  } catch (err: unknown) {
    localTier.value = previous; // revert on failure
    errorMessage.value = is403(err)
      ? 'You do not have permission to change the tier. Contact your account manager.'
      : 'The tier change could not be saved. Please try again.';
  }
}

// ---------------------------------------------------------------------------
// Approve
// ---------------------------------------------------------------------------

async function onApprove() {
  approving.value = true;
  errorMessage.value = '';
  try {
    await onboardingService.approve(props.advertiserId, props.sessionId);
    emit('approved');
  } catch (err: unknown) {
    errorMessage.value = is403(err)
      ? 'You do not have permission to approve this session. Contact your account manager.'
      : 'Approval could not be submitted. Please try again.';
  } finally {
    approving.value = false;
  }
}

// ---------------------------------------------------------------------------
// Helper
// ---------------------------------------------------------------------------

function is403(err: unknown): boolean {
  return (
    typeof err === 'object' &&
    err !== null &&
    'response' in err &&
    (err as { response?: { status?: number } }).response?.status === 403
  );
}
</script>
```

- [ ] **Step 2: Run the tests — confirm all pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/CsmAdminBar.spec.ts
```

Expected output:

```text
 ✓ CsmAdminBar > renders nothing when isASAdmin is false
 ✓ CsmAdminBar > renders the admin bar when isASAdmin is true
 ✓ CsmAdminBar > renders the admin info banner when isASAdmin is true
 ✓ CsmAdminBar > Pro v-btn is disabled when auto-recommendation is aim_x
 ✓ CsmAdminBar > Pro v-btn is enabled when auto-recommendation is aim_pro
 ✓ CsmAdminBar > AIM X v-btn is always enabled when isASAdmin is true
 ✓ CsmAdminBar > selecting Pro fires tierOverride with Override aim_pro when auto=Pro
 ✓ CsmAdminBar > selecting AIM X fires tierOverride with Override aim_x
 ✓ CsmAdminBar > Approve button calls onboardingService.approve and emits approved
 ✓ CsmAdminBar > Approve button is disabled once isApproved is true
 ✓ CsmAdminBar > 403 on tierOverride shows human-readable error without 403 jargon
 ✓ CsmAdminBar > 403 on tierOverride reverts the toggle to prior value
 ✓ CsmAdminBar > 403 on approve shows human-readable error without 403 jargon

Test Files  1 passed (1)
Tests       13 passed (13)
```

- [ ] **Step 3: TypeScript check**

```bash
npx tsc --noEmit --project packages/advertiser/tsconfig.json \
  2>&1 | grep -i "aim\|onboarding\|CsmAdmin"
```

Expected: no output.

- [ ] **Step 4: Lint**

```bash
npx eslint packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/CsmAdminBar.vue --fix
```

Expected: no errors remaining.

- [ ] **Step 5: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/CsmAdminBar.vue
git commit -m "feat(aim-onboarding): add CsmAdminBar component (X1)"
```

---

## Task 3: Verify the isASAdmin gate end-to-end

This task validates the full client-mode path — no CSM controls must leak into
the rendered output when `isASAdmin` is false.

- [ ] **Step 1: Run the full test suite for the AimOnboardingTab**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab
```

Expected: all tests in the tab directory pass; `CsmAdminBar.spec.ts` 13/13.

- [ ] **Step 2: Run markdownlint on this plan file**

```bash
npx markdownlint-cli2 "/path/to/docs/docs/16-PRDs/061-aim-onboarding-decision-tool/plans/X1-csm-mode-ui.md"
```

Expected: no errors.

- [ ] **Step 3: Final commit**

```bash
git add .
git commit -m "feat(aim-onboarding): X1 CSM mode UI complete — CsmAdminBar, gating, asymmetric toggle, 403 handling"
```

---

## O7 integration seam

X1 owns `CsmAdminBar.vue`. O7's output shell renders it in the `.ohead` row:

```vue
<!-- In O7's OutputShell.vue or AimOnboardingTab.vue output section -->
<div class="ohead d-flex align-center ga-4 flex-wrap mb-4" style="max-width: 820px;">
  <div>
    <div class="text-h6 font-weight-bold">{{ companyName }}</div>
    <div class="text-caption text-medium-emphasis">{{ metaLine }}</div>
  </div>
  <v-spacer />
  <!-- tier chip (always visible) -->
  <v-chip
    :color="effectiveTier === 'aim_pro' ? 'warning' : 'primary'"
    variant="tonal"
    size="small"
    label
  >
    {{ effectiveTier === 'aim_pro' ? 'AIM Pro' : 'AIM X' }}
  </v-chip>

  <!-- CSM bar — self-gates on isASAdmin; client mode renders nothing -->
  <CsmAdminBar
    :advertiser-id="advertiserId"
    :session-id="sessionId"
    :auto-recommended="tier.Recommended ?? 'aim_x'"
    :effective-tier="tier.Effective ?? 'aim_x'"
    :is-approved="!!scopeOfWork.ApprovedAt"
    @approved="onCsmApproved"
    @tier-changed="onTierChanged"
  />
</div>
```

O7 handles the `approved` and `tier-changed` events (updating composable state
and re-fetching the session). X1 has no coupling to O7 internals.

---

## Self-review

**Spec coverage:**

- design-consolidated §7 CSM-only set: `approve` + `tier-override` both present.
- design-consolidated §8a: approval is terminal; no Unlock. The mockup's Unlock
  button is not ported.
- product-spec-v2 §4.3 override asymmetry: Pro `:disabled` when `autoRecommended !== 'aim_pro'`;
  AIM X always selectable; both directions send `{ Override: selectedTier }`.
- product-spec-v2 §4.6 user modes: `isASAdmin` gate; no URL param; no separate login.
- design-consolidated §7: "UI gate is convenience; 403 still handled." The component
  catches 403 and shows human copy; reverts optimistic toggle.
- design-consolidated §8b: `v-alert variant="tonal"` for banners; `v-btn-toggle`
  matching ActionBarV2 pattern (`:model-value` / `mandatory` / `density` /
  `@update:model-value`); colors via `rgb(var(--v-theme-*))`, no hardcoded hex.
- No `is_as_admin` or `403` text in any user-visible copy.
- `changeLog` entries are server-stamped (B6); the component never builds them.

**Placeholder scan:** No TBD / TODO / "fill in" / placeholder language. All
code blocks are complete.

**Test coverage (13 tests across 4 areas):**

| Area | Tests |
|------|-------|
| Gating | renders nothing (isASAdmin=false); renders bar + banner (isASAdmin=true); banner copy safe |
| Asymmetric toggle | Pro disabled when auto=X; Pro enabled when auto=Pro; X always enabled; Pro fires tierOverride; X fires tierOverride |
| Approve | fires approve + emits approved; disabled when isApproved=true |
| 403 handling | tierOverride 403 → human error, revert toggle; approve 403 → human error |
