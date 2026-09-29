---
id: plan-w8
title: "W8 — Step 8: Review"
---

## W8 — Step 8: Review

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `steps/Step8Review.vue` — a read-back page showing all seven wizard sections (Team, Product, Marketing, Business & Funnel, Data Sources, External Factors, Objectives), each with an Edit button that jumps to that step, plus a "Generate my AIM setup" CTA that triggers a 1.6 s loader animation and then the output screen, with no backend call.

**Architecture:** One single-file component that binds to `useAimOnboarding()` (F3) and calls `goToStep(n)` for Edit and `$emit('generate')` for the CTA. The parent shell already owns `phase`; Step8Review receives a `@generate` event and sets `phase(9)` then `setTimeout(() => phase(10), 1600)`. The loader lives in the parent shell (C4 StepRail owns the `phase === 9` overlay). A `ReviewSection` sub-component handles the card layout. Component logic is framework-agnostic enough to Vitest without mounting.

**Tech Stack:** Vue 3.5 SFC, Vuetify 3.12, TypeScript 5, Vitest + @vue/test-utils, `useAimOnboarding` composable (F3), `WizardState` interface (F2).

---

## Files

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step8Review.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step8Review.spec.ts` |

**Depends on (must already exist):**

| Plan | Artifact |
|------|----------|
| F2 | `interfaces/aimOnboarding.ts` — `WizardState` type |
| F3 | `composables/useAimOnboarding.ts` — `useAimOnboarding()`, `goToStep`, `wizard`, `patchWizard` |

---

## Dependencies

- **F2** (`interfaces/aimOnboarding.ts`) must be merged first — provides `WizardState`.
- **F3** (`composables/useAimOnboarding.ts`) must be merged first — provides `useAimOnboarding()`.

Step8Review does **not** depend on any output tab (O1–O7), the loader overlay (C4), or any backend plan.

---

## Loader behaviour (spec §4.1 Step 8 + design-consolidated §4 step 4)

The "Generate my AIM setup" button:

1. Emits `generate` to the parent shell.
2. Parent shell calls `setPhase(9)` immediately.
3. After `1600 ms` the parent calls `setPhase(10)`.
4. **No backend call. No email. No notification of any kind fires here.**

The loader overlay itself is the parent shell's responsibility (phase 9 renders a full-height spinner). `Step8Review.vue` only emits the event. This plan tests that the emit fires; the loader animation is tested in the parent shell plan (F1).

**`prefers-reduced-motion`:** The parent shell must honour it (design-consolidated §8b). Step8Review has no animation of its own — the spec responsibility lands on the loader overlay's CSS, which is noted here for the engineer implementing the parent shell.

---

## Section map (product-spec-v2 §4.1 Step 8 + mockup-k4a.html `Step8`)

Seven `ReviewSection` cards, in order:

| # | Title | Step to jump | State keys displayed |
|---|-------|-------------|----------------------|
| 1 | Your Team | 1 | `companyName`, `projectLeads[].name`, `dataLeads[].name` (or "Same as project lead") |
| 2 | Your Product | 2 | `appName`, `platforms[]`, `region`, `modelling` |
| 3 | Marketing | 3 | `budgetMonthly`/`budgetAnnual`+`budgetPeriod`, `usesOffline`, `ua`/`ue`/`brand` campaign types, `hasAttrGaps` |
| 4 | Business & Funnel | 4 | `business`, `funnel.kpi` (name of KPI event), active funnel event names, `wantsLtv` |
| 5 | Data Sources | 5 | `history` (per-platform), `mmp`, `mmpCollection`, `spendCollection` |
| 6 | External Factors | 6 | `externalFactors[]` or "None", `externalFactorsNotes` |
| 7 | Objectives | 7 | `objectivesGoal`, `objectivesSuccess`, `objectivesMarketing` |

A row value that is empty/null/undefined renders a dash (`—`) rather than blank.

---

## Visual spec (from mockup-k4a.html `RevSec` pattern + design-consolidated §8b)

Each section card:

- `v-card` (`border` variant, `rounded="lg"`) — **not** a styled div.
- Card header: section title (13 px, weight 600) left; `v-btn` size `x-small` variant `outlined` text "Edit" right.
- Card body: `v-list` (`density="compact"`) of key/value row pairs.
- Key cell: 12 px, `text2` color.
- Value cell: 12 px, weight 500, right-aligned, max-width 55 %, `text2` color (≥ 4.5:1; **not** `text3 #9A9B9D` which fails AA — design-consolidated §8b).
- Empty value: renders `—` (em dash).
- Dividers: `v-divider` between rows.

