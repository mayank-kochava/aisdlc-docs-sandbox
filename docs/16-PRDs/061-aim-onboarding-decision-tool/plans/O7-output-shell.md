---
id: plan-o7
title: "O7 — Output Shell"
---

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `output/OutputScreen.vue` — the container rendered after the 1.6-second Generate animation (W8) and as the read-only record when status is `complete`. Owns the tier header, validation summary with deep-link back to offending steps, and the plain `v-tabs`/`v-window` shell hosting the three child tabs (O1/O4/O6). Does not build child tab content.

**Architecture:** Single component receiving wizard state and tier via `useAimOnboarding` (F3). Derives display state from two composable flags — `allRequiredDone` (F3) and `isComplete` (approval terminal). Emits `edit(step, field?)` for deep-link navigation back to the wizard; parent (F1) owns output↔wizard toggling. Approval is terminal per §8a — no Unlock. Validation summary renders one `v-alert variant="tonal" color="error"` row per failing required step, first invalid field only.

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript, Vitest + @vue/test-utils, child tabs stubbed in tests.

---

## Files

**Create**

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/OutputScreen.vue
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/OutputScreen.spec.ts
```

**Consumed (must exist before this plan)**

```text
AimOnboardingTab/composables/useAimOnboarding.ts             ← F3
AimOnboardingTab/logic/validation.ts                         ← F7
AimOnboardingTab/output/SowTab.vue                           ← O1 (stub in tests)
AimOnboardingTab/output/TimelineTab.vue                      ← O4 (stub in tests)
AimOnboardingTab/output/DataSchemaTab.vue                    ← O6 (stub in tests)
```

---

## Dependencies

**F3** (`useAimOnboarding`) — provides `wizard`, `stepStatus`, `allRequiredDone`, `goToStep`, `visitedSteps`.

**F7** (`logic/validation`) — `validateStep(stepKey, wizard)` returns `{ valid: boolean; invalidFields: string[] }`. O7 calls it to read the first `invalidField` per failing step for the summary text.

**O1** (`SowTab.vue`), **O4** (`TimelineTab.vue`), **O6** (`DataSchemaTab.vue`) — rendered inside `v-window-item`s; must exist at their import paths (or be empty stubs) before this plan merges.

---

## §8a State Matrix (authoritative)

| Condition | Validation summary | "Edit answers" button |
|-----------|-------------------|-----------------------|
| Post-Generate, `!allRequiredDone` | **visible** | visible |
| Post-Generate, `allRequiredDone` | **hidden** | visible |
| `isComplete` (approval terminal) | hidden | **hidden** |

`isComplete` = `wizard.value.approvedAt != null`. When complete, the wizard questionnaire is hidden by the parent (F1); OutputScreen shows output tabs only. There is no Unlock affordance.

---

## Step Labels + Field Labels (copy verbatim — no "appropriate text" placeholder)

```ts
// Used in validation summary rows
const STEP_LABELS: Record<number, string> = {
  1: 'Step 1 — Your Team',
  2: 'Step 2 — About Your Product',
  3: 'Step 3 — Marketing Setup',
  4: 'Step 4 — Business & Funnel',
  5: 'Step 5 — Data Sources',
  8: 'Step 8 — Review',
}

const FIELD_LABELS: Record<string, string> = {
  companyName: 'Company name is required',
  projectLeads: 'At least one project lead (name + email) is required',
  dataLeads: 'At least one data lead is required',
  appName: 'App / brand name is required',
  platforms: 'Select at least one platform (iOS, Android, or Web)',
  region: 'Primary market is required',
  budgetMonthly: 'Total paid media budget is required',
  budgetAnnual: 'Total paid media budget is required',
  digitalMediaTypes: 'Select at least one digital media type',
  hasAttrGaps: 'Attribution data reliability answer is required',
  coverageConfidence: 'Coverage confidence answer is required',
  ua: 'Select at least one campaign type (UA, UE, or Brand)',
  business: 'Business model is required',
  'funnel.kpi': 'A KPI event must be selected in your funnel',
  mmp: 'MMP selection is required for mobile platforms',
  spendCollection: 'Ad spend data collection method is required',
  adSpendSources: 'At least one ad spend source is required',
  history: 'Data history selection is required for each platform',
}

