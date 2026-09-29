---
id: plan-w5
title: "W5 — Step 5: Data Sources"
---

## W5 — Step 5: Data Sources Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `steps/Step5Data.vue` — the Data Sources wizard step that collects historical data depth per platform, MMP identity and collection method, AppsFlyer cohort access, web attribution sources, and ad spend method/sources, all bound to the `useAimOnboarding` composable via `patchWizard`.

**Architecture:** Controlled step component — pure UI and conditional reveals, zero business logic. All state lives in the module-scope `wizard` ref inside `useAimOnboarding`; this component calls `patchWizard(patch)` on every user action and reads from `wizard.value.*` for display. Conditional reveals are derived directly from `wizard.value.platforms`, `wizard.value.mmp`, and the fields themselves — no local state. The 12–24 month history warning is informational only (CG-007) — it renders a `v-alert` but does not gate any field, block navigation, or influence tier logic.

**Tech stack:** Vue 3.5, Vuetify 3.12, `@vue/test-utils`, Vitest, `useAimOnboarding` (F3), `ToggleCard` (C1), `InputSelect` (`packages/core`), `InputTags` (`packages/core`).

---

## Files

| Action | Path |
|--------|------|
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step5Data.vue` |
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step5Data.spec.ts` |

No other files are created or modified by this plan.

---

## Dependencies

- **F2** — `interfaces/aimOnboarding.ts` must exist (provides `WizardState` with `mmp`, `mmpCollection`, `appsflyerCohortAccess`, `webAttrSources`, `spendCollection`, `adSpendSources`, `history`).
- **F3** — `composables/useAimOnboarding.ts` must exist and export `{ wizard, patchWizard, reset }`.
- **C1** — `components/ToggleCard.vue` must exist (used for history depth and MMP collection method selection).

`InputSelect` and `InputTags` are imported from `@mos/core` (already a workspace dependency of `packages/advertiser`).

---

## Slug reference (authoritative — match F5/F6 consumers)

These string values are set by this step and consumed by `recommendTier` (F5) and `buildProvisionPlan` (F6). Do not deviate.

| Field | Values |
|-------|--------|
| `history[platform]` | `'12_24'` · `'24_36'` · `'36_plus'` |
| `mmp` | `'appsflyer'` · `'adjust'` · `'singular'` · `'branch'` · `'kochava'` · `'other_none'` |
| `mmpCollection` | `'direct_api'` · `'file'` · `'cloud'` |
| `spendCollection` | `'direct_api'` · `'file'` · `'cloud'` |
| `webAttrSources` items | `'ga4'` · `'adobe_analytics'` · `'amplitude'` · `'mixpanel'` · `'other'` |
| `adSpendSources` items | `'meta'` · `'google'` · `'tiktok'` · `'asa'` · `'dv360'` · `'the_trade_desk'` · `'other'` |

`history` is a `Record<string, string>` keyed by platform name (`'iOS'`, `'Android'`, `'Web'`). Only platforms present in `wizard.value.platforms` are rendered and stored.

---

## Composable binding pattern (set the precedent for all step plans)

Step components do not call `useAimOnboarding` directly in tests — the module-scope singleton leaks between tests. The pattern below isolates state cleanly:

**In the component:** call `useAimOnboarding()` at the top of `<script setup>`. Destructure only what is needed: `wizard` and `patchWizard`.

**In tests:** import `useAimOnboarding`, call `reset()` in `beforeEach`, then seed state via `patchWizard(...)` before mounting. Mount the component globally with the Vuetify plugin (already done in `tests.config.ts`). Do not mock `useAimOnboarding` — use the real singleton and reset it.

---

## Task 1: Write the failing spec

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step5Data.spec.ts`

- [ ] **Step 1: Create the spec directory if it does not exist**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__
  ```

  Expected: no error.

