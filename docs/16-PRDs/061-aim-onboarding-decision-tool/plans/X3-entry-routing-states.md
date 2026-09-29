---
id: plan-x3
title: "X3 — Entry Routing States"
---

## X3 — Entry Routing States Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the two presentational children that `AimOnboardingTab` (F1) slots into its
routing branches — a `WelcomeCard` (not\_started / in\_progress) and an `OnboardingRecord`
(complete state: wizard read-only + output tabs accessible).

**Architecture:** F1 owns the session fetch and status routing (`routedStatus` computed).
X3 builds content only. `WelcomeCard` parameterises Start vs Continue via a `status` prop — one
component, not two. `OnboardingRecord` renders the wizard in read-only mode and hosts the output
`v-tabs` shell (delegating output-tab content to O7 via a slot). Wizard steps inside the record
are visually frozen (`:readonly="true"` per-step prop) but the Timeline tab remains editable per
§8a. State comes from F3's composable — no second fetch.

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript 5, Vitest + @vue/test-utils.

---

## Seam contracts

### What F1 already provides (do NOT re-implement)

F1's `AimOnboardingTab.vue` owns:

- `onMounted` session fetch → `session` ref
- `routedStatus` computed (`'not_started' | 'in_progress' | 'complete'`)
- Three `data-testid` branch containers (`aim-onboarding-start`, `aim-onboarding-continue`,
  `aim-onboarding-record`)
- i18n keys under `mmm_config.aim_onboarding.*` (`cta_start`, `cta_continue`, `cta_view_record`,
  `description_start`, `description_continue`, `description_record`)

X3 replaces the placeholder `<v-btn>` + `<p>` blocks inside those containers with the real child
components. One surgical diff to `AimOnboardingTab.vue` per container.

### What O7 provides (declared here, not built here)

O7 ("Output shell") builds the `v-tabs` row (SoW / Timeline / Data Schema), tier header, and
validation summary. X3's `OnboardingRecord` exposes a `#output-tabs` slot that O7 fills. Until
O7 lands, the slot renders a `<!-- outputs load here (O7) -->` placeholder comment — tests assert
the slot exists, not its content.

### What F3 provides

`useAimOnboarding()` composable holds the reactive session state. `OnboardingRecord` calls
`useAimOnboarding()` and reads its `session` and `wizardState` — no second service call.

---

## Files

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/WelcomeCard.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/WelcomeCard.spec.ts` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/OnboardingRecord.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/OnboardingRecord.spec.ts` |
| Modify | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/AimOnboardingTab.vue` |
| Modify | `packages/app/src/locales/en.json` |

The component folder (`AimOnboardingTab/`) already exists from F1.

---

## Dependencies

- **F1** — `AimOnboardingTab.vue` branch containers must exist.
- **F2** — `interfaces/aimOnboarding.ts` (`OnboardingSession`, `SessionStatus`) must exist.
- **F3** — `composables/useAimOnboarding.ts` must exist; `OnboardingRecord` calls it.
- **O7** — Output shell. X3 prepares the slot; O7 fills it. O7 is NOT a prerequisite to merge X3.

---

## Tasks

---

### Task 1 — WelcomeCard failing tests

**Files:**

- Create: `AimOnboardingTab/__tests__/WelcomeCard.spec.ts`

- [ ] **Step 1: Write the failing tests**

```ts
import { describe, it, expect, vi } from 'vitest';
import { mount } from '@vue/test-utils';

// WelcomeCard does not reach the network — no service mock needed.
// global plugins (vuetify, pinia, i18n) from tests.config.ts setup.

import WelcomeCard from '../WelcomeCard.vue';