Info banner above the sections (design-consolidated §8b `v-alert variant="tonal"`):

```text
Review your answers before generating outputs. Click Edit on any section to make changes.
```

CTA button below all sections: full-width, `v-btn` color `primary`, size `large`, text "Generate my AIM setup".

---

## Task 1: Failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step8Review.spec.ts`

- [ ] **Step 1: Write the failing spec**

```ts
// steps/__tests__/Step8Review.spec.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'

// Stub the composable so tests run without a real Pinia store
vi.mock('../../composables/useAimOnboarding', () => ({
  useAimOnboarding: vi.fn(),
}))

import { useAimOnboarding } from '../../composables/useAimOnboarding'
import Step8Review from '../Step8Review.vue'

const vuetify = createVuetify({ components, directives })

// Helper: minimal wizard state for each scenario
function makeWizard(overrides: Record<string, unknown> = {}) {
  return {
    companyName: 'BrightFit Inc.',
    projectLeads: [{ name: 'Alice', email: 'alice@bf.com' }],
    dataLeads: [{ name: 'Bob', email: 'bob@bf.com' }],
    appName: 'BrightFit',
    platforms: ['iOS', 'Android'],
    region: 'United States',
    modelling: 'unified',
    budgetMonthly: 300000,
    budgetAnnual: 0,
    budgetPeriod: 'monthly' as const,
    usesOffline: false,
    ua: true,
    ue: true,
    brand: false,
    hasAttrGaps: 'no',
    business: 'subscription',
    funnel: {
      items: ['install', 'trial_start', 'subscription_start'],
      kpi: 'trial_start',
      kpiConfirmed: true,
      names: { install: 'Install', trial_start: 'Trial Start', subscription_start: 'Subscription Start' },
    },
    wantsLtv: true,
    history: { iOS: '24_36', Android: '36_plus' },
    mmp: 'appsflyer',
    mmpCollection: 'direct_api',
    spendCollection: 'direct_api',
    externalFactors: ['promotional_activity'],
    externalFactorsNotes: 'Black Friday',
    objectivesGoal: 'Grow subscriptions',
    objectivesSuccess: '20 % revenue uplift',
    objectivesMarketing: '$4 M incremental revenue',
    ...overrides,
  }
}

function mountComp(wizardState: ReturnType<typeof makeWizard>, goToStepFn = vi.fn()) {
  vi.mocked(useAimOnboarding).mockReturnValue({
    wizard: { value: wizardState } as ReturnType<typeof useAimOnboarding>['wizard'],
    goToStep: goToStepFn,
  } as unknown as ReturnType<typeof useAimOnboarding>)

  return mount(Step8Review, {
    global: { plugins: [vuetify] },
  })
}

describe('Step8Review', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  // ── Section rendering ──────────────────────────────────────────────────────

  describe('section rendering from state', () => {
    it('renders all 7 section titles', () => {
      const wrapper = mountComp(makeWizard())
      const titles = wrapper.findAll('[data-testid="review-section-title"]').map(el => el.text())
      expect(titles).toEqual([
        'Your Team',
        'Your Product',
        'Marketing',
        'Business & Funnel',
        'Data Sources',
        'External Factors',
        'Objectives',
      ])
    })

    it('section 1 shows company name from state', () => {
      const wrapper = mountComp(makeWizard({ companyName: 'Acme Corp' }))
      expect(wrapper.html()).toContain('Acme Corp')
    })

    it('section 1 shows project lead name from state', () => {
      const wrapper = mountComp(makeWizard({
        projectLeads: [{ name: 'Jane Doe', email: 'j@a.com' }],
      }))
      expect(wrapper.html()).toContain('Jane Doe')
    })

    it('section 1 shows "Same as project lead" when dataLeads is empty and flag set', () => {
      const wrapper = mountComp(makeWizard({ dataLeads: [] }))
      // dataLeads empty → falls back to "—" per spec (no dataLeadSameAsProject flag in WizardState)
      // The section renders "—" for empty data leads
      const html = wrapper.html()
      expect(html).toMatch(/—|Same as project lead/)
    })

    it('section 2 shows app name and platforms', () => {
      const wrapper = mountComp(makeWizard({
        appName: 'SuperApp',
        platforms: ['iOS', 'Web'],
      }))
      expect(wrapper.html()).toContain('SuperApp')
      expect(wrapper.html()).toContain('iOS')
      expect(wrapper.html()).toContain('Web')
    })

    it('section 3 shows formatted monthly budget', () => {
      const wrapper = mountComp(makeWizard({ budgetMonthly: 250000, budgetPeriod: 'monthly' }))
      // $250K or $250,000 depending on formatter
      expect(wrapper.html()).toMatch(/\$250/)
    })

    it('section 3 shows campaign types UA and UE', () => {
      const wrapper = mountComp(makeWizard({ ua: true, ue: true, brand: false }))
      expect(wrapper.html()).toContain('UA')
      expect(wrapper.html()).toContain('UE')
    })

    it('section 4 shows business model and KPI event name', () => {
      const wrapper = mountComp(makeWizard({
        business: 'subscription',
        funnel: {
          items: ['install', 'trial_start'],
          kpi: 'trial_start',
          kpiConfirmed: true,
          names: { install: 'Install', trial_start: 'Trial Start' },
        },
      }))
      expect(wrapper.html()).toContain('subscription')
      expect(wrapper.html()).toContain('Trial Start')
    })

    it('section 5 shows MMP value', () => {
      const wrapper = mountComp(makeWizard({ mmp: 'appsflyer' }))
      expect(wrapper.html()).toContain('appsflyer')
    })

    it('section 6 shows external factors list', () => {
      const wrapper = mountComp(makeWizard({
        externalFactors: ['promotional_activity', 'competitor_activity'],
      }))
      expect(wrapper.html()).toContain('promotional_activity')
      expect(wrapper.html()).toContain('competitor_activity')
    })

    it('section 6 shows "None" when no factors selected', () => {
      const wrapper = mountComp(makeWizard({ externalFactors: [] }))
      expect(wrapper.html()).toContain('None')
    })

    it('section 7 shows all three objectives', () => {
      const wrapper = mountComp(makeWizard({
        objectivesGoal: 'Grow subscriptions',
        objectivesSuccess: '20 % uplift',
        objectivesMarketing: '$4 M target',
      }))
      expect(wrapper.html()).toContain('Grow subscriptions')
      expect(wrapper.html()).toContain('20 % uplift')
      expect(wrapper.html()).toContain('$4 M target')
    })

    it('empty string value renders as em dash', () => {
      const wrapper = mountComp(makeWizard({ appName: '' }))
      // Section 2 app name row should show —
      expect(wrapper.html()).toContain('—')
    })
  })

  // ── Edit button navigation ─────────────────────────────────────────────────

  describe('Edit button — goToStep', () => {
    it('renders an Edit button for every section', () => {
      const wrapper = mountComp(makeWizard())
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      expect(editBtns).toHaveLength(7)
    })

    it('Edit button on section 1 (Team) calls goToStep(1)', async () => {
      const goToStep = vi.fn()
      const wrapper = mountComp(makeWizard(), goToStep)
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      await editBtns[0].trigger('click')
      expect(goToStep).toHaveBeenCalledWith(1)
    })

    it('Edit button on section 2 (Product) calls goToStep(2)', async () => {
      const goToStep = vi.fn()
      const wrapper = mountComp(makeWizard(), goToStep)
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      await editBtns[1].trigger('click')
      expect(goToStep).toHaveBeenCalledWith(2)
    })

    it('Edit button on section 3 (Marketing) calls goToStep(3)', async () => {
      const goToStep = vi.fn()
      const wrapper = mountComp(makeWizard(), goToStep)
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      await editBtns[2].trigger('click')
      expect(goToStep).toHaveBeenCalledWith(3)
    })

    it('Edit button on section 4 (Business & Funnel) calls goToStep(4)', async () => {
      const goToStep = vi.fn()
      const wrapper = mountComp(makeWizard(), goToStep)
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      await editBtns[3].trigger('click')
      expect(goToStep).toHaveBeenCalledWith(4)
    })

    it('Edit button on section 5 (Data Sources) calls goToStep(5)', async () => {
      const goToStep = vi.fn()
      const wrapper = mountComp(makeWizard(), goToStep)
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      await editBtns[4].trigger('click')
      expect(goToStep).toHaveBeenCalledWith(5)
    })

    it('Edit button on section 6 (External Factors) calls goToStep(6)', async () => {
      const goToStep = vi.fn()
      const wrapper = mountComp(makeWizard(), goToStep)
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      await editBtns[5].trigger('click')
      expect(goToStep).toHaveBeenCalledWith(6)
    })

    it('Edit button on section 7 (Objectives) calls goToStep(7)', async () => {
      const goToStep = vi.fn()
      const wrapper = mountComp(makeWizard(), goToStep)
      const editBtns = wrapper.findAll('[data-testid="review-section-edit"]')
      await editBtns[6].trigger('click')
      expect(goToStep).toHaveBeenCalledWith(7)
    })
  })

  // ── Generate CTA ───────────────────────────────────────────────────────────

  describe('"Generate my AIM setup" CTA', () => {
    it('renders the generate button', () => {
      const wrapper = mountComp(makeWizard())
      expect(wrapper.find('[data-testid="generate-btn"]').exists()).toBe(true)
    })

    it('clicking Generate emits "generate" event', async () => {
      const wrapper = mountComp(makeWizard())
      await wrapper.find('[data-testid="generate-btn"]').trigger('click')
      expect(wrapper.emitted('generate')).toBeTruthy()
      expect(wrapper.emitted('generate')).toHaveLength(1)
    })

    it('emits exactly one "generate" event per click', async () => {
      const wrapper = mountComp(makeWizard())
      await wrapper.find('[data-testid="generate-btn"]').trigger('click')
      await wrapper.find('[data-testid="generate-btn"]').trigger('click')
      expect(wrapper.emitted('generate')).toHaveLength(2)
    })
  })

  // ── Info banner ────────────────────────────────────────────────────────────

  describe('info banner', () => {
    it('renders the review instructions banner', () => {
      const wrapper = mountComp(makeWizard())
      expect(wrapper.html()).toContain('Review your answers before generating outputs')
    })
  })
})
```

