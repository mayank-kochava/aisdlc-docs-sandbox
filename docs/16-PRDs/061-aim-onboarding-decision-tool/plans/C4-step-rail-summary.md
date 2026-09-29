---
id: plan-c4
title: "C4 — StepRail + SummaryPanel"
---

## C4 — StepRail + SummaryPanel

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development`
> (recommended) or `superpowers:executing-plans` to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build two pure-presentational Vue 3 / Vuetify 3 components — a custom vertical
`StepRail` and a collapsible `ModelConfigSummary` — for the AIM Onboarding wizard in
`frontend-mos`.

**Architecture:** `StepRail.vue` is a fully custom component (native `v-stepper` vertical
interleaves content, making it incompatible with the rail-only layout). It renders
focusable `<button>` elements per step with `aria-current="step"` on the active item.
`ModelConfigSummary.vue` wraps content in a single `v-expansion-panels` panel so Vuetify
renders a real `<button aria-expanded>` at no extra cost. Both components are pure
props-in / emits-out; no store access.

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript, Vitest + `@vue/test-utils` (jsdom),
`@mdi/font` icons.

---

## File map

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/StepRail.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ModelConfigSummary.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/StepRail.spec.ts` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ModelConfigSummary.spec.ts` |

All paths are relative to `/Users/mukey/Documents/kochava-projects/k4a/frontend-mos/`.

---

## Token reference (from `packages/app/src/plugins/vuetify.ts`)

Use `rgb(var(--v-theme-TOKEN))` in CSS — never hardcode hex.

| Semantic role | Token | Light hex |
|---|---|---|
| Active / primary | `primary` | `#1E4B97` |
| Done / success | `success` | `#427900` |
| Error | `error` | `#BE202E` |
| Warning / Pro tier | `warning` | `#BA4E00` |
| Muted text | `grey-2` | `#6D757B` |
| Border | `border` | `#E1E1E1` |
| AIM X chip text | `chips-blue` | `#095ABD` |
| AIM X chip fill | `chips-blue_fill` | `#0C72EE` |
| AIM Pro chip text | `chips-orange` | `#BA4E00` |
| AIM Pro chip fill | `chips-orange_fill` | `#D56428` |

---

## Shared types

Both components reference a `StepState` type. Define it inline in each component
(no shared file needed at this stage — YAGNI).

```typescript
type StepState = 'pending' | 'active' | 'done' | 'error'
```

---

## Task 1: `StepRail.vue` — failing tests first

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/StepRail.spec.ts`

### Context for the test

`StepRail` receives an array of step descriptors and the index of the current step.
It renders one focusable `<button>` per step. The active button carries `aria-current="step"`.
Done steps show `mdi-check` inside the number badge. Error steps show `mdi-alert-circle`
inside the number badge AND red text (not color alone — the icon is the non-color signal).
Below a `<hr>` divider lives an "Output" step (index 9, always `pending` state from the
rail's perspective). A "Save progress" `<button>` is pinned at the bottom.

- [ ] **Step 1.1: Create the test file**

```bash
mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__
```

- [ ] **Step 1.2: Write the failing tests**

Create `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/StepRail.spec.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import StepRail from '../StepRail.vue'

// tests.config.ts pre-registers createVuetify() + pinia as global plugins.
// StepRail is pure-presentational so no store mocking is needed.

const steps = [
  { label: 'Your Team',         icon: 'mdi-account-multiple-outline' },
  { label: 'About your product', icon: 'mdi-cellphone' },
  { label: 'Marketing setup',   icon: 'mdi-bullhorn-outline' },
]

function mountRail(props: Record<string, unknown>) {
  return mount(StepRail, { props })
}