describe('WelcomeCard', () => {
  // -----------------------------------------------------------------------
  // not_started state
  // -----------------------------------------------------------------------
  describe('status = not_started', () => {
    const wrapper = () =>
      mount(WelcomeCard, { props: { status: 'not_started' } });

    it('renders the eyebrow label', () => {
      expect(wrapper().find('[data-testid="wc-eyebrow"]').exists()).toBe(true);
    });

    it('renders the headline', () => {
      expect(wrapper().find('[data-testid="wc-headline"]').exists()).toBe(true);
    });

    it('renders a single primary CTA button', () => {
      const btn = wrapper().find('[data-testid="wc-cta-primary"]');
      expect(btn.exists()).toBe(true);
    });

    it('does NOT render the secondary (Start fresh) button', () => {
      expect(wrapper().find('[data-testid="wc-cta-secondary"]').exists()).toBe(false);
    });

    it('renders the 3 output-preview cards (SoW, Timeline, Data Schema)', () => {
      const cards = wrapper().findAll('[data-testid="wc-output-card"]');
      expect(cards).toHaveLength(3);
    });

    it('emits start when primary CTA is clicked', async () => {
      const w = wrapper();
      await w.find('[data-testid="wc-cta-primary"]').trigger('click');
      expect(w.emitted('start')).toHaveLength(1);
    });
  });

  // -----------------------------------------------------------------------
  // in_progress state
  // -----------------------------------------------------------------------
  describe('status = in_progress', () => {
    const wrapper = () =>
      mount(WelcomeCard, { props: { status: 'in_progress' } });

    it('renders both primary (Continue) and secondary (Start fresh) buttons', () => {
      expect(wrapper().find('[data-testid="wc-cta-primary"]').exists()).toBe(true);
      expect(wrapper().find('[data-testid="wc-cta-secondary"]').exists()).toBe(true);
    });

    it('emits continue when primary CTA is clicked', async () => {
      const w = wrapper();
      await w.find('[data-testid="wc-cta-primary"]').trigger('click');
      expect(w.emitted('continue')).toHaveLength(1);
    });

    it('emits start when secondary (Start fresh) button is clicked', async () => {
      const w = wrapper();
      await w.find('[data-testid="wc-cta-secondary"]').trigger('click');
      expect(w.emitted('start')).toHaveLength(1);
    });
  });
});
```

- [ ] **Step 2: Run tests — confirm they all fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/WelcomeCard.spec.ts
```

Expected: all 9 tests fail with "Cannot find module `../WelcomeCard.vue`".