/**
 * Fallback for any invalidField key not in FIELD_LABELS.
 * Converts 'camelCase' → 'camel case is required'.
 */
function fieldLabel(key: string): string {
  return (
    FIELD_LABELS[key] ??
    `${key.replace(/([A-Z])/g, ' $1').toLowerCase()} is required`
  )
}
```

---

## Tasks

### Task 1 — Write the failing spec

- [ ] **1.1** Create:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/OutputScreen.spec.ts
```

Contents:

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'

// ── Stub child tabs (O1/O4/O6 don't exist yet) ───────────────────────────────
vi.mock('../SowTab.vue', () => ({ default: { template: '<div data-testid="sow-tab" />' } }))
vi.mock('../TimelineTab.vue', () => ({ default: { template: '<div data-testid="timeline-tab" />' } }))
vi.mock('../DataSchemaTab.vue', () => ({ default: { template: '<div data-testid="schema-tab" />' } }))

// ── Stub useAimOnboarding (F3) ────────────────────────────────────────────────
import { ref, computed } from 'vue'

type StepStatus = 'done' | 'error' | 'pending' | 'optional'

interface MockComposable {
  wizard: ReturnType<typeof ref>
  stepStatus: ReturnType<typeof computed>
  allRequiredDone: ReturnType<typeof computed>
  goToStep: ReturnType<typeof vi.fn>
  visitedSteps: ReturnType<typeof ref>
}

const mockWizard = ref({
  companyName: 'BrightFit Inc.',
  appName: 'BrightFit',
  region: 'United States',
  approvedAt: null as string | null,
  tier: { effective: 'aim_x' as 'aim_x' | 'aim_pro', override: null, recommended: 'aim_x' as 'aim_x' | 'aim_pro' },
})
const mockStepStatus = ref<Record<number, StepStatus>>({
  1: 'done', 2: 'done', 3: 'done', 4: 'done',
  5: 'done', 6: 'optional', 7: 'optional', 8: 'done',
})
const mockAllRequiredDone = computed(() =>
  [1, 2, 3, 4, 5, 8].every((s) => mockStepStatus.value[s] === 'done'),
)
const mockGoToStep = vi.fn()
const mockVisitedSteps = ref(new Set<number>())

vi.mock('../../composables/useAimOnboarding', () => ({
  useAimOnboarding: (): MockComposable => ({
    wizard: mockWizard,
    stepStatus: computed(() => mockStepStatus.value),
    allRequiredDone: mockAllRequiredDone,
    goToStep: mockGoToStep,
    visitedSteps: mockVisitedSteps,
  }),
}))

// ── Stub validateStep (F7) ────────────────────────────────────────────────────
const mockValidateStep = vi.fn(
  (_stepKey: string, _wizard: unknown): { valid: boolean; invalidFields: string[] } => ({
    valid: true,
    invalidFields: [],
  }),
)

vi.mock('../../logic/validation', () => ({
  validateStep: (...args: Parameters<typeof mockValidateStep>) => mockValidateStep(...args),
}))

import OutputScreen from '../OutputScreen.vue'

// ── Helpers ───────────────────────────────────────────────────────────────────

function mountScreen(overrides: { approvedAt?: string | null } = {}) {
  if (overrides.approvedAt !== undefined) {
    mockWizard.value = { ...mockWizard.value, approvedAt: overrides.approvedAt }
  }
  const vuetify = createVuetify({ components, directives })
  return mount(OutputScreen, {
    global: { plugins: [vuetify] },
  })
}