- [ ] **Step 2: Write the spec file**

  Create with this exact content:

  ```ts
  import { mount } from '@vue/test-utils'
  import { describe, it, expect, beforeEach } from 'vitest'
  import { useAimOnboarding } from '../../composables/useAimOnboarding'
  import Step5Data from '../Step5Data.vue'

  // Vuetify plugin + ResizeObserver stub provided by tests.config.ts.
  // Do NOT stub ToggleCard, InputSelect, or InputTags — their rendered output
  // carries the attributes we assert on.

  describe('Step5Data.vue', () => {
    let patchWizard: ReturnType<typeof useAimOnboarding>['patchWizard']
    let reset: ReturnType<typeof useAimOnboarding>['reset']

    beforeEach(() => {
      const composable = useAimOnboarding()
      patchWizard = composable.patchWizard
      reset = composable.reset
      reset()
    })

    // ── History per platform ─────────────────────────────────────────────────

    describe('history depth section', () => {
      it('renders one history group per selected platform', async () => {
        patchWizard({ platforms: ['iOS', 'Android'] })
        const wrapper = mount(Step5Data)
        expect(wrapper.findAll('[data-testid="history-group"]')).toHaveLength(2)
      })

      it('renders no history groups when no platform is selected', () => {
        const wrapper = mount(Step5Data)
        expect(wrapper.findAll('[data-testid="history-group"]')).toHaveLength(0)
      })

      it('renders three ToggleCards (12-24, 24-36, 36+) per platform group', async () => {
        patchWizard({ platforms: ['iOS'] })
        const wrapper = mount(Step5Data)
        const group = wrapper.find('[data-testid="history-group"]')
        expect(group.findAll('[data-testid="history-card"]')).toHaveLength(3)
      })

      it('selecting 12-24 months calls patchWizard with history entry', async () => {
        patchWizard({ platforms: ['iOS'] })
        const wrapper = mount(Step5Data)
        const card = wrapper.find('[data-testid="history-card-12_24"]')
        await card.trigger('click')
        const { wizard } = useAimOnboarding()
        expect(wizard.value.history['iOS']).toBe('12_24')
      })

      it('selecting 36+ months calls patchWizard with history entry', async () => {
        patchWizard({ platforms: ['Android'] })
        const wrapper = mount(Step5Data)
        const card = wrapper.find('[data-testid="history-card-36_plus"]')
        await card.trigger('click')
        const { wizard } = useAimOnboarding()
        expect(wizard.value.history['Android']).toBe('36_plus')
      })

      it('shows informational warning banner when history is 12_24 for any platform', async () => {
        patchWizard({ platforms: ['iOS'], history: { iOS: '12_24' } })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="history-warning-12_24"]').exists()).toBe(true)
      })

      it('does not show 12-24 warning banner when all platforms are 24+ months', async () => {
        patchWizard({ platforms: ['iOS'], history: { iOS: '24_36' } })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="history-warning-12_24"]').exists()).toBe(false)
      })

      it('shows green confirmation when history is 24_36', async () => {
        patchWizard({ platforms: ['iOS'], history: { iOS: '24_36' } })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="history-confirm-24plus"]').exists()).toBe(true)
      })

      it('12-24 warning banner does not disable any card or field', async () => {
        patchWizard({ platforms: ['iOS'], history: { iOS: '12_24' } })
        const wrapper = mount(Step5Data)
        // No card is aria-disabled due to the warning
        const disabledCards = wrapper
          .findAll('[data-testid="history-card"]')
          .filter(w => w.attributes('aria-disabled') === 'true')
        expect(disabledCards).toHaveLength(0)
      })
    })

    // ── MMP section ──────────────────────────────────────────────────────────

    describe('MMP section', () => {
      it('renders MMP select when iOS is a selected platform', async () => {
        patchWizard({ platforms: ['iOS'] })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="mmp-select"]').exists()).toBe(true)
      })

      it('renders MMP select when Android is a selected platform', async () => {
        patchWizard({ platforms: ['Android'] })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="mmp-select"]').exists()).toBe(true)
      })

      it('does not render MMP section when only Web is selected', async () => {
        patchWizard({ platforms: ['Web'] })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="mmp-select"]').exists()).toBe(false)
      })

      it('does not render collection method cards until MMP is selected', async () => {
        patchWizard({ platforms: ['iOS'], mmp: '' })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="mmp-collection-group"]').exists()).toBe(false)
      })

      it('renders collection method cards after MMP is selected', async () => {
        patchWizard({ platforms: ['iOS'], mmp: 'appsflyer' })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="mmp-collection-group"]').exists()).toBe(true)
        expect(wrapper.findAll('[data-testid="mmp-collection-card"]')).toHaveLength(3)
      })

      it('marks Direct API card as selected when mmpCollection is direct_api', async () => {
        patchWizard({ platforms: ['iOS'], mmp: 'appsflyer', mmpCollection: 'direct_api' })
        const wrapper = mount(Step5Data)
        const card = wrapper.find('[data-testid="mmp-collection-card-direct_api"]')
        expect(card.attributes('aria-checked')).toBe('true')
      })

      it('does not render AppsFlyer cohort access question when MMP is Adjust', async () => {
        patchWizard({ platforms: ['iOS'], mmp: 'adjust' })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="appsflyer-cohort"]').exists()).toBe(false)
      })

      it('renders AppsFlyer cohort access question when MMP is AppsFlyer', async () => {
        patchWizard({ platforms: ['iOS'], mmp: 'appsflyer' })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="appsflyer-cohort"]').exists()).toBe(true)
      })

      it('stores appsflyerCohortAccess=true when Yes is selected', async () => {
        patchWizard({ platforms: ['iOS'], mmp: 'appsflyer' })
        const wrapper = mount(Step5Data)
        await wrapper.find('[data-testid="appsflyer-cohort-yes"]').trigger('click')
        const { wizard } = useAimOnboarding()
        expect(wizard.value.appsflyerCohortAccess).toBe(true)
      })

      it('stores appsflyerCohortAccess=false when No is selected', async () => {
        patchWizard({ platforms: ['iOS'], mmp: 'appsflyer', appsflyerCohortAccess: true })
        const wrapper = mount(Step5Data)
        await wrapper.find('[data-testid="appsflyer-cohort-no"]').trigger('click')
        const { wizard } = useAimOnboarding()
        expect(wizard.value.appsflyerCohortAccess).toBe(false)
      })
    })

    // ── Web attribution section ──────────────────────────────────────────────

    describe('web attribution section', () => {
      it('does not render web attribution section when Web is not selected', async () => {
        patchWizard({ platforms: ['iOS'] })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="web-attr-section"]').exists()).toBe(false)
      })

      it('renders web attribution section when Web is selected', async () => {
        patchWizard({ platforms: ['Web'] })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="web-attr-section"]').exists()).toBe(true)
      })

      it('renders web attribution section when Web is one of multiple platforms', async () => {
        patchWizard({ platforms: ['iOS', 'Web'] })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="web-attr-section"]').exists()).toBe(true)
      })

      it('updates webAttrSources via InputTags', async () => {
        patchWizard({ platforms: ['Web'], webAttrSources: [] })
        const wrapper = mount(Step5Data)
        // Simulate the component updating state (InputTags emits update:model-value)
        const inputTags = wrapper.findComponent({ name: 'InputTags' })
        await inputTags.vm.$emit('update:model-value', ['ga4', 'amplitude'])
        const { wizard } = useAimOnboarding()
        expect(wizard.value.webAttrSources).toEqual(['ga4', 'amplitude'])
      })
    })

    // ── Ad spend section ─────────────────────────────────────────────────────

    describe('ad spend section', () => {
      it('always renders the ad spend section', () => {
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="ad-spend-section"]').exists()).toBe(true)
      })

      it('renders three collection method cards for ad spend', () => {
        const wrapper = mount(Step5Data)
        expect(wrapper.findAll('[data-testid="spend-collection-card"]')).toHaveLength(3)
      })

      it('marks Direct API card selected when spendCollection is direct_api', async () => {
        patchWizard({ spendCollection: 'direct_api' })
        const wrapper = mount(Step5Data)
        const card = wrapper.find('[data-testid="spend-collection-card-direct_api"]')
        expect(card.attributes('aria-checked')).toBe('true')
      })

      it('renders ad spend source chips after spendCollection is set', async () => {
        patchWizard({ spendCollection: 'file' })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="ad-spend-sources"]').exists()).toBe(true)
      })

      it('does not render ad spend source chips until spendCollection is set', () => {
        patchWizard({ spendCollection: '' })
        const wrapper = mount(Step5Data)
        expect(wrapper.find('[data-testid="ad-spend-sources"]').exists()).toBe(false)
      })

      it('updates adSpendSources via InputTags', async () => {
        patchWizard({ spendCollection: 'file', adSpendSources: [] })
        const wrapper = mount(Step5Data)
        const inputTags = wrapper.findComponent({ name: 'InputTags' })
        await inputTags.vm.$emit('update:model-value', ['meta', 'google'])
        const { wizard } = useAimOnboarding()
        expect(wizard.value.adSpendSources).toEqual(['meta', 'google'])
      })
    })
  })
  ```