describe('StepRail', () => {
  describe('aria-current', () => {
    it('sets aria-current="step" on the active step button', () => {
      const wrapper = mountRail({ steps, activeStep: 1 })
      const buttons = wrapper.findAll('button.step-rail__item')
      expect(buttons[1].attributes('aria-current')).toBe('step')
    })

    it('does not set aria-current on inactive step buttons', () => {
      const wrapper = mountRail({ steps, activeStep: 1 })
      const buttons = wrapper.findAll('button.step-rail__item')
      expect(buttons[0].attributes('aria-current')).toBeUndefined()
      expect(buttons[2].attributes('aria-current')).toBeUndefined()
    })
  })

  describe('done state', () => {
    it('renders mdi-check icon in the badge for a done step', () => {
      const wrapper = mountRail({ steps, activeStep: 2, doneSteps: [0, 1] })
      const doneButton = wrapper.findAll('button.step-rail__item')[0]
      expect(doneButton.find('.step-rail__badge .mdi-check').exists()).toBe(true)
    })

    it('does not render mdi-check for a pending step', () => {
      const wrapper = mountRail({ steps, activeStep: 0 })
      const pendingButton = wrapper.findAll('button.step-rail__item')[2]
      expect(pendingButton.find('.mdi-check').exists()).toBe(false)
    })
  })

  describe('error state', () => {
    it('renders mdi-alert-circle icon in the badge for an error step', () => {
      const wrapper = mountRail({ steps, activeStep: 1, errorSteps: [0] })
      const errButton = wrapper.findAll('button.step-rail__item')[0]
      expect(errButton.find('.step-rail__badge .mdi-alert-circle').exists()).toBe(true)
    })

    it('adds error class to the step item for an error step', () => {
      const wrapper = mountRail({ steps, activeStep: 1, errorSteps: [0] })
      const errButton = wrapper.findAll('button.step-rail__item')[0]
      expect(errButton.classes()).toContain('step-rail__item--error')
    })
  })

  describe('Output step and divider', () => {
    it('renders a divider element before the output step', () => {
      const wrapper = mountRail({ steps, activeStep: 0 })
      expect(wrapper.find('.step-rail__divider').exists()).toBe(true)
    })

    it('renders the Output step with number 9', () => {
      const wrapper = mountRail({ steps, activeStep: 0 })
      const outputStep = wrapper.find('.step-rail__output')
      expect(outputStep.exists()).toBe(true)
      expect(outputStep.text()).toContain('Output')
      expect(outputStep.text()).toContain('9')
    })
  })

  describe('Save progress button', () => {
    it('renders a Save progress button', () => {
      const wrapper = mountRail({ steps, activeStep: 0 })
      const saveBtn = wrapper.find('.step-rail__save button')
      expect(saveBtn.exists()).toBe(true)
      expect(saveBtn.text()).toContain('Save progress')
    })

    it('emits save event when Save progress is clicked', async () => {
      const wrapper = mountRail({ steps, activeStep: 0 })
      await wrapper.find('.step-rail__save button').trigger('click')
      expect(wrapper.emitted('save')).toHaveLength(1)
    })
  })

  describe('step navigation', () => {
    it('emits navigate with the step index when a step button is clicked', async () => {
      const wrapper = mountRail({ steps, activeStep: 0 })
      await wrapper.findAll('button.step-rail__item')[2].trigger('click')
      expect(wrapper.emitted('navigate')?.[0]).toEqual([2])
    })
  })
})
```

- [ ] **Step 1.3: Run the tests — expect them to fail (StepRail.vue does not exist)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/StepRail.spec.ts 2>&1 | tail -20
```

Expected output: `FAIL` — "Cannot find module '../StepRail.vue'"

---