describe('OutputScreen', () => {
  beforeEach(() => {
    mockWizard.value = {
      companyName: 'BrightFit Inc.',
      appName: 'BrightFit',
      region: 'United States',
      approvedAt: null,
      tier: { effective: 'aim_x', override: null, recommended: 'aim_x' },
    }
    mockStepStatus.value = {
      1: 'done', 2: 'done', 3: 'done', 4: 'done',
      5: 'done', 6: 'optional', 7: 'optional', 8: 'done',
    }
    mockGoToStep.mockClear()
    mockValidateStep.mockImplementation(() => ({ valid: true, invalidFields: [] }))
    mockVisitedSteps.value = new Set()
  })

  // ── Tier header ───────────────────────────────────────────────────────────

  describe('tier header', () => {
    it('renders appName from wizard', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('BrightFit')
    })

    it('renders region subline', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('United States')
    })

    it('renders "Generated from your 8-step setup" line', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('Generated from your 8-step setup')
    })

    it('shows tier chip with aim_x label when tier.effective = aim_x', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('AIM X')
    })

    it('shows tier chip with aim_pro label when tier.effective = aim_pro', async () => {
      mockWizard.value = {
        ...mockWizard.value,
        tier: { effective: 'aim_pro', override: null, recommended: 'aim_pro' },
      }
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('AIM Pro')
    })

    it('shows "Edit answers" button when not complete', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('Edit answers')
    })

    it('hides "Edit answers" button when complete (approval is terminal)', async () => {
      const wrapper = mountScreen({ approvedAt: '2026-06-11T10:00:00Z' })
      await flushPromises()
      expect(wrapper.text()).not.toContain('Edit answers')
    })

    it('emits edit with no step when "Edit answers" clicked', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      const btn = wrapper.findAll('button').find((b) => b.text().includes('Edit answers'))
      await btn?.trigger('click')
      const emitted = wrapper.emitted('edit')
      expect(emitted).toBeTruthy()
      expect(emitted![0]).toEqual([undefined])
    })
  })

  // ── Validation summary ────────────────────────────────────────────────────

  describe('validation summary', () => {
    it('is hidden when allRequiredDone is true', async () => {
      // default mockStepStatus has all required done
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.find('[data-testid="validation-summary"]').exists()).toBe(false)
    })

    it('is visible when a required step is not done', async () => {
      mockStepStatus.value = { ...mockStepStatus.value, 3: 'error' }
      mockValidateStep.mockImplementation((stepKey) => {
        if (stepKey === '3') return { valid: false, invalidFields: ['budgetMonthly'] }
        return { valid: true, invalidFields: [] }
      })
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.find('[data-testid="validation-summary"]').exists()).toBe(true)
    })

    it('shows step label for each failing step', async () => {
      mockStepStatus.value = { ...mockStepStatus.value, 3: 'error' }
      mockValidateStep.mockImplementation((stepKey) => {
        if (stepKey === '3') return { valid: false, invalidFields: ['budgetMonthly'] }
        return { valid: true, invalidFields: [] }
      })
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('Step 3 — Marketing Setup')
    })

    it('shows first invalid field message', async () => {
      mockStepStatus.value = { ...mockStepStatus.value, 3: 'error' }
      mockValidateStep.mockImplementation((stepKey) => {
        if (stepKey === '3') return { valid: false, invalidFields: ['budgetMonthly'] }
        return { valid: true, invalidFields: [] }
      })
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.text()).toContain('Total paid media budget is required')
    })

    it('shows one row per failing required step, not more', async () => {
      mockStepStatus.value = { ...mockStepStatus.value, 2: 'error', 3: 'error' }
      mockValidateStep.mockImplementation((stepKey) => {
        if (stepKey === '2') return { valid: false, invalidFields: ['appName'] }
        if (stepKey === '3') return { valid: false, invalidFields: ['budgetMonthly'] }
        return { valid: true, invalidFields: [] }
      })
      const wrapper = mountScreen()
      await flushPromises()
      expect(wrapper.findAll('[data-testid="validation-row"]').length).toBe(2)
    })

    it('optional steps (6, 7) never produce a validation row', async () => {
      // optional steps are never error — stepStatus returns 'optional' regardless
      mockStepStatus.value = { ...mockStepStatus.value }
      const wrapper = mountScreen()
      await flushPromises()
      const rows = wrapper.findAll('[data-testid="validation-row"]')
      expect(rows.length).toBe(0)
    })

    it('is hidden when isComplete', async () => {
      mockStepStatus.value = { ...mockStepStatus.value, 3: 'error' }
      const wrapper = mountScreen({ approvedAt: '2026-06-11T10:00:00Z' })
      await flushPromises()
      expect(wrapper.find('[data-testid="validation-summary"]').exists()).toBe(false)
    })

    describe('deep-link: "Go back to fix →" button', () => {
      it('emits edit event with the failing step number', async () => {
        mockStepStatus.value = { ...mockStepStatus.value, 3: 'error' }
        mockValidateStep.mockImplementation((stepKey) => {
          if (stepKey === '3') return { valid: false, invalidFields: ['budgetMonthly'] }
          return { valid: true, invalidFields: [] }
        })
        const wrapper = mountScreen()
        await flushPromises()

        const fixBtn = wrapper.find('[data-testid="fix-btn-3"]')
        await fixBtn.trigger('click')

        const emitted = wrapper.emitted('edit')
        expect(emitted).toBeTruthy()
        expect(emitted![0][0]).toBe(3)
      })

      it('marks the target step visited (so rail shows error state)', async () => {
        mockStepStatus.value = { ...mockStepStatus.value, 3: 'error' }
        mockValidateStep.mockImplementation((stepKey) => {
          if (stepKey === '3') return { valid: false, invalidFields: ['budgetMonthly'] }
          return { valid: true, invalidFields: [] }
        })
        const wrapper = mountScreen()
        await flushPromises()

        const fixBtn = wrapper.find('[data-testid="fix-btn-3"]')
        await fixBtn.trigger('click')

        expect(mockVisitedSteps.value.has(3)).toBe(true)
      })
    })
  })

  // ── Output tabs ───────────────────────────────────────────────────────────

  describe('output tabs', () => {
    it('renders three tabs: Scope of Work, Timeline, Data Schema', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      const text = wrapper.text()
      expect(text).toContain('Scope of Work')
      expect(text).toContain('Timeline')
      expect(text).toContain('Data Schema')
    })

    it('does NOT render numbered circles on tab labels', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      // Tab labels must be plain text — no digits before the label names
      const tabEls = wrapper.findAll('[role="tab"]')
      tabEls.forEach((tab) => {
        expect(tab.text()).not.toMatch(/^[123]\s/)
      })
    })

    it('defaults to the Scope of Work tab active', async () => {
      const wrapper = mountScreen()
      await flushPromises()
      // SowTab stub is present in the initial render
      expect(wrapper.find('[data-testid="sow-tab"]').exists()).toBe(true)
    })

    it('switches to Timeline tab on click', async () => {
      const wrapper = mountScreen()
      await flushPromises()

      const tabs = wrapper.findAll('[role="tab"]')
      const timelineTab = tabs.find((t) => t.text().includes('Timeline'))
      await timelineTab?.trigger('click')
      await flushPromises()

      expect(wrapper.find('[data-testid="timeline-tab"]').exists()).toBe(true)
    })

    it('switches to Data Schema tab on click', async () => {
      const wrapper = mountScreen()
      await flushPromises()

      const tabs = wrapper.findAll('[role="tab"]')
      const schemaTab = tabs.find((t) => t.text().includes('Data Schema'))
      await schemaTab?.trigger('click')
      await flushPromises()

      expect(wrapper.find('[data-testid="schema-tab"]').exists()).toBe(true)
    })
  })

  // ── Complete (read-only record) state ─────────────────────────────────────

  describe('complete / read-only record state', () => {
    it('renders output tabs when complete', async () => {
      const wrapper = mountScreen({ approvedAt: '2026-06-11T10:00:00Z' })
      await flushPromises()
      expect(wrapper.text()).toContain('Scope of Work')
    })

    it('does NOT render validation summary when complete', async () => {
      mockStepStatus.value = { ...mockStepStatus.value, 3: 'error' }
      const wrapper = mountScreen({ approvedAt: '2026-06-11T10:00:00Z' })
      await flushPromises()
      expect(wrapper.find('[data-testid="validation-summary"]').exists()).toBe(false)
    })

    it('does NOT render "Edit answers" button when complete', async () => {
      const wrapper = mountScreen({ approvedAt: '2026-06-11T10:00:00Z' })
      await flushPromises()
      expect(wrapper.text()).not.toContain('Edit answers')
    })
  })
})
```

- [ ] **1.2** Run to confirm compile failure (component does not exist yet):

```bash
cd packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/OutputScreen.spec.ts 2>&1 | tail -30
```

Expected: `Error: Failed to resolve import` or similar — component file absent.

---

### Task 2 — Create OutputScreen.vue

- [ ] **2.1** Create:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/OutputScreen.vue
```