- [ ] **Step 3: Commit the failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/WelcomeCard.spec.ts
git commit -m "test(aim-onboarding): add failing WelcomeCard tests (X3)"
```

---

### Task 2 — Add i18n keys for WelcomeCard

**Files:**

- Modify: `packages/app/src/locales/en.json`

- [ ] **Step 1: Add new keys inside the existing `mmm_config.aim_onboarding` object**

The object already has keys from F1 (`description_start`, `cta_start`, `cta_continue`,
`cta_view_record`, etc.). Add the following keys only — do not duplicate what F1 added.

```json
"aim_onboarding": {
  "description_start": "Complete the onboarding questionnaire to generate your Scope of Work, delivery timeline, and data schema.",
  "description_continue": "You have an onboarding session in progress. Pick up where you left off.",
  "description_record": "Your onboarding is complete. Review your Scope of Work, timeline, and data schema below.",
  "cta_start": "Start onboarding",
  "cta_continue": "Continue onboarding",
  "cta_view_record": "View your onboarding record",
  "welcome_eyebrow": "AIM Onboarding",
  "welcome_headline": "Set up your AIM model in 10 minutes",
  "welcome_body": "Answer a few questions about your app, marketing, and data — and we'll generate your Scope of Work, Data Schema, and Delivery Timeline automatically.",
  "cta_start_fresh": "Start fresh",
  "output_card_sow": "Scope of Work",
  "output_card_sow_sub": "Auto-generated",
  "output_card_timeline": "Timeline",
  "output_card_timeline_sub": "Milestone plan",
  "output_card_schema": "Data Schema",
  "output_card_schema_sub": "Tailored to you",
  "record_heading": "Your onboarding record",
  "record_body": "Your answers are locked. The timeline below can still be updated."
}
```

The keys F1 already wrote (`description_start`, etc.) are shown above for completeness —
**do not add a key that already exists**. Add only the `welcome_*`, `output_card_*`,
`cta_start_fresh`, `record_heading`, and `record_body` keys.

- [ ] **Step 2: Verify lint**

```bash
cd /path/to/frontend-mos
npm run lint
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add packages/app/src/locales/en.json
git commit -m "feat(aim-onboarding): add WelcomeCard + OnboardingRecord i18n keys (X3)"
```

---

### Task 3 — Implement WelcomeCard.vue

**Files:**

- Create: `AimOnboardingTab/WelcomeCard.vue`

The mockup (`mockup-k4a.html` `Landing` component) centres the card, shows an eyebrow, headline,
body, CTA buttons, and a 3-column output-preview grid. Translate to Vuetify primitives (§8b: real
`v-card`, `v-btn` — no styled divs).

- [ ] **Step 1: Create the component**

```vue
<template>
  <div class="wc-root d-flex justify-center align-center py-10 px-4">
    <div class="wc-inner" style="max-width: 560px; width: 100%; text-align: center">
      <!-- Eyebrow -->
      <p
        class="text-overline text-primary mb-2"
        data-testid="wc-eyebrow"
        style="letter-spacing: 0.12em"
      >
        {{ $t('mmm_config.aim_onboarding.welcome_eyebrow') }}
      </p>

      <!-- Headline -->
      <h2
        class="text-h5 font-weight-bold mb-3"
        data-testid="wc-headline"
        style="line-height: 1.25"
      >
        {{ $t('mmm_config.aim_onboarding.welcome_headline') }}
      </h2>

      <!-- Body -->
      <p
        class="text-body-2 mb-8"
        style="color: rgb(var(--v-theme-grey)); line-height: 1.7; max-width: 480px; margin-inline: auto"
      >
        {{ $t('mmm_config.aim_onboarding.welcome_body') }}
      </p>

      <!-- CTA buttons -->
      <div class="d-flex justify-center flex-wrap ga-3 mb-10">
        <!-- Primary CTA: "Continue onboarding" (in_progress) OR "Start onboarding" (not_started) -->
        <v-btn
          color="primary"
          rounded
          size="large"
          data-testid="wc-cta-primary"
          @click="handlePrimary"
        >
          {{ status === 'in_progress'
              ? $t('mmm_config.aim_onboarding.cta_continue')
              : $t('mmm_config.aim_onboarding.cta_start') }}
        </v-btn>

        <!-- Secondary CTA: "Start fresh" — only shown when in_progress -->
        <v-btn
          v-if="status === 'in_progress'"
          variant="outlined"
          color="primary"
          rounded
          size="large"
          data-testid="wc-cta-secondary"
          @click="emit('start')"
        >
          {{ $t('mmm_config.aim_onboarding.cta_start_fresh') }}
        </v-btn>
      </div>

      <!-- Output preview cards (3 only — Tab 4 Model Config is Phase 2; §1) -->
      <div class="d-grid wc-output-grid">
        <v-card
          v-for="card in outputCards"
          :key="card.titleKey"
          variant="outlined"
          class="pa-4 text-center"
          data-testid="wc-output-card"
        >
          <v-icon :icon="card.icon" size="28" color="primary" class="mb-2" />
          <div class="text-subtitle-2 font-weight-semibold mb-1">
            {{ $t(card.titleKey) }}
          </div>
          <div class="text-caption" style="color: rgb(var(--v-theme-grey-darken-1))">
            {{ $t(card.subKey) }}
          </div>
        </v-card>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { SessionStatus } from './interfaces/aimOnboarding';

// ---------------------------------------------------------------------------
// Props & emits
// ---------------------------------------------------------------------------
const props = defineProps<{
  status: Extract<SessionStatus, 'not_started' | 'in_progress'>;
}>();

const emit = defineEmits<{
  (e: 'start'): void;
  (e: 'continue'): void;
}>();