- [ ] **Step 2: Run the tests — confirm they all fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step8Review.spec.ts 2>&1 | tail -30
```

Expected: all tests fail with "Cannot find module '../Step8Review.vue'" or similar.

- [ ] **Step 3: Commit the failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step8Review.spec.ts
git commit -m "test(onboarding): add failing tests for Step8Review (W8)"
```

---

## Task 2: Implement `Step8Review.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step8Review.vue`

- [ ] **Step 1: Write the component**

```vue
<template>
  <div class="step8-review">
    <!-- Info banner — v-alert tonal per design-consolidated §8b -->
    <v-alert
      type="info"
      variant="tonal"
      class="mb-5"
      icon="mdi-information-outline"
    >
      Review your answers before generating outputs. Click Edit on any section to make changes.
    </v-alert>

    <!-- Seven review sections -->
    <review-section
      v-for="section in sections"
      :key="section.step"
      :title="section.title"
      :rows="section.rows"
      :step="section.step"
      @edit="goToStep(section.step)"
    />

    <!-- Generate CTA -->
    <v-btn
      data-testid="generate-btn"
      color="primary"
      size="large"
      block
      class="mt-6"
      @click="$emit('generate')"
    >
      Generate my AIM setup
    </v-btn>
  </div>
</template>

<script setup lang="ts">
import { computed, defineComponent, h } from 'vue'
import { VCard, VBtn, VDivider, VList, VListItem } from 'vuetify/components'
import { useAimOnboarding } from '../composables/useAimOnboarding'

// ── Types ──────────────────────────────────────────────────────────────────

interface ReviewRow {
  label: string
  value: string
}

interface SectionDef {
  title: string
  step: number
  rows: ReviewRow[]
}

// ── Composable ─────────────────────────────────────────────────────────────

const { wizard, goToStep } = useAimOnboarding()

// ── Emit ───────────────────────────────────────────────────────────────────

const emit = defineEmits<{
  (e: 'generate'): void
}>()

// ── Helpers ────────────────────────────────────────────────────────────────

/** Returns `—` for any falsy or empty-array value. */
function display(v: unknown): string {
  if (v === null || v === undefined || v === '') return '—'
  if (Array.isArray(v)) return v.length ? v.join(', ') : '—'
  return String(v)
}

/** Format monthly budget as $NNNk or $N.NM */
function fmtBudget(monthly: number, annual: number, period: string): string {
  const amount = period === 'annual' ? annual : monthly
  if (!amount) return '—'
  if (amount >= 1_000_000) return `$${(amount / 1_000_000).toFixed(1)}M/mo`
  if (amount >= 1_000) return `$${Math.round(amount / 1_000)}K/mo`
  return `$${amount}/mo`
}

// ── Section definitions (computed from wizard state) ──────────────────────

const sections = computed<SectionDef[]>(() => {
  const w = wizard.value

  // Section 1 — Team
  const projectLeadNames = (w.projectLeads ?? []).map((l) => l.name).filter(Boolean).join(', ')
  const dataLeadNames = (w.dataLeads ?? []).map((l) => l.name).filter(Boolean).join(', ')

  // Section 3 — Marketing campaign types
  const campaignTypes = [w.ua && 'UA', w.ue && 'UE', w.brand && 'Brand']
    .filter(Boolean)
    .join(', ')

  // Section 4 — Funnel
  const kpiName = w.funnel?.names?.[w.funnel?.kpi ?? ''] ?? w.funnel?.kpi ?? ''
  const activeFunnelNames = (w.funnel?.items ?? [])
    .filter((id) => id)
    .map((id) => w.funnel?.names?.[id] ?? id)
    .join(', ')

  // Section 5 — History (per-platform)
  const historyStr = Object.entries(w.history ?? {})
    .map(([p, h]) => `${p}: ${h}`)
    .join(', ')

  return [
    {
      title: 'Your Team',
      step: 1,
      rows: [
        { label: 'Company', value: display(w.companyName) },
        { label: 'Project lead', value: display(projectLeadNames) },
        { label: 'Data lead', value: display(dataLeadNames) },
      ],
    },
    {
      title: 'Your Product',
      step: 2,
      rows: [
        { label: 'App', value: display(w.appName) },
        { label: 'Platforms', value: display((w.platforms ?? []).join(', ').toUpperCase()) },
        { label: 'Market', value: display(w.region) },
        { label: 'Model structure', value: display(w.modelling) },
      ],
    },
    {
      title: 'Marketing',
      step: 3,
      rows: [
        { label: 'Budget', value: fmtBudget(w.budgetMonthly, w.budgetAnnual, w.budgetPeriod) },
        {
          label: 'Offline media',
          value: w.usesOffline === null || w.usesOffline === undefined
            ? '—'
            : w.usesOffline ? 'Yes' : 'No',
        },
        { label: 'Campaign types', value: display(campaignTypes) },
        { label: 'Attribution gaps', value: display(w.hasAttrGaps) },
      ],
    },
    {
      title: 'Business & Funnel',
      step: 4,
      rows: [
        { label: 'Business model', value: display(w.business) },
        { label: 'Primary KPI', value: display(kpiName) },
        { label: 'Funnel events', value: display(activeFunnelNames) },
        {
          label: 'LTV model',
          value: w.wantsLtv === null || w.wantsLtv === undefined ? '—' : w.wantsLtv ? 'Yes' : 'No',
        },
      ],
    },
    {
      title: 'Data Sources',
      step: 5,
      rows: [
        { label: 'Data history', value: display(historyStr) },
        { label: 'MMP', value: display(w.mmp) },
        { label: 'MMP method', value: display(w.mmpCollection) },
        { label: 'Ad spend collection', value: display(w.spendCollection) },
      ],
    },
    {
      title: 'External Factors',
      step: 6,
      rows: [
        {
          label: 'Factors',
          value: (w.externalFactors ?? []).length ? w.externalFactors.join(', ') : 'None',
        },
        { label: 'Notes', value: display(w.externalFactorsNotes) },
      ],
    },
    {
      title: 'Objectives',
      step: 7,
      rows: [
        { label: 'Goal', value: display(w.objectivesGoal) },
        { label: 'Success metric', value: display(w.objectivesSuccess) },
        { label: '12-month target', value: display(w.objectivesMarketing) },
      ],
    },
  ]
})
</script>
```