Contents:

```vue
<template>
  <div class="output-screen">
    <!-- ── Tier header ───────────────────────────────────────────────────── -->
    <div class="output-screen__header">
      <div class="output-screen__identity">
        <span class="output-screen__app-name">{{ appName }}</span>
        <span class="output-screen__meta">{{ companyName }} · {{ region }}</span>
      </div>

      <div class="output-screen__header-actions">
        <!-- Tier chip — chips-orange for Pro, chips-blue for X -->
        <v-chip
          v-if="tier"
          :class="tier.effective === 'aim_pro' ? 'chips-orange' : 'chips-blue'"
          size="small"
          label
        >
          {{ tier.effective === 'aim_pro' ? 'AIM Pro' : 'AIM X' }}
        </v-chip>

        <!-- Edit answers — hidden once approval is terminal -->
        <v-btn
          v-if="!isComplete"
          variant="outlined"
          size="small"
          prepend-icon="mdi-pencil-outline"
          @click="$emit('edit', undefined)"
        >
          Edit answers
        </v-btn>
      </div>
    </div>

    <!-- ── Generated-from line ───────────────────────────────────────────── -->
    <p class="output-screen__gen-line">
      <v-icon size="16" class="mr-1">mdi-creation-outline</v-icon>
      Generated from your 8-step setup
    </p>

    <!-- ── Validation summary ───────────────────────────────────────────── -->
    <v-alert
      v-if="!isComplete && failingSteps.length > 0"
      data-testid="validation-summary"
      variant="tonal"
      color="error"
      :icon="false"
      class="output-screen__val-summary mb-4"
    >
      <template #prepend>
        <v-icon color="error">mdi-alert-circle-outline</v-icon>
      </template>

      <div class="output-screen__val-header">
        {{ failingSteps.length === 1
          ? 'One step needs attention before you can approve'
          : `${failingSteps.length} steps need attention before you can approve` }}
      </div>

      <div
        v-for="row in failingSteps"
        :key="row.step"
        data-testid="validation-row"
        class="output-screen__val-row"
      >
        <div class="output-screen__val-text">
          <strong>{{ row.stepLabel }}:</strong>
          <span class="ml-1">{{ row.fieldMessage }}</span>
        </div>
        <v-btn
          :data-testid="`fix-btn-${row.step}`"
          variant="outlined"
          size="x-small"
          color="error"
          @click="onFixClick(row.step)"
        >
          Go back to fix →
        </v-btn>
      </div>
    </v-alert>

    <!-- ── Output v-tabs (plain underline, no numbered circles) ──────────── -->
    <v-tabs
      v-model="activeTab"
      color="primary"
      class="output-screen__tabs"
    >
      <v-tab value="sow">Scope of Work</v-tab>
      <v-tab value="timeline">Timeline</v-tab>
      <v-tab value="schema">Data Schema</v-tab>
    </v-tabs>

    <v-window v-model="activeTab" class="output-screen__window mt-4">
      <v-window-item value="sow">
        <SowTab />
      </v-window-item>
      <v-window-item value="timeline">
        <TimelineTab />
      </v-window-item>
      <v-window-item value="schema">
        <DataSchemaTab />
      </v-window-item>
    </v-window>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { useAimOnboarding } from '../composables/useAimOnboarding'
import { validateStep } from '../logic/validation'
import SowTab from './SowTab.vue'
import TimelineTab from './TimelineTab.vue'
import DataSchemaTab from './DataSchemaTab.vue'

// ── Emits ─────────────────────────────────────────────────────────────────────
// edit(step?) — parent (F1) toggles back to wizard and optionally focuses a step
defineEmits<{
  edit: [step: number | undefined]
}>()

// ── Composable ────────────────────────────────────────────────────────────────

const { wizard, stepStatus, visitedSteps } = useAimOnboarding()

// ── Wizard fields ─────────────────────────────────────────────────────────────

const appName = computed(() => wizard.value.appName || wizard.value.companyName || '—')
const companyName = computed(() => wizard.value.companyName)
const region = computed(() => wizard.value.region)
const tier = computed(() => wizard.value.tier)

// ── Terminal state ────────────────────────────────────────────────────────────
// Approval is terminal (§8a). Once approvedAt is set there is no unlock.

const isComplete = computed(() => wizard.value.approvedAt != null)

// ── Tabs ──────────────────────────────────────────────────────────────────────

const activeTab = ref<'sow' | 'timeline' | 'schema'>('sow')

// ── Validation summary (§8a) ──────────────────────────────────────────────────
// Required steps: 1–5, 8. Optional: 6, 7 (never block).
// One row per failing required step, first invalid field only.

const REQUIRED_STEPS = [1, 2, 3, 4, 5, 8] as const

const STEP_LABELS: Record<number, string> = {
  1: 'Step 1 — Your Team',
  2: 'Step 2 — About Your Product',
  3: 'Step 3 — Marketing Setup',
  4: 'Step 4 — Business & Funnel',
  5: 'Step 5 — Data Sources',
  8: 'Step 8 — Review',
}

const FIELD_LABELS: Record<string, string> = {
  companyName: 'Company name is required',
  projectLeads: 'At least one project lead (name + email) is required',
  dataLeads: 'At least one data lead is required',
  appName: 'App / brand name is required',
  platforms: 'Select at least one platform (iOS, Android, or Web)',
  region: 'Primary market is required',
  budgetMonthly: 'Total paid media budget is required',
  budgetAnnual: 'Total paid media budget is required',
  digitalMediaTypes: 'Select at least one digital media type',
  hasAttrGaps: 'Attribution data reliability answer is required',
  coverageConfidence: 'Coverage confidence answer is required',
  ua: 'Select at least one campaign type (UA, UE, or Brand)',
  business: 'Business model is required',
  'funnel.kpi': 'A KPI event must be selected in your funnel',
  mmp: 'MMP selection is required for mobile platforms',
  spendCollection: 'Ad spend data collection method is required',
  adSpendSources: 'At least one ad spend source is required',
  history: 'Data history selection is required for each platform',
}

function fieldLabel(key: string): string {
  return (
    FIELD_LABELS[key] ??
    `${key.replace(/([A-Z])/g, ' $1').toLowerCase()} is required`
  )
}

interface FailingStepRow {
  step: number
  stepLabel: string
  fieldMessage: string
}

const failingSteps = computed<FailingStepRow[]>(() => {
  if (isComplete.value) return []
  const rows: FailingStepRow[] = []
  for (const step of REQUIRED_STEPS) {
    if (stepStatus.value[step] !== 'done') {
      const { invalidFields } = validateStep(String(step), wizard.value)
      const firstField = invalidFields[0] ?? ''
      rows.push({
        step,
        stepLabel: STEP_LABELS[step],
        fieldMessage: fieldLabel(firstField),
      })
    }
  }
  return rows
})

// ── Deep-link handler ─────────────────────────────────────────────────────────
// Marks the step visited (so F3 stepStatus can show error in the rail)
// then emits edit(step) so F1 can toggle back to the wizard.

const emit = defineEmits<{ edit: [step: number | undefined] }>()

function onFixClick(step: number): void {
  visitedSteps.value.add(step)
  emit('edit', step)
}
</script>

<style scoped>
.output-screen {
  max-width: 880px;
  margin: 0 auto;
  padding: 4px;
}

.output-screen__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 12px;
  margin-bottom: 4px;
}

.output-screen__identity {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.output-screen__app-name {
  font-size: 19px;
  font-weight: 700;
  color: rgb(var(--v-theme-on-surface));
}

.output-screen__meta {
  font-size: 12.5px;
  color: rgb(var(--v-theme-on-surface-variant));
}

.output-screen__header-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

.output-screen__gen-line {
  display: flex;
  align-items: center;
  font-size: 11.5px;
  color: rgb(var(--v-theme-on-surface-variant));
  margin: 0 0 16px;
}

.output-screen__val-summary {
  border-radius: 8px;
}

.output-screen__val-header {
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 8px;
}

.output-screen__val-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  font-size: 12.5px;
  padding: 8px 0;
  border-top: 1px solid rgba(var(--v-theme-error), 0.12);
}

.output-screen__val-row:first-of-type {
  border-top: none;
}

.output-screen__val-text {
  flex: 1;
  min-width: 0;
}

.output-screen__tabs {
  border-bottom: 1px solid rgb(var(--v-theme-outline-variant));
}
</style>
```