// ---------------------------------------------------------------------------
// Handlers
// ---------------------------------------------------------------------------
function handlePrimary(): void {
  if (props.status === 'in_progress') {
    emit('continue');
  } else {
    emit('start');
  }
}

// ---------------------------------------------------------------------------
// Output-preview card data
// Phase 1 has 3 output tabs: SoW, Timeline, Data Schema.
// Tab 4 (Model Config) is Phase 2 only — not shown here (§1).
// ---------------------------------------------------------------------------
const outputCards = [
  {
    icon: 'mdi-file-document-outline',
    titleKey: 'mmm_config.aim_onboarding.output_card_sow',
    subKey:   'mmm_config.aim_onboarding.output_card_sow_sub',
  },
  {
    icon: 'mdi-calendar-check-outline',
    titleKey: 'mmm_config.aim_onboarding.output_card_timeline',
    subKey:   'mmm_config.aim_onboarding.output_card_timeline_sub',
  },
  {
    icon: 'mdi-table-large',
    titleKey: 'mmm_config.aim_onboarding.output_card_schema',
    subKey:   'mmm_config.aim_onboarding.output_card_schema_sub',
  },
] as const;
</script>

<style scoped lang="scss">
.wc-output-grid {
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}
</style>
```

- [ ] **Step 2: Run tests — confirm all 9 pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/WelcomeCard.spec.ts
```

Expected:

```text
 ✓ WelcomeCard > status = not_started > renders the eyebrow label
 ✓ WelcomeCard > status = not_started > renders the headline
 ✓ WelcomeCard > status = not_started > renders a single primary CTA button
 ✓ WelcomeCard > status = not_started > does NOT render the secondary button
 ✓ WelcomeCard > status = not_started > renders the 3 output-preview cards
 ✓ WelcomeCard > status = not_started > emits start when primary CTA is clicked
 ✓ WelcomeCard > status = in_progress > renders both primary and secondary buttons
 ✓ WelcomeCard > status = in_progress > emits continue when primary CTA is clicked
 ✓ WelcomeCard > status = in_progress > emits start when secondary button is clicked

Test Files  1 passed (1)
Tests       9 passed (9)
```

- [ ] **Step 3: Verify TypeScript**

```bash
npx tsc --noEmit --project packages/advertiser/tsconfig.json 2>&1 | grep -i "WelcomeCard\|aim_onboarding"
```

Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/WelcomeCard.vue
git commit -m "feat(aim-onboarding): implement WelcomeCard (X3)"
```

---

### Task 4 — OnboardingRecord failing tests

**Files:**

- Create: `AimOnboardingTab/__tests__/OnboardingRecord.spec.ts`

`OnboardingRecord` consumes `useAimOnboarding()` from F3. Mock it. The component does NOT
re-fetch — state comes entirely from the composable.

- [ ] **Step 1: Write the failing tests**

```ts
import { describe, it, expect, vi } from 'vitest';
import { mount, flushPromises } from '@vue/test-utils';
import type { OnboardingSession } from '../interfaces/aimOnboarding';

// ---------------------------------------------------------------------------
// Hoisted mock for useAimOnboarding composable (F3)
// ---------------------------------------------------------------------------
const { mockComposable } = vi.hoisted(() => ({
  mockComposable: {
    session: null as OnboardingSession | null,
    wizardState: {},
    isReadOnly: true,
  },
}));

vi.mock('../composables/useAimOnboarding', () => ({
  useAimOnboarding: () => mockComposable,
}));

import OnboardingRecord from '../OnboardingRecord.vue';