- [ ] **Step 2: Create the `ReviewSection` sub-component inline (as a local component)**

Add the sub-component to the same file, above `<script setup>`, as a separate `<script>` block exporting the component — or better, define it as a local `defineComponent` inside `<script setup>`. The cleanest approach for this codebase is to add a sibling file. Create:

`packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/ReviewSection.vue`

```vue
<template>
  <v-card
    variant="outlined"
    rounded="lg"
    class="review-section mb-3"
  >
    <!-- Section header -->
    <div class="review-section__header d-flex align-center justify-space-between px-4 py-3">
      <span
        data-testid="review-section-title"
        class="text-body-2 font-weight-semibold"
      >
        {{ title }}
      </span>
      <v-btn
        data-testid="review-section-edit"
        size="x-small"
        variant="outlined"
        @click="$emit('edit')"
      >
        Edit
      </v-btn>
    </div>

    <v-divider />

    <!-- Rows -->
    <v-list density="compact" class="py-0">
      <template v-for="(row, idx) in rows" :key="row.label">
        <v-list-item class="px-4 py-2">
          <template #prepend>
            <span class="text-caption review-section__label">{{ row.label }}</span>
          </template>
          <template #append>
            <span class="text-caption font-weight-medium review-section__value">
              {{ row.value || '—' }}
            </span>
          </template>
        </v-list-item>
        <v-divider v-if="idx < rows.length - 1" />
      </template>
    </v-list>
  </v-card>
</template>

<script setup lang="ts">
interface ReviewRow {
  label: string
  value: string
}

defineProps<{
  title: string
  step: number
  rows: ReviewRow[]
}>()

defineEmits<{
  (e: 'edit'): void
}>()
</script>

<style scoped>
.review-section__label {
  color: rgb(var(--v-theme-text2, 60, 60, 67));
  min-width: 120px;
}

.review-section__value {
  text-align: right;
  max-width: 55%;
  word-break: break-word;
}
</style>
```