**Important:** The `<script setup>` block calls `defineEmits` twice. Consolidate: keep only the second `defineEmits` call (the one used by `onFixClick`). Remove the first `defineEmits` from the `// ── Emits ──` section — you cannot call `defineEmits` more than once. The template's `$emit('edit', undefined)` and the `onFixClick` call both use the same `emit` constant.

Fix before Task 3 — replace the template `$emit` inline call with `emit('edit', undefined)` through a handler function, or wire via a method. Complete corrected `<script setup>`:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useAimOnboarding } from '../composables/useAimOnboarding'
import { validateStep } from '../logic/validation'
import SowTab from './SowTab.vue'
import TimelineTab from './TimelineTab.vue'
import DataSchemaTab from './DataSchemaTab.vue'

const emit = defineEmits<{
  edit: [step: number | undefined]
}>()

const { wizard, stepStatus, visitedSteps } = useAimOnboarding()

const appName = computed(() => wizard.value.appName || wizard.value.companyName || '—')
const companyName = computed(() => wizard.value.companyName)
const region = computed(() => wizard.value.region)
const tier = computed(() => wizard.value.tier)

const isComplete = computed(() => wizard.value.approvedAt != null)

const activeTab = ref<'sow' | 'timeline' | 'schema'>('sow')

const REQUIRED_STEPS = [1, 2, 3, 4, 5, 8] as const