## Task 2: `StepRail.vue` — implement until tests pass

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/StepRail.vue`

- [ ] **Step 2.1: Create the component**

```vue
<template>
  <nav class="step-rail" aria-label="Wizard steps">
    <!-- Step buttons 1–N -->
    <button
      v-for="(step, index) in steps"
      :key="index"
      class="step-rail__item"
      :class="{
        'step-rail__item--active': index === activeStep,
        'step-rail__item--done':   doneSteps.includes(index),
        'step-rail__item--error':  errorSteps.includes(index),
      }"
      :aria-current="index === activeStep ? 'step' : undefined"
      type="button"
      @click="$emit('navigate', index)"
    >
      <v-icon class="step-rail__icon" size="17">{{ step.icon }}</v-icon>

      <span class="step-rail__label">{{ step.label }}</span>

      <span class="step-rail__badge" :aria-hidden="true">
        <!-- Error: icon is the non-color signal (§8b WCAG) -->
        <v-icon
          v-if="errorSteps.includes(index)"
          class="mdi-alert-circle"
          size="12"
        >mdi-alert-circle</v-icon>
        <!-- Done -->
        <v-icon
          v-else-if="doneSteps.includes(index)"
          class="mdi-check"
          size="12"
        >mdi-check</v-icon>
        <!-- Pending / active: show number -->
        <span v-else>{{ index + 1 }}</span>
      </span>
    </button>

    <!-- Divider before Output -->
    <hr class="step-rail__divider" />

    <!-- Output step (step 9, always navigable but visually distinct) -->
    <div class="step-rail__output step-rail__item">
      <v-icon class="step-rail__icon" size="17">mdi-file-document-outline</v-icon>
      <span class="step-rail__label">Output</span>
      <span class="step-rail__badge" aria-hidden="true">9</span>
    </div>

    <!-- Save progress -->
    <div class="step-rail__save">
      <button type="button" @click="$emit('save')">
        <v-icon size="16">mdi-content-save-outline</v-icon>
        Save progress
      </button>
    </div>
  </nav>
</template>

<script setup lang="ts">
export interface StepDescriptor {
  label: string
  icon: string
}

const props = withDefaults(
  defineProps<{
    steps: StepDescriptor[]
    activeStep: number
    doneSteps?: number[]
    errorSteps?: number[]
  }>(),
  {
    doneSteps: () => [],
    errorSteps: () => [],
  }
)

defineEmits<{
  navigate: [index: number]
  save: []
}>()
</script>

<style scoped>
.step-rail {
  display: flex;
  flex-direction: column;
  width: 218px;
}

/* ── Step items ── */
.step-rail__item {
  display: flex;
  align-items: center;
  gap: 11px;
  padding: 9px 10px;
  border-radius: 8px;
  cursor: pointer;
  color: rgb(var(--v-theme-grey-2));
  font-size: 13.5px;
  font-family: inherit;
  background: transparent;
  border: none;
  width: 100%;
  text-align: left;
}
.step-rail__item:hover {
  background: rgb(var(--v-theme-surface-4));
}

/* Active */
.step-rail__item--active {
  background: rgba(var(--v-theme-primary), 0.07);
  color: rgb(var(--v-theme-primary));
  font-weight: 600;
}

/* Done: text stays grey-2, badge goes success */
.step-rail__item--done {
  color: rgb(var(--v-theme-grey-2));
}

/* Error: text + icon go to error color; icon is the non-color signal */
.step-rail__item--error {
  color: rgb(var(--v-theme-error));
}

.step-rail__icon {
  width: 18px;
  text-align: center;
  opacity: 0.75;
  flex-shrink: 0;
}
.step-rail__item--active .step-rail__icon,
.step-rail__item--error .step-rail__icon {
  opacity: 1;
}

.step-rail__label {
  flex: 1;
  line-height: 1.3;
}

/* Badge: numbered circle */
.step-rail__badge {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: rgb(var(--v-theme-surface-4));
  border: 1px solid rgb(var(--v-theme-border));
  color: rgb(var(--v-theme-grey-2));
  display: grid;
  place-items: center;
  font-size: 11px;
  font-weight: 600;
  flex-shrink: 0;
}
.step-rail__item--active .step-rail__badge {
  background: rgb(var(--v-theme-primary));
  color: #fff;
  border-color: rgb(var(--v-theme-primary));
}
.step-rail__item--done .step-rail__badge {
  background: rgb(var(--v-theme-success));
  color: #fff;
  border-color: rgb(var(--v-theme-success));
}
.step-rail__item--error .step-rail__badge {
  background: rgb(var(--v-theme-error));
  color: #fff;
  border-color: rgb(var(--v-theme-error));
}

/* ── Divider ── */
.step-rail__divider {
  height: 1px;
  background: rgb(var(--v-theme-border));
  border: none;
  margin: 8px 10px;
}