- [ ] **Step 3: Run the tests — confirm they fail**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step5Data.spec.ts 2>&1 | tail -30
  ```

  Expected: all tests **fail** (module not found — `Step5Data.vue` does not exist yet). This is the correct TDD red state.

---

## Task 2: Implement Step5Data.vue

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step5Data.vue`

- [ ] **Step 1: Create the steps directory if it does not exist**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps
  ```

  Expected: no error.

- [ ] **Step 2: Write Step5Data.vue**

  Create the file with this content:

  ```vue
  <template>
    <div class="step5-data">
      <!-- ── Historical data per platform ─────────────────────────────── -->
      <section v-if="hasPlatforms" class="step5-data__section">
        <h3 class="step5-data__section-title text-body-1 font-weight-semibold mb-4">
          Historical data
        </h3>
        <p class="text-body-2 mb-4" style="color: rgb(var(--v-theme-grey-2))">
          How far back does your available data go, per platform?
        </p>

        <div
          v-for="platform in wizard.platforms"
          :key="platform"
          class="mb-6"
          data-testid="history-group"
        >
          <p class="text-body-2 font-weight-medium mb-2">{{ platform }}</p>
          <div class="step5-data__card-row">
            <ToggleCard
              v-for="opt in HISTORY_OPTIONS"
              :key="opt.value"
              :title="opt.label"
              :selected="wizard.history[platform] === opt.value"
              :data-testid="`history-card-${opt.value}`"
              data-testid-group="history-card"
              @toggle="setHistory(platform, opt.value)"
            />
          </div>

          <!-- 12-24 months informational warning (CG-007: NOT a gate) -->
          <v-alert
            v-if="wizard.history[platform] === '12_24'"
            variant="tonal"
            color="warning"
            density="compact"
            class="mt-2"
            icon="mdi-alert-outline"
            data-testid="history-warning-12_24"
          >
            Seasonality will be partially modelled with 12–24 months of data. This does not
            affect your tier eligibility.
          </v-alert>

          <!-- 24+ months confirmation -->
          <v-alert
            v-if="wizard.history[platform] === '24_36' || wizard.history[platform] === '36_plus'"
            variant="tonal"
            color="success"
            density="compact"
            class="mt-2"
            icon="mdi-check-circle-outline"
            data-testid="history-confirm-24plus"
          >
            Great — full seasonality modelling is supported.
          </v-alert>
        </div>
      </section>

      <!-- ── MMP selection (mobile only) ──────────────────────────────── -->
      <section v-if="hasMobile" class="step5-data__section">
        <h3 class="step5-data__section-title text-body-1 font-weight-semibold mb-2">
          Mobile measurement partner (MMP)
        </h3>
        <p class="text-body-2 mb-4" style="color: rgb(var(--v-theme-grey-2))">
          Which MMP do you use for mobile attribution?
          <span aria-hidden="true" style="color: rgb(var(--v-theme-error))">*</span>
        </p>

        <InputSelect
          :model-value="wizard.mmp || null"
          :items="MMP_ITEMS"
          label="Select MMP"
          no-search
          variant="outlined"
          density="compact"
          aria-required="true"
          data-testid="mmp-select"
          @update:model-value="(v) => patchWizard({ mmp: v ?? '' })"
        />

        <!-- MMP collection method (revealed once MMP chosen) -->
        <div v-if="wizard.mmp" class="mt-5">
          <p class="text-body-2 font-weight-medium mb-2">How will Kochava collect MMP data?</p>
          <div class="step5-data__card-row" data-testid="mmp-collection-group">
            <ToggleCard
              v-for="opt in COLLECTION_OPTIONS"
              :key="opt.value"
              :title="opt.label"
              :description="opt.description"
              :selected="wizard.mmpCollection === opt.value"
              :data-testid="`mmp-collection-card-${opt.value}`"
              data-testid-group="mmp-collection-card"
              @toggle="patchWizard({ mmpCollection: opt.value })"
            />
          </div>
        </div>

        <!-- AppsFlyer cohort access (revealed only if mmp === 'appsflyer') -->
        <div v-if="wizard.mmp === 'appsflyer'" class="mt-5" data-testid="appsflyer-cohort">
          <p class="text-body-2 font-weight-medium mb-2">
            Do you have AppsFlyer cohort data access?
          </p>
          <div class="step5-data__card-row">
            <ToggleCard
              title="Yes"
              :selected="wizard.appsflyerCohortAccess === true"
              data-testid="appsflyer-cohort-yes"
              @toggle="patchWizard({ appsflyerCohortAccess: true })"
            />
            <ToggleCard
              title="No"
              :selected="wizard.appsflyerCohortAccess === false && wizard.mmp === 'appsflyer'"
              data-testid="appsflyer-cohort-no"
              @toggle="patchWizard({ appsflyerCohortAccess: false })"
            />
          </div>
        </div>
      </section>

      <!-- ── Web attribution sources (Web only) ───────────────────────── -->
      <section v-if="hasWeb" class="step5-data__section" data-testid="web-attr-section">
        <h3 class="step5-data__section-title text-body-1 font-weight-semibold mb-2">
          Web attribution sources
        </h3>
        <p class="text-body-2 mb-4" style="color: rgb(var(--v-theme-grey-2))">
          Which tools do you use to measure web conversions?
          <span aria-hidden="true" style="color: rgb(var(--v-theme-error))">*</span>
        </p>

        <InputTags
          :model-value="wizard.webAttrSources"
          :items="WEB_ATTR_ITEMS"
          label="Select web attribution sources"
          variant="outlined"
          density="compact"
          aria-required="true"
          data-testid="web-attr-tags"
          @update:model-value="(v) => patchWizard({ webAttrSources: (v as string[]) ?? [] })"
        />
      </section>

      <!-- ── Ad spend data ─────────────────────────────────────────────── -->
      <section class="step5-data__section" data-testid="ad-spend-section">
        <h3 class="step5-data__section-title text-body-1 font-weight-semibold mb-2">
          Ad spend data
        </h3>
        <p class="text-body-2 mb-2" style="color: rgb(var(--v-theme-grey-2))">
          How will Kochava collect your ad spend data?
          <span aria-hidden="true" style="color: rgb(var(--v-theme-error))">*</span>
        </p>

        <div class="step5-data__card-row">
          <ToggleCard
            v-for="opt in COLLECTION_OPTIONS"
            :key="opt.value"
            :title="opt.label"
            :description="opt.description"
            :selected="wizard.spendCollection === opt.value"
            :data-testid="`spend-collection-card-${opt.value}`"
            data-testid-group="spend-collection-card"
            @toggle="patchWizard({ spendCollection: opt.value })"
          />
        </div>

        <!-- Ad spend sources (revealed once collection method chosen) -->
        <div v-if="wizard.spendCollection" class="mt-5" data-testid="ad-spend-sources">
          <p class="text-body-2 font-weight-medium mb-2">
            Which ad platforms or channels will you provide spend data for?
          </p>
          <InputTags
            :model-value="wizard.adSpendSources"
            :items="AD_SPEND_ITEMS"
            label="Select ad spend sources"
            variant="outlined"
            density="compact"
            data-testid="ad-spend-tags"
            @update:model-value="(v) => patchWizard({ adSpendSources: (v as string[]) ?? [] })"
          />
        </div>
      </section>
    </div>
  </template>

  <script setup lang="ts">
  import { computed } from 'vue'
  import { useAimOnboarding } from '../composables/useAimOnboarding'
  import ToggleCard from '../components/ToggleCard.vue'
  import InputSelect from '@mos/core/src/components/inputs/InputSelect.vue'
  import InputTags from '@mos/core/src/components/inputs/InputTags.vue'

  const { wizard, patchWizard } = useAimOnboarding()

  // ── Computed flags ────────────────────────────────────────────────────────

  const hasPlatforms = computed(() => wizard.value.platforms.length > 0)
  const hasMobile = computed(() =>
    wizard.value.platforms.some(p => p === 'iOS' || p === 'Android'),
  )
  const hasWeb = computed(() => wizard.value.platforms.includes('Web'))

  // ── Helpers ───────────────────────────────────────────────────────────────

  function setHistory(platform: string, value: string): void {
    patchWizard({ history: { ...wizard.value.history, [platform]: value } })
  }

  // ── Static option lists ───────────────────────────────────────────────────

  const HISTORY_OPTIONS = [
    { value: '12_24', label: '12–24 months' },
    { value: '24_36', label: '24–36 months' },
    { value: '36_plus', label: '36+ months' },
  ] as const

  const MMP_ITEMS = [
    { title: 'AppsFlyer', value: 'appsflyer' },
    { title: 'Adjust', value: 'adjust' },
    { title: 'Singular', value: 'singular' },
    { title: 'Branch', value: 'branch' },
    { title: 'Kochava', value: 'kochava' },
    { title: 'Other / None', value: 'other_none' },
  ]

  const COLLECTION_OPTIONS = [
    { value: 'direct_api', label: 'Direct API', description: 'Recommended' },
    { value: 'file', label: 'File upload', description: undefined },
    { value: 'cloud', label: 'Cloud storage', description: undefined },
  ] as const

  const WEB_ATTR_ITEMS = [
    { title: 'Google Analytics 4', value: 'ga4' },
    { title: 'Adobe Analytics', value: 'adobe_analytics' },
    { title: 'Amplitude', value: 'amplitude' },
    { title: 'Mixpanel', value: 'mixpanel' },
    { title: 'Other', value: 'other' },
  ]

  const AD_SPEND_ITEMS = [
    { title: 'Meta', value: 'meta' },
    { title: 'Google', value: 'google' },
    { title: 'TikTok', value: 'tiktok' },
    { title: 'Apple Search Ads', value: 'asa' },
    { title: 'DV360', value: 'dv360' },
    { title: 'The Trade Desk', value: 'the_trade_desk' },
    { title: 'Other', value: 'other' },
  ]
  </script>

  <style scoped lang="scss">
  .step5-data {
    max-width: 720px;

    &__section {
      margin-bottom: 40px;

      &:last-child {
        margin-bottom: 0;
      }
    }

    &__section-title {
      color: rgb(var(--v-theme-black));
    }

    &__card-row {
      display: flex;
      gap: 12px;
      flex-wrap: wrap;
    }
  }
  </style>
  ```

**Design notes (locked):**

- All color tokens use `rgb(var(--v-theme-*))` — no hardcoded hex.
- `v-alert variant="tonal"` with `color="warning"` uses `--alert-orange #BA4E00` token (AA-compliant per §8b). The 12-24 banner is display-only — it carries no `disabled` prop, no aria gate, and does not affect any sibling field.
- `ToggleCard` receives `data-testid` per card value so tests can assert selection state via `aria-checked`. The `data-testid-group` attribute lets `wrapper.findAll('[data-testid-group="history-card"]')` group all history cards for count assertions.
- `InputSelect` uses `no-search` (renders `v-select` not `v-autocomplete`) — appropriate for a bounded MMP list. Emits `update:modelValue`.
- `InputTags` uses `v-autocomplete` with chips — appropriate for multi-select web attribution and ad spend sources. Emits `update:model-value` (note hyphenated form matches the `InputTags` emit declaration).
- `setHistory` spreads the existing `history` record so patches to one platform do not erase another platform's value.
- The `appsflyerCohortAccess` "No" card's `:selected` binding checks both the boolean and `wizard.mmp === 'appsflyer'` to prevent a stale `false` from appearing selected after MMP changes.
- `other_none` MMP stores as `'other_none'` — buildProvisionPlan (F6) already guards `mmp !== 'other_none'` before adding MMP rows.