const STEP_LABELS: Record<number, string> = {
  1: 'Step 1 — Your Team',
  2: 'Step 2 — About Your Product',
  3: 'Step 3 — Marketing Setup',
  4: 'Step 4 — Business & Funnel',
  5: 'Step 5 — Data Sources',
  8: 'Step 8 — Review',
}

const FIELD_LABELS: Record<string, string> = {
  companyName: 'Company name is required',
  projectLeads: 'At least one project lead (name + email) is required',
  dataLeads: 'At least one data lead is required',
  appName: 'App / brand name is required',
  platforms: 'Select at least one platform (iOS, Android, or Web)',
  region: 'Primary market is required',
  budgetMonthly: 'Total paid media budget is required',
  budgetAnnual: 'Total paid media budget is required',
  digitalMediaTypes: 'Select at least one digital media type',
  hasAttrGaps: 'Attribution data reliability answer is required',
  coverageConfidence: 'Coverage confidence answer is required',
  ua: 'Select at least one campaign type (UA, UE, or Brand)',
  business: 'Business model is required',
  'funnel.kpi': 'A KPI event must be selected in your funnel',
  mmp: 'MMP selection is required for mobile platforms',
  spendCollection: 'Ad spend data collection method is required',
  adSpendSources: 'At least one ad spend source is required',
  history: 'Data history selection is required for each platform',
}