/* ── Save progress ── */
.step-rail__save {
  margin: 14px 10px 0;
}
.step-rail__save button {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  width: 100%;
  border: 1px solid rgb(var(--v-theme-border));
  background: rgb(var(--v-theme-surface-2));
  border-radius: 8px;
  padding: 9px;
  font-size: 13px;
  color: rgb(var(--v-theme-grey-2));
  cursor: pointer;
  font-family: inherit;
}
.step-rail__save button:hover {
  background: rgb(var(--v-theme-surface-4));
}
</style>
```

- [ ] **Step 2.2: Run the tests — expect them to pass**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/StepRail.spec.ts 2>&1 | tail -20
```

Expected output: all tests `PASS`.

- [ ] **Step 2.3: Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/StepRail.vue packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/StepRail.spec.ts && git commit -m "feat(aim-onboarding): add StepRail component with a11y aria-current and TDD tests"
```

---

## Task 3: `ModelConfigSummary.vue` — failing tests first

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ModelConfigSummary.spec.ts`

### Context for the test

`ModelConfigSummary` wraps its content in a single `v-expansion-panels` panel. Vuetify
renders the panel title as a real `<button aria-expanded="true|false">`, so no custom
ARIA wiring is needed. The component receives:

- `modelCount: number` — shown as a large numeral
- `tier: 'aim_x' | 'aim_pro' | null` — `null` means "Estimating…" (pre-Step 3)
- `tierDrivers: string[]` — bullet reasons shown when tier is set

When `tier` is `null`, the tier row shows the italic string "Estimating…". When `tier`
is `'aim_pro'`, a chip with class `chips-orange` is rendered. When `tier` is `'aim_x'`,
a chip with class `chips-blue` is rendered.

- [ ] **Step 3.1: Write the failing tests**

Create `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ModelConfigSummary.spec.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import { nextTick } from 'vue'
import ModelConfigSummary from '../ModelConfigSummary.vue'

// tests.config.ts pre-registers createVuetify() as a global plugin.

function mountSummary(props: Record<string, unknown>) {
  return mount(ModelConfigSummary, { props, attachTo: document.body })
}

describe('ModelConfigSummary', () => {
  describe('model count', () => {
    it('displays the model count', () => {
      const wrapper = mountSummary({ modelCount: 2, tier: null, tierDrivers: [] })
      expect(wrapper.find('.model-summary__count').text()).toContain('2')
    })
  })

  describe('tier — estimating state', () => {
    it('shows Estimating… when tier is null', () => {
      const wrapper = mountSummary({ modelCount: 1, tier: null, tierDrivers: [] })
      expect(wrapper.find('.model-summary__tier-row').text()).toContain('Estimating')
    })

    it('does not render a tier chip when tier is null', () => {
      const wrapper = mountSummary({ modelCount: 1, tier: null, tierDrivers: [] })
      expect(wrapper.find('.v-chip').exists()).toBe(false)
    })
  })

  describe('tier — aim_pro', () => {
    it('renders an AIM Pro chip with chips-orange color', () => {
      const wrapper = mountSummary({
        modelCount: 1,
        tier: 'aim_pro',
        tierDrivers: ['Budget $250K/mo'],
      })
      const chip = wrapper.find('.v-chip')
      expect(chip.exists()).toBe(true)
      // Vuetify applies color as a class like "text-chips-orange" or via style
      expect(chip.attributes('class')).toContain('chips-orange')
    })

    it('shows tier drivers when tier is set', () => {
      const wrapper = mountSummary({
        modelCount: 1,
        tier: 'aim_pro',
        tierDrivers: ['Budget qualifies for Pro'],
      })
      expect(wrapper.find('.model-summary__drivers').text()).toContain('Budget qualifies for Pro')
    })
  })

  describe('tier — aim_x', () => {
    it('renders an AIM X chip with chips-blue color', () => {
      const wrapper = mountSummary({ modelCount: 1, tier: 'aim_x', tierDrivers: [] })
      const chip = wrapper.find('.v-chip')
      expect(chip.exists()).toBe(true)
      expect(chip.attributes('class')).toContain('chips-blue')
    })
  })

  describe('collapse (aria-expanded)', () => {
    it('renders a button with aria-expanded attribute (Vuetify v-expansion-panel-title)', () => {
      const wrapper = mountSummary({ modelCount: 1, tier: null, tierDrivers: [] })
      // Vuetify renders v-expansion-panel-title as a <button aria-expanded="...">
      const toggleBtn = wrapper.find('button[aria-expanded]')
      expect(toggleBtn.exists()).toBe(true)
    })

    it('toggles aria-expanded when the panel title is clicked', async () => {
      const wrapper = mountSummary({ modelCount: 1, tier: null, tierDrivers: [] })
      const toggleBtn = wrapper.find('button[aria-expanded]')
      const initialExpanded = toggleBtn.attributes('aria-expanded')
      await toggleBtn.trigger('click')
      await nextTick()
      const afterExpanded = wrapper.find('button[aria-expanded]').attributes('aria-expanded')
      expect(afterExpanded).not.toBe(initialExpanded)
    })
  })
})
```