---

## Task 3: Run tests — confirm all pass

- [ ] **Step 1: Run the Step5Data spec**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step5Data.spec.ts 2>&1 | tail -40
  ```

  Expected output (28 tests):

  ```text
  ✓ Step5Data.vue
    ✓ history depth section
      ✓ renders one history group per selected platform
      ✓ renders no history groups when no platform is selected
      ✓ renders three ToggleCards (12-24, 24-36, 36+) per platform group
      ✓ selecting 12-24 months calls patchWizard with history entry
      ✓ selecting 36+ months calls patchWizard with history entry
      ✓ shows informational warning banner when history is 12_24 for any platform
      ✓ does not show 12-24 warning banner when all platforms are 24+ months
      ✓ shows green confirmation when history is 24_36
      ✓ 12-24 warning banner does not disable any card or field
    ✓ MMP section
      ✓ renders MMP select when iOS is a selected platform
      ✓ renders MMP select when Android is a selected platform
      ✓ does not render MMP section when only Web is selected
      ✓ does not render collection method cards until MMP is selected
      ✓ renders collection method cards after MMP is selected
      ✓ marks Direct API card as selected when mmpCollection is direct_api
      ✓ does not render AppsFlyer cohort access question when MMP is Adjust
      ✓ renders AppsFlyer cohort access question when MMP is AppsFlyer
      ✓ stores appsflyerCohortAccess=true when Yes is selected
      ✓ stores appsflyerCohortAccess=false when No is selected
    ✓ web attribution section
      ✓ does not render web attribution section when Web is not selected
      ✓ renders web attribution section when Web is selected
      ✓ renders web attribution section when Web is one of multiple platforms
      ✓ updates webAttrSources via InputTags
    ✓ ad spend section
      ✓ always renders the ad spend section
      ✓ renders three collection method cards for ad spend
      ✓ marks Direct API card selected when spendCollection is direct_api
      ✓ renders ad spend source chips after spendCollection is set
      ✓ does not render ad spend source chips until spendCollection is set
      ✓ updates adSpendSources via InputTags

  Test Files  1 passed (1)
  Tests       30 passed (30)
  ```

  If any test fails, diagnose before proceeding.

- [ ] **Step 2: Run the full advertiser test suite to confirm no regressions**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run --project packages/advertiser 2>&1 | tail -20
  ```

  Expected: existing tests all pass; Step5Data spec shows passing.