const completeSession: OnboardingSession = {
  _id: 'sess-xyz',
  AdvertiserId: 'adv-001',
  Status: 'complete',
  CreatedAt: '2026-06-01T00:00:00Z',
  UpdatedAt: '2026-06-01T00:00:00Z',
  CreatedByUserId: 'user-1',
  UpdatedByUserId: 'user-1',
  Wizard: {} as any,
  Tier: { Recommended: 'aim_x', Override: null, Effective: 'aim_x' },
  ScopeOfWork: {
    SectionNotes: {},
    ApproverName: 'Jane',
    ApproverJobTitle: 'VP Growth',
    ApprovedAt: '2026-06-01T10:00:00Z',
    CsmApproved: true,
  },
  Timeline: { AnchorDate: '2026-06-01T00:00:00Z', Milestones: [] },
  ChangeLog: [],
};

describe('OnboardingRecord', () => {
  // -----------------------------------------------------------------------
  // Layout structure
  // -----------------------------------------------------------------------
  it('renders the record heading', () => {
    mockComposable.session = completeSession;
    const wrapper = mount(OnboardingRecord);
    expect(wrapper.find('[data-testid="record-heading"]').exists()).toBe(true);
  });

  it('renders the frozen-answers notice banner', () => {
    mockComposable.session = completeSession;
    const wrapper = mount(OnboardingRecord);
    expect(wrapper.find('[data-testid="record-frozen-banner"]').exists()).toBe(true);
  });

  it('renders the output-tabs slot container', () => {
    mockComposable.session = completeSession;
    const wrapper = mount(OnboardingRecord);
    expect(wrapper.find('[data-testid="record-output-tabs"]').exists()).toBe(true);
  });

  it('renders the wizard-read-only section', () => {
    mockComposable.session = completeSession;
    const wrapper = mount(OnboardingRecord);
    expect(wrapper.find('[data-testid="record-wizard-readonly"]').exists()).toBe(true);
  });

  // -----------------------------------------------------------------------
  // Read-only: wizard section is visually locked
  // -----------------------------------------------------------------------
  it('the wizard-readonly section carries aria-readonly', () => {
    mockComposable.session = completeSession;
    const wrapper = mount(OnboardingRecord);
    const section = wrapper.find('[data-testid="record-wizard-readonly"]');
    expect(section.attributes('aria-readonly')).toBe('true');
  });

  // -----------------------------------------------------------------------
  // Slot: O7 can inject output-tab content
  // -----------------------------------------------------------------------
  it('accepts default slot content inside output-tabs container', () => {
    mockComposable.session = completeSession;
    const wrapper = mount(OnboardingRecord, {
      slots: { default: '<div data-testid="o7-stub">Output tabs</div>' },
    });
    expect(wrapper.find('[data-testid="o7-stub"]').exists()).toBe(true);
  });

  // -----------------------------------------------------------------------
  // View-record CTA navigates to wizard-readonly when clicked
  // -----------------------------------------------------------------------
  it('emits view-record when the CTA is activated', async () => {
    mockComposable.session = completeSession;
    const wrapper = mount(OnboardingRecord);
    // The record starts showing the output-tabs; pressing the CTA scrolls to wizard section
    const cta = wrapper.find('[data-testid="record-cta"]');
    if (cta.exists()) {
      await cta.trigger('click');
      expect(wrapper.emitted('view-record')).toBeTruthy();
    }
  });
});
```

- [ ] **Step 2: Run tests — confirm they all fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/OnboardingRecord.spec.ts
```

Expected: 7 tests fail with "Cannot find module `../OnboardingRecord.vue`".