- [ ] **Step 3.2: Run the tests — expect them to fail (ModelConfigSummary.vue does not exist)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ModelConfigSummary.spec.ts 2>&1 | tail -20
```

Expected output: `FAIL` — "Cannot find module '../ModelConfigSummary.vue'"

---

## Task 4: `ModelConfigSummary.vue` — implement until tests pass

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ModelConfigSummary.vue`

- [ ] **Step 4.1: Create the component**

```vue
<template>
  <div class="model-summary">
    <v-expansion-panels v-model="openPanel">
      <v-expansion-panel value="open">
        <!-- Panel title = real <button aria-expanded> from Vuetify -->
        <v-expansion-panel-title class="model-summary__header">
          <span class="model-summary__title">Model configuration</span>
        </v-expansion-panel-title>

        <v-expansion-panel-text>
          <!-- Model count -->
          <div class="model-summary__count">
            <span class="model-summary__count-num">{{ modelCount }}</span>
            <span class="model-summary__count-label">
              {{ modelCount === 1 ? 'model estimated' : 'models estimated' }}
            </span>
          </div>

          <!-- Tier row -->
          <div class="model-summary__tier-row">
            <span class="model-summary__tier-label">Recommended tier</span>
            <!-- Estimating state (pre-Step 3) -->
            <span v-if="tier === null" class="model-summary__tier-est">
              Estimating&hellip;
            </span>
            <!-- AIM Pro chip -->
            <v-chip
              v-else-if="tier === 'aim_pro'"
              size="small"
              variant="tonal"
              color="chips-orange"
              class="chips-orange"
            >
              AIM Pro
            </v-chip>
            <!-- AIM X chip -->
            <v-chip
              v-else
              size="small"
              variant="tonal"
              color="chips-blue"
              class="chips-blue"
            >
              AIM X
            </v-chip>
          </div>

          <!-- Tier drivers (shown once tier is set) -->
          <ul v-if="tier !== null && tierDrivers.length" class="model-summary__drivers">
            <li v-for="(driver, i) in tierDrivers" :key="i">
              <v-icon size="14" color="success">mdi-arrow-up</v-icon>
              {{ driver }}
            </li>
          </ul>

          <!-- Placeholder hint (estimating state) -->
          <p v-if="tier === null" class="model-summary__hint">
            Your tier firms up once budget &amp; campaigns are set (Step&nbsp;3).
          </p>
        </v-expansion-panel-text>
      </v-expansion-panel>
    </v-expansion-panels>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

defineProps<{
  modelCount: number
  tier: 'aim_x' | 'aim_pro' | null
  tierDrivers: string[]
}>()

// Start expanded; user can collapse.
const openPanel = ref<string | undefined>('open')
</script>

<style scoped>
.model-summary {
  position: sticky;
  top: 0;
}

/* Override expansion-panel chrome to match mock */
:deep(.v-expansion-panel) {
  border-radius: 12px !important;
  box-shadow: 0 8px 24px rgba(16, 24, 40, 0.07);
  overflow: hidden;
}

:deep(.v-expansion-panel-title) {
  font-size: 13px;
  font-weight: 600;
  min-height: 48px;
  padding: 12px 18px;
}

:deep(.v-expansion-panel-text__wrapper) {
  padding: 0;
}

.model-summary__title {
  font-size: 13px;
  font-weight: 600;
}

/* Count block */
.model-summary__count {
  display: flex;
  align-items: baseline;
  gap: 8px;
  padding: 16px;
  border-bottom: 1px solid rgb(var(--v-theme-border));
}
.model-summary__count-num {
  font-size: 30px;
  font-weight: 700;
  color: rgb(var(--v-theme-primary));
  line-height: 1;
}
.model-summary__count-label {
  font-size: 12.5px;
  color: rgb(var(--v-theme-grey-2));
}

/* Tier row */
.model-summary__tier-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 13px 16px;
}
.model-summary__tier-label {
  font-size: 11px;
  color: rgb(var(--v-theme-grey-2));
}
.model-summary__tier-est {
  font-size: 12px;
  color: rgb(var(--v-theme-grey-2));
  font-style: italic;
}

/* Drivers */
.model-summary__drivers {
  list-style: none;
  padding: 0 16px 14px;
  margin: 0;
}
.model-summary__drivers li {
  display: flex;
  gap: 7px;
  font-size: 12px;
  color: rgb(var(--v-theme-grey-2));
  margin: 7px 0;
}

/* Hint */
.model-summary__hint {
  padding: 0 16px 14px;
  font-size: 12px;
  color: rgb(var(--v-theme-grey-2));
  margin: 0;
}
</style>
```