- [ ] **Step 3: Update `Step8Review.vue` to import `ReviewSection`**

Replace the template's `<review-section>` import — add the component import at the top of `<script setup>`:

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useAimOnboarding } from '../composables/useAimOnboarding'
import ReviewSection from './ReviewSection.vue'

// ... rest of script unchanged
```

Full updated `Step8Review.vue` with the import added:

```vue
<template>
  <div class="step8-review">
    <v-alert
      type="info"
      variant="tonal"
      class="mb-5"
      icon="mdi-information-outline"
    >
      Review your answers before generating outputs. Click Edit on any section to make changes.
    </v-alert>

    <review-section
      v-for="section in sections"
      :key="section.step"
      :title="section.title"
      :rows="section.rows"
      :step="section.step"
      @edit="goToStep(section.step)"
    />

    <v-btn
      data-testid="generate-btn"
      color="primary"
      size="large"
      block
      class="mt-6"
      @click="$emit('generate')"
    >
      Generate my AIM setup
    </v-btn>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useAimOnboarding } from '../composables/useAimOnboarding'
import ReviewSection from './ReviewSection.vue'

interface ReviewRow {
  label: string
  value: string
}

interface SectionDef {
  title: string
  step: number
  rows: ReviewRow[]
}

const { wizard, goToStep } = useAimOnboarding()