---

## Task 4: Type-check, lint, and commit

- [ ] **Step 1: Type-check**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && \
    npx tsc --noEmit 2>&1 | head -30
  ```

  Expected: zero errors. Fix any TS errors before proceeding.

- [ ] **Step 2: Lint the two new files**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx eslint \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step5Data.vue \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step5Data.spec.ts
  ```

  Expected: no errors or warnings. Fix any lint issues before committing.

- [ ] **Step 3: Run markdown lint on this plan file (repo CLAUDE.md requirement)**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/aim-onboarding-tool/docs && \
    npx markdownlint-cli2 "docs/16-PRDs/061-aim-onboarding-decision-tool/plans/W5-step5-data-sources.md"
  ```

  Expected: no errors.

- [ ] **Step 4: Commit**

  ```bash
  git add \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step5Data.vue \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step5Data.spec.ts
  git commit -m "feat(aim-onboarding): W5 Step5Data — history depth, MMP, web attr, ad spend with conditional reveals + Vitest"
  ```

  Expected: commit succeeds; CI test run passes.

---

## Self-review: spec coverage check

| Requirement (spec §4.1 Step 5) | Covered |
|---|---|
| Historical data per platform — 12–24/24–36/36+ ToggleCards | Task 1 tests + Task 2 history group loop |
| 12–24 months shows informational warning, NOT a tier gate (CG-007) | Warn-renders test + no-disable test + design notes |
| 24+ months shows green confirmation | History-confirm test + `v-alert color="success"` |
| MMP select — AppsFlyer/Adjust/Singular/Branch/Kochava/Other-none | MMP_ITEMS list + select tests |
| MMP section only shown when mobile platform selected | hasMobile computed + tests |
| MMP collection method (Direct API/File/Cloud) — Direct API recommended | COLLECTION_OPTIONS with description "Recommended" + tests |
| Collection method revealed only after MMP chosen | `v-if="wizard.mmp"` + test |
| AppsFlyer cohort access — shown only if mmp === 'appsflyer' | `v-if="wizard.mmp === 'appsflyer'"` + tests |
| Web attribution sources — shown only if Web platform | hasWeb computed + tests |
| Web attribution via chips (InputTags) | InputTags + webAttrSources update test |
| Ad spend — collection method first | spend-collection-card tests |
| Ad spend — source chips after method chosen | `v-if="wizard.spendCollection"` + tests |
| Ad spend source chips (InputTags) | InputTags + adSpendSources update test |
| Binds to useAimOnboarding composable | patchWizard calls throughout + reset/seed pattern |
| history is per-platform Record | setHistory spreads existing history + slug tests |
| `other_none` MMP stored as slug (feeds buildProvisionPlan F6) | MMP_ITEMS value + F6 guard compatibility |
| All colors via `rgb(var(--v-theme-*))` | SCSS + inline style — no hardcoded hex |
| CG-007: 12-24 qualifies for tier algorithm (not gated) | Plan explicitly delegates tier to F5; no tier logic in component |
| TDD: test → fail → implement → pass → commit | Tasks 1 → 2 → 3 → 4 |
| Lint-clean markdown | Task 4 Step 3 |