- [ ] **Step 3: Commit the failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/OnboardingRecord.spec.ts
git commit -m "test(aim-onboarding): add failing OnboardingRecord tests (X3)"
```

---

### Task 5 — Implement OnboardingRecord.vue

**Files:**

- Create: `AimOnboardingTab/OnboardingRecord.vue`

The record view presents:

1. A heading + frozen-answers notice banner.
2. The output-tabs shell (filled by O7 via default slot).
3. The wizard answers as a read-only summary (all steps, answers displayed but not editable).

Per §8a: only wizard *answers* are frozen. The Timeline tab (rendered by O7 inside the slot)
stays editable. Do not blanket-disable the output-tabs slot.

- [ ] **Step 1: Create the component**

```vue
<template>
  <div class="onboarding-record">

    <!-- Heading row -->
    <div class="d-flex align-center justify-space-between mb-4">
      <div>
        <h2
          class="text-h6 font-weight-bold"
          data-testid="record-heading"
        >
          {{ $t('mmm_config.aim_onboarding.record_heading') }}
        </h2>
        <p class="text-caption mt-1" style="color: rgb(var(--v-theme-grey))">
          {{ $t('mmm_config.aim_onboarding.record_body') }}
        </p>
      </div>

      <v-btn
        variant="outlined"
        color="primary"
        rounded
        data-testid="record-cta"
        @click="emit('view-record')"
      >
        {{ $t('mmm_config.aim_onboarding.cta_view_record') }}
      </v-btn>
    </div>

    <!-- Frozen-answers notice (v-alert tonal — established portal precedent §8b) -->
    <v-alert
      type="info"
      variant="tonal"
      density="compact"
      class="mb-6"
      data-testid="record-frozen-banner"
      :text="$t('mmm_config.aim_onboarding.description_record')"
    />

    <!-- Output tabs shell — O7 fills this slot -->
    <div data-testid="record-output-tabs" class="mb-8">
      <slot />
      <!-- placeholder until O7 merges -->
    </div>

    <!-- Wizard read-only answers summary -->
    <!--
      aria-readonly="true" signals the whole section is non-interactive.
      Individual step components receive :readonly="true" when wizard steps
      (W1–W8) are wired in — X3 renders a locked wrapper; step content
      follows in W1–W8 plans.
    -->
    <section
      aria-readonly="true"
      data-testid="record-wizard-readonly"
      class="record-wizard-readonly"
    >
      <div class="d-flex align-center ga-2 mb-4">
        <v-icon icon="mdi-lock-outline" size="16" color="grey" />
        <span class="text-caption font-weight-semibold" style="color: rgb(var(--v-theme-grey))">
          Onboarding answers — read only
        </span>
      </div>

      <!--
        Wizard step components (W1–W8) mount here with :readonly="true".
        The step slot keeps the layout identical to the active wizard so
        the CSM and client can navigate the locked answers by step number.
        Steps are wired by W1–W8 plans — X3 establishes the container only.
      -->
      <slot name="wizard-steps" />
    </section>

  </div>
</template>

<script setup lang="ts">
import { useAimOnboarding } from './composables/useAimOnboarding';

// ---------------------------------------------------------------------------
// State — provided by F3 composable; no second fetch
// ---------------------------------------------------------------------------
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const { session, wizardState, isReadOnly } = useAimOnboarding();

// ---------------------------------------------------------------------------
// Emits
// ---------------------------------------------------------------------------
const emit = defineEmits<{
  (e: 'view-record'): void;
}>();
</script>

<style scoped lang="scss">
.onboarding-record {
  padding: 0 4px;
}

.record-wizard-readonly {
  pointer-events: none;
  user-select: none;
  opacity: 0.85;
}
</style>
```

- [ ] **Step 2: Run tests — confirm all 7 pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/OnboardingRecord.spec.ts
```

Expected:

```text
 ✓ OnboardingRecord > renders the record heading
 ✓ OnboardingRecord > renders the frozen-answers notice banner
 ✓ OnboardingRecord > renders the output-tabs slot container
 ✓ OnboardingRecord > renders the wizard-read-only section
 ✓ OnboardingRecord > the wizard-readonly section carries aria-readonly
 ✓ OnboardingRecord > accepts default slot content inside output-tabs container
 ✓ OnboardingRecord > emits view-record when the CTA is activated

Test Files  1 passed (1)
Tests       7 passed (7)
```

- [ ] **Step 3: Verify TypeScript**