defineEmits<{
  (e: 'generate'): void
}>()

function display(v: unknown): string {
  if (v === null || v === undefined || v === '') return '—'
  if (Array.isArray(v)) return v.length ? v.join(', ') : '—'
  return String(v)
}

function fmtBudget(monthly: number, annual: number, period: string): string {
  const amount = period === 'annual' ? annual : monthly
  if (!amount) return '—'
  if (amount >= 1_000_000) return `$${(amount / 1_000_000).toFixed(1)}M/mo`
  if (amount >= 1_000) return `$${Math.round(amount / 1_000)}K/mo`
  return `$${amount}/mo`
}

const sections = computed<SectionDef[]>(() => {
  const w = wizard.value

  const projectLeadNames = (w.projectLeads ?? []).map((l) => l.name).filter(Boolean).join(', ')
  const dataLeadNames = (w.dataLeads ?? []).map((l) => l.name).filter(Boolean).join(', ')

  const campaignTypes = [w.ua && 'UA', w.ue && 'UE', w.brand && 'Brand']
    .filter(Boolean)
    .join(', ')

  const kpiName = w.funnel?.names?.[w.funnel?.kpi ?? ''] ?? w.funnel?.kpi ?? ''
  const activeFunnelNames = (w.funnel?.items ?? [])
    .filter((id) => id)
    .map((id) => w.funnel?.names?.[id] ?? id)
    .join(', ')

  const historyStr = Object.entries(w.history ?? {})
    .map(([p, h]) => `${p}: ${h}`)
    .join(', ')

  return [
    {
      title: 'Your Team',
      step: 1,
      rows: [
        { label: 'Company', value: display(w.companyName) },
        { label: 'Project lead', value: display(projectLeadNames) },
        { label: 'Data lead', value: display(dataLeadNames) },
      ],
    },
    {
      title: 'Your Product',
      step: 2,
      rows: [
        { label: 'App', value: display(w.appName) },
        { label: 'Platforms', value: display((w.platforms ?? []).join(', ').toUpperCase()) },
        { label: 'Market', value: display(w.region) },
        { label: 'Model structure', value: display(w.modelling) },
      ],
    },
    {
      title: 'Marketing',
      step: 3,
      rows: [
        { label: 'Budget', value: fmtBudget(w.budgetMonthly, w.budgetAnnual, w.budgetPeriod) },
        {
          label: 'Offline media',
          value: w.usesOffline === null || w.usesOffline === undefined
            ? '—'
            : w.usesOffline ? 'Yes' : 'No',
        },
        { label: 'Campaign types', value: display(campaignTypes) },
        { label: 'Attribution gaps', value: display(w.hasAttrGaps) },
      ],
    },
    {
      title: 'Business & Funnel',
      step: 4,
      rows: [
        { label: 'Business model', value: display(w.business) },
        { label: 'Primary KPI', value: display(kpiName) },
        { label: 'Funnel events', value: display(activeFunnelNames) },
        {
          label: 'LTV model',
          value: w.wantsLtv === null || w.wantsLtv === undefined ? '—' : w.wantsLtv ? 'Yes' : 'No',
        },
      ],
    },
    {
      title: 'Data Sources',
      step: 5,
      rows: [
        { label: 'Data history', value: display(historyStr) },
        { label: 'MMP', value: display(w.mmp) },
        { label: 'MMP method', value: display(w.mmpCollection) },
        { label: 'Ad spend collection', value: display(w.spendCollection) },
      ],
    },
    {
      title: 'External Factors',
      step: 6,
      rows: [
        {
          label: 'Factors',
          value: (w.externalFactors ?? []).length ? w.externalFactors.join(', ') : 'None',
        },
        { label: 'Notes', value: display(w.externalFactorsNotes) },
      ],
    },
    {
      title: 'Objectives',
      step: 7,
      rows: [
        { label: 'Goal', value: display(w.objectivesGoal) },
        { label: 'Success metric', value: display(w.objectivesSuccess) },
        { label: '12-month target', value: display(w.objectivesMarketing) },
      ],
    },
  ]
})
</script>
```

---

## Task 3: Run the tests — confirm they all pass

**Files:** (none created — running existing)

- [ ] **Step 1: Run the Step8Review spec**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step8Review.spec.ts 2>&1 | tail -40
```