- [ ] **Step 4.2: Run the tests — expect them to pass**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ModelConfigSummary.spec.ts 2>&1 | tail -20
```

Expected output: all tests `PASS`.

- [ ] **Step 4.3: Run all C4 tests together to confirm no regressions**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ 2>&1 | tail -30
```

Expected output: all tests `PASS` — 0 failures.

- [ ] **Step 4.4: Run the full advertiser test suite to confirm no regressions**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run 2>&1 | tail -30
```

Expected output: no new failures (pre-existing skipped tests are fine).

- [ ] **Step 4.5: Lint**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx eslint src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ModelConfigSummary.vue src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ModelConfigSummary.spec.ts 2>&1 | tail -20
```

Expected output: no errors. Fix any warnings before the commit.

- [ ] **Step 4.6: Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ModelConfigSummary.vue packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ModelConfigSummary.spec.ts && git commit -m "feat(aim-onboarding): add ModelConfigSummary with v-expansion-panels collapse and tier chip"
```

---

## Self-review checklist

**Spec coverage:**

| Requirement | Task |
|---|---|
| Custom vertical rail (no native v-stepper) | Task 2 |
| Icon + label + numbered circle per step | Task 2 |
| Done = check icon + green badge | Tasks 1 + 2 |
| Active = primary highlight | Tasks 1 + 2 |
| Error = red text + `mdi-alert-circle` icon (not color alone) | Tasks 1 + 2 |
| "Output" shown as step 9 below a divider | Tasks 1 + 2 |
| "Save progress" button pinned; emits `save` | Tasks 1 + 2 |
| `aria-current="step"` on active step button | Tasks 1 + 2 |
| Rail steps are focusable (real `<button>` elements) | Task 2 |
| Collapsible right panel via `v-expansion-panels` | Tasks 3 + 4 |
| Real `<button aria-expanded>` from Vuetify | Tasks 3 + 4 |
| Model count display | Tasks 3 + 4 |
| "Estimating…" until tier is set | Tasks 3 + 4 |
| Tier chip: `chips-orange` (Pro) / `chips-blue` (X) | Tasks 3 + 4 |
| Tier drivers list | Tasks 3 + 4 |
| All colors via `rgb(var(--v-theme-*))` | Tasks 2 + 4 |
| AA-compliant tokens (`success=#427900`, `error=#BE202E`) | Tasks 2 + 4 |

**Placeholder scan:** No TBD / TODO / "similar to" language present.

**Type consistency:** `StepDescriptor`, `activeStep`, `doneSteps`, `errorSteps` used
consistently across spec and component. `tier: 'aim_x' | 'aim_pro' | null` matches in
both spec and component.