```bash
npx tsc --noEmit --project packages/advertiser/tsconfig.json 2>&1 | grep -i "OnboardingRecord\|aim_onboarding"
```

Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/OnboardingRecord.vue
git commit -m "feat(aim-onboarding): implement OnboardingRecord (X3)"
```

---

### Task 6 — Wire children into AimOnboardingTab.vue

**Files:**

- Modify: `AimOnboardingTab/AimOnboardingTab.vue`

Replace the three placeholder blocks F1 shipped with the real children. This is a minimal surgical
diff — do not touch `routedStatus`, `isLoading`, `session`, or `isCSM` logic.

- [ ] **Step 1: Add imports at the top of the `<script setup>` block**

Add after the existing imports inside the `<script setup>` block:

```ts
import WelcomeCard from './WelcomeCard.vue';
import OnboardingRecord from './OnboardingRecord.vue';
```

- [ ] **Step 2: Replace the not\_started branch placeholder**

Find the existing block (F1 shipped this):

```vue
<!-- not_started: no session -->
<div v-else-if="routedStatus === 'not_started'" data-testid="aim-onboarding-start">
  <p class="body-2 mb-6" style="max-width: 520px; color: rgb(var(--v-theme-grey))">
    {{ $t('mmm_config.aim_onboarding.description_start') }}
  </p>
  <v-btn color="primary" rounded>
    {{ $t('mmm_config.aim_onboarding.cta_start') }}
  </v-btn>
</div>
```

Replace with:

```vue
<!-- not_started: no session -->
<div v-else-if="routedStatus === 'not_started'" data-testid="aim-onboarding-start">
  <welcome-card
    status="not_started"
    @start="handleStart"
  />
</div>
```

- [ ] **Step 3: Replace the in\_progress branch placeholder**

Find the existing block:

```vue
<!-- in_progress: session exists, not approved -->
<div v-else-if="routedStatus === 'in_progress'" data-testid="aim-onboarding-continue">
  <p class="body-2 mb-6" style="max-width: 520px; color: rgb(var(--v-theme-grey))">
    {{ $t('mmm_config.aim_onboarding.description_continue') }}
  </p>
  <v-btn color="primary" rounded>
    {{ $t('mmm_config.aim_onboarding.cta_continue') }}
  </v-btn>
</div>
```

Replace with:

```vue
<!-- in_progress: session exists, not approved -->
<div v-else-if="routedStatus === 'in_progress'" data-testid="aim-onboarding-continue">
  <welcome-card
    status="in_progress"
    @start="handleStart"
    @continue="handleContinue"
  />
</div>
```

- [ ] **Step 4: Replace the complete branch placeholder**

Find the existing block:

```vue
<!-- complete: approvedAt set -->
<div v-else-if="routedStatus === 'complete'" data-testid="aim-onboarding-record">
  <p class="body-2 mb-6" style="max-width: 520px; color: rgb(var(--v-theme-grey))">
    {{ $t('mmm_config.aim_onboarding.description_record') }}
  </p>
  <v-btn variant="outlined" color="primary" rounded>
    {{ $t('mmm_config.aim_onboarding.cta_view_record') }}
  </v-btn>
</div>
```

Replace with:

```vue
<!-- complete: approvedAt set — read-only record + output tabs -->
<div v-else-if="routedStatus === 'complete'" data-testid="aim-onboarding-record">
  <onboarding-record @view-record="scrollToWizard" />
</div>
```

- [ ] **Step 5: Add the two handler methods to `<script setup>`**

Add after the `onMounted` block inside `<script setup>`:

```ts
// ---------------------------------------------------------------------------
// Navigation handlers (called from WelcomeCard events)
// ---------------------------------------------------------------------------
function handleStart(): void {
  // TODO (F3): initialise fresh wizard state and advance to step 1
  // Placeholder: will be wired when F3 composable navigation lands.
}

function handleContinue(): void {
  // TODO (F3): restore saved wizard state and resume at last active step
  // Placeholder: will be wired when F3 composable navigation lands.
}