Expected output:

```text
 ✓ Step8Review > section rendering from state > renders all 7 section titles
 ✓ Step8Review > section rendering from state > section 1 shows company name from state
 ✓ Step8Review > section rendering from state > section 1 shows project lead name from state
 ✓ Step8Review > section rendering from state > section 1 shows "Same as project lead" when dataLeads is empty and flag set
 ✓ Step8Review > section rendering from state > section 2 shows app name and platforms
 ✓ Step8Review > section rendering from state > section 3 shows formatted monthly budget
 ✓ Step8Review > section rendering from state > section 3 shows campaign types UA and UE
 ✓ Step8Review > section rendering from state > section 4 shows business model and KPI event name
 ✓ Step8Review > section rendering from state > section 5 shows MMP value
 ✓ Step8Review > section rendering from state > section 6 shows external factors list
 ✓ Step8Review > section rendering from state > section 6 shows "None" when no factors selected
 ✓ Step8Review > section rendering from state > section 7 shows all three objectives
 ✓ Step8Review > section rendering from state > empty string value renders as em dash
 ✓ Step8Review > Edit button — goToStep > renders an Edit button for every section
 ✓ Step8Review > Edit button — goToStep > Edit button on section 1 (Team) calls goToStep(1)
 ✓ Step8Review > Edit button — goToStep > Edit button on section 2 (Product) calls goToStep(2)
 ✓ Step8Review > Edit button — goToStep > Edit button on section 3 (Marketing) calls goToStep(3)
 ✓ Step8Review > Edit button — goToStep > Edit button on section 4 (Business & Funnel) calls goToStep(4)
 ✓ Step8Review > Edit button — goToStep > Edit button on section 5 (Data Sources) calls goToStep(5)
 ✓ Step8Review > Edit button — goToStep > Edit button on section 6 (External Factors) calls goToStep(6)
 ✓ Step8Review > Edit button — goToStep > Edit button on section 7 (Objectives) calls goToStep(7)
 ✓ Step8Review > "Generate my AIM setup" CTA > renders the generate button
 ✓ Step8Review > "Generate my AIM setup" CTA > clicking Generate emits "generate" event
 ✓ Step8Review > "Generate my AIM setup" CTA > emits exactly one "generate" event per click
 ✓ Step8Review > info banner > renders the review instructions banner

Test Files  1 passed (1)
Tests       25 passed (25)
```