function fieldLabel(key: string): string {
  return (
    FIELD_LABELS[key] ??
    `${key.replace(/([A-Z])/g, ' $1').toLowerCase()} is required`
  )
}

interface FailingStepRow {
  step: number
  stepLabel: string
  fieldMessage: string
}

const failingSteps = computed<FailingStepRow[]>(() => {
  if (isComplete.value) return []
  const rows: FailingStepRow[] = []
  for (const step of REQUIRED_STEPS) {
    if (stepStatus.value[step] !== 'done') {
      const { invalidFields } = validateStep(String(step), wizard.value)
      const firstField = invalidFields[0] ?? ''
      rows.push({
        step,
        stepLabel: STEP_LABELS[step],
        fieldMessage: fieldLabel(firstField),
      })
    }
  }
  return rows
})

function onEditAnswers(): void {
  emit('edit', undefined)
}

function onFixClick(step: number): void {
  visitedSteps.value.add(step)
  emit('edit', step)
}
</script>
```

And update the template's Edit-answers button to use `onEditAnswers`:

```html
<v-btn
  v-if="!isComplete"
  variant="outlined"
  size="small"
  prepend-icon="mdi-pencil-outline"
  @click="onEditAnswers"
>
  Edit answers
</v-btn>
```

---

### Task 3 — Run tests (expect pass)

- [ ] **3.1** Run the output spec:

```bash
cd packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/OutputScreen.spec.ts 2>&1 | tail -40
```

All tests must pass. Common failures and fixes:

- `Cannot find module '../SowTab.vue'` — create empty stub files:

```bash
mkdir -p packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output
touch packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue
touch packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue
touch packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.vue
```

  Each stub should contain at minimum:

```vue
<template><div /></template>
```

- `Cannot find module '../../composables/useAimOnboarding'` — F3 must be merged. If not yet, create a minimal stub matching the mock interface in the spec.

- `v-tabs` aria issue: Vuetify `v-tab` renders `role="tab"` automatically. If `wrapper.findAll('[role="tab"]')` is empty, check that the Vuetify plugin is registered (`createVuetify({ components, directives })`) and passed in `global.plugins`.

- [ ] **3.2** Run the full advertiser test suite for regressions:

```bash
cd packages/advertiser
npx vitest run 2>&1 | tail -20
```

- [ ] **3.3** Type-check:

```bash
cd packages/advertiser
npx tsc --noEmit 2>&1 | tail -20
```

---

### Task 4 — Lint

- [ ] **4.1** Run markdownlint on this plan file (one-time sanity check):

```bash
npx markdownlint-cli2 "packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/OutputScreen.spec.ts" 2>&1 | tail -10
```

- [ ] **4.2** Run ESLint on the new files:

```bash
cd /path/to/frontend-mos
pnpm --filter @mos/advertiser lint -- --fix-dry-run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/OutputScreen.vue src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/OutputScreen.spec.ts 2>&1 | tail -20
```

Fix any reported issues before committing.

---

### Task 5 — Commit

- [ ] **5.1**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/OutputScreen.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/OutputScreen.spec.ts

git commit -m "feat(onboarding): add OutputScreen — tier header, validation summary deep-link, v-tabs shell (O7)"
```