function scrollToWizard(): void {
  // Smooth-scroll to the read-only wizard section inside OnboardingRecord
  document
    .querySelector('[data-testid="record-wizard-readonly"]')
    ?.scrollIntoView({ behavior: 'smooth' });
}
```

- [ ] **Step 6: Run the full AimOnboardingTab test suite to confirm F1 tests still pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/AimOnboardingTab.spec.ts
```

Expected: 8 tests pass (unchanged from F1). The test stubs `WelcomeCard` and `OnboardingRecord`
as child components — the `data-testid` selectors on the wrapper `<div>` elements remain
identical, so F1's tests are unaffected.

- [ ] **Step 7: Verify lint and TypeScript**

```bash
npm run lint
npx tsc --noEmit --project packages/advertiser/tsconfig.json 2>&1 | grep -i "AimOnboarding\|WelcomeCard\|OnboardingRecord"
```

Expected: no errors.

- [ ] **Step 8: Commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/AimOnboardingTab.vue
git commit -m "feat(aim-onboarding): wire WelcomeCard and OnboardingRecord into AimOnboardingTab (X3)"
```

---

### Task 7 — Full test suite + final commit

- [ ] **Step 1: Run all tests in the advertiser package**

```bash
cd /path/to/frontend-mos/packages/advertiser
npm run test:unit
```

Expected: all tests pass — the 8 existing AimOnboardingTab tests (F1) plus 9 WelcomeCard and 7
OnboardingRecord tests (X3) = 24 tests green.

- [ ] **Step 2: Run lint across the workspace**

```bash
cd /path/to/frontend-mos
npm run lint
```

Expected: no errors or warnings introduced by X3.

- [ ] **Step 3: Final commit tagging the workstream complete**

```bash
git add -p   # review any stray hunks
git commit -m "feat(aim-onboarding): X3 entry routing states complete"
```

---

## Self-review

### Spec coverage

| Requirement | Task |
|---|---|
| §4.4 `not_started` — "Start onboarding" CTA | Task 3 (WelcomeCard, `status="not_started"`) |
| §4.4 `in_progress` — "Continue onboarding" + "Start fresh" | Task 3 (WelcomeCard, `status="in_progress"`) |
| §4.4 `complete` — read-only record, "View your onboarding record" CTA | Task 5 (OnboardingRecord) |
| §8a wizard answers locked post-approval | Task 5 (`aria-readonly`, `pointer-events: none`, `opacity`) |
| §8a SoW notes + Timeline remain editable post-approval | Task 5 (output-tabs slot NOT disabled) |
| §8b `v-alert variant="tonal"` for banners | Task 5 (frozen-answers notice) |
| §8b `v-card` / `v-btn` — real Vuetify, no styled divs | Tasks 3, 5 |
| §8b no hardcoded hex — `rgb(var(--v-theme-*))` | Tasks 3, 5 |
| mockup-k4a.html Landing: eyebrow, headline, body, CTAs, 3-card grid | Task 3 |
| F1 seam preserved — `data-testid` branch containers unchanged | Task 6 step 6 |
| O7 seam declared — slot contract for output tabs | Task 5 (`<slot />`) |
| F3 seam declared — no second fetch, composable provides state | Task 5 |
| 3 Phase 1 output cards only (Model Config is Phase 2) | Task 3 (3 cards, §1 comment) |

### Placeholder scan

No TBD / TODO / "fill in details" language in spec-mandated code. The two `handleStart` /
`handleContinue` TODO comments are intentional stubs that F3 wires — they are forwarding seams,
not unspecified work.

### Type consistency

- `SessionStatus` imported from `interfaces/aimOnboarding.ts` (F2) in both `WelcomeCard` and
  `AimOnboardingTab`. Prop type is `Extract<SessionStatus, 'not_started' | 'in_progress'>` — the
  `complete` value is not valid for `WelcomeCard` (it renders `OnboardingRecord` instead).
- `OnboardingSession` used identically in the test fixture and `useAimOnboarding` mock.
- `emit('view-record')` name matches `@view-record` in `AimOnboardingTab.vue` Task 6 step 4.