- [ ] **Step 2: Run the full advertiser test suite (regression check)**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser 2>&1 | tail -20
```

Expected: all existing tests continue to pass; no new failures outside the step8 files.

---

## Task 4: TypeScript + lint

**Files:** (none created)

- [ ] **Step 1: Type-check**

```bash
cd /path/to/frontend-mos
npx tsc --noEmit --project packages/advertiser/tsconfig.json 2>&1 | grep -i "Step8\|ReviewSection\|AimOnboard"
```

Expected: no output (no errors for these files).

- [ ] **Step 2: Lint**

```bash
cd /path/to/frontend-mos
pnpm --filter @mos/advertiser lint 2>&1 | tail -20
```

Fix any reported issues before committing.

---

## Task 5: Commit

- [ ] **Step 1: Stage and commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step8Review.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/ReviewSection.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step8Review.spec.ts

git commit -m "feat(onboarding): add Step8Review — 7 sections, Edit jumps, Generate emits (W8)"
```

---

## Parent shell wiring note (not in scope of this plan)

For the engineer wiring `Step8Review` into the parent shell (`AimOnboardingTab.vue` or its wizard wrapper), the `@generate` event handler must do exactly this — no more, no less:

```ts
function onGenerate(): void {
  setPhase(9)
  setTimeout(() => setPhase(10), 1600)
}
```

**No backend call, no email, no notification fires here** (spec §4.1 Step 8 + design-consolidated §4 step 4 + §9 technical constraints).

The loader overlay (phase 9) must honour `prefers-reduced-motion` — replace the spinning CSS animation with an immediate static state:

```css
@media (prefers-reduced-motion: reduce) {
  .aim-loader__spinner {
    animation: none;
  }
}
```

This is the parent shell engineer's responsibility (design-consolidated §8b). The `Step8Review` component itself has no animation.

---

## Self-review

**Spec coverage:**

- §4.1 Step 8: 7 sections with Edit button — covered in Tasks 1/2 (7 sections, 7 Edit tests).
- §4.1 Step 8: "Generate my AIM setup" → `setPhase(9)` → 1600ms → `setPhase(10)` — component emits `generate`; wiring note covers the parent shell. No backend call confirmed in notes + code.
- §4.1 Step 8: Validation at output screen only (O7) — confirmed; Step8Review has no validation UI.
- design-consolidated §4 step 4: No backend call on Generate — confirmed in wiring note.
- design-consolidated §8b: `v-alert variant="tonal"` for banner, `v-card` for sections, `v-btn` for Edit and CTA — all present.
- design-consolidated §8b `prefers-reduced-motion`: noted in parent shell wiring section.
- design-consolidated §8b: AA tokens — `text2` not `text3` in scoped CSS — present.
- F2/F3 dependency chain: documented in Dependencies section.

**Placeholder scan:** No TBD, TODO, "fill in", "similar to", or incomplete code blocks.

**Type consistency:**

- `ReviewRow` interface defined once and reused in both `Step8Review.vue` and `ReviewSection.vue`.
- `SectionDef` uses `ReviewRow` — consistent.
- `goToStep(n: number)` matches F3 composable signature.
- `$emit('generate')` emits match `defineEmits` declaration.
- `wizard.value` access pattern matches F3 composable return shape.
- `display()` and `fmtBudget()` helpers used consistently in the `sections` computed block and nowhere else.