---

## Spec Coverage Check

| Requirement | Task |
|-------------|------|
| Tier header: appName, tier chip (Pro/X), Edit answers | Task 2, tests in Task 1 tier-header suite |
| AIM Pro chip orange / AIM X chip blue (`chips-orange`/`chips-blue`) | Task 2 `v-chip :class` |
| "Generated from your 8-step setup" line | Task 2 gen-line paragraph, test 3 |
| Validation summary visible when `!allRequiredDone` | Task 1 validation-summary suite |
| Validation summary hidden when `allRequiredDone` | Task 1 "is hidden when allRequiredDone is true" |
| One row per failing required step | Task 1 "shows one row per failing required step" |
| First invalid field message per row | Task 1 "shows first invalid field message" |
| Optional steps (6, 7) never produce a row | Task 1 "optional steps never produce a validation row" |
| "Go back to fix →" emits `edit(step)` | Task 1 deep-link suite |
| "Go back to fix →" marks step visited | Task 1 "marks the target step visited" |
| `v-tabs` plain underline (no numbered circles) | Task 1 "does NOT render numbered circles" |
| Default tab: Scope of Work | Task 1 "defaults to Scope of Work tab active" |
| Tab switching: Timeline, Data Schema | Task 1 tab-switching tests |
| `isComplete` hides validation summary | Task 1 "is hidden when isComplete" |
| `isComplete` hides Edit answers button | Task 1 "hides Edit answers button when complete" |
| Approval is terminal — no Unlock affordance | No Unlock in component, no test for it |
| Colors via `rgb(var(--v-theme-*))` not hardcoded hex | Task 2 style block |
| Real Vuetify components (`v-alert`, `v-tabs`, `v-window`, `v-btn`, `v-chip`) | Task 2 template |
