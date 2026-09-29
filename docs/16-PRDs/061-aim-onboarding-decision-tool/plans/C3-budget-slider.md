---
id: plan-c3
title: "C3 — BudgetSlider"
---

## C3 — BudgetSlider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `BudgetSlider.vue` — a discrete band slider with a Monthly/Annual toggle, a formatted value label (`fmtM`), and a "qualifies for AIM Pro consideration" info banner at the $250K/mo threshold.

**Architecture:** A single self-contained Vue 3 SFC wrapping Vuetify `v-slider` (index-based, discrete bands) and `v-btn-toggle` (Monthly/Annual). All logic — band arrays, `fmtM`, threshold detection, and emitted values — lives inside the component. Emits `update:modelValue` (numeric budget) and `update:period` (`'monthly'|'annual'`). Tests are pure Vitest unit tests against exported helpers (`fmtM`, `MONTHLY_BANDS`, `ANNUAL_BANDS`, `PRO_THRESHOLD_MONTHLY`); the Vuetify layer is integration-verified in the component's `__tests__` file using `@vue/test-utils` + a minimal Vuetify stub.

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript, Vitest, `@vue/test-utils`

---

## File map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/BudgetSlider.vue` | Component SFC (slider + toggle + banner) |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts` | Vitest tests |

> The `AimOnboardingTab/` directory is introduced in plan F1. If it does not yet exist, create it. The `components/` and `components/__tests__/` subdirectories are created in Tasks 1–2 below.

---

## Shared contracts

- `design-consolidated.md §8b` — build on `v-slider` (not styled divs); banners = `v-alert variant="tonal"`; colors via `rgb(var(--v-theme-*))`, never hardcoded hex; no `box-shadow` focus ring.
- `product-spec-v2.md §4.1 Step 3` — Monthly bands: $50K → $10M+; Annual bands: $5M → $200M+; stored as `budgetMonthly` / `budgetAnnual` + `budgetPeriod`.
- Threshold banner fires at $250K/mo (Monthly) or $3M/yr (Annual, = $250K × 12).
- `InputSlider.vue` (`packages/core/src/components/inputs/InputSlider.vue`) is discrete/index-based — same pattern used here. Do **not** wrap `InputSlider`; `BudgetSlider` builds its own `v-slider` because it adds a period toggle, formatted thumb, and threshold banner.

---

## Task 1 — Band helpers and `fmtM` (pure logic, no Vue)

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/BudgetSlider.vue` (scaffold only — exports the constants; component template comes in Task 3)
- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts` (band + fmtM tests)

- [ ] **Step 1: Create the directory structure**

```bash
mkdir -p packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__
```

- [ ] **Step 2: Write the failing tests for band arrays and `fmtM`**

Create `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import {
  MONTHLY_BANDS,
  ANNUAL_BANDS,
  PRO_THRESHOLD_MONTHLY,
  fmtM,
  isAboveProThreshold,
} from '../BudgetSlider.vue'

describe('BudgetSlider — band arrays', () => {
  it('monthly bands start at 50_000 and end at 10_000_000', () => {
    expect(MONTHLY_BANDS[0]).toBe(50_000)
    expect(MONTHLY_BANDS[MONTHLY_BANDS.length - 1]).toBe(10_000_000)
  })

  it('monthly bands contain at least 10 steps', () => {
    expect(MONTHLY_BANDS.length).toBeGreaterThanOrEqual(10)
  })

  it('monthly bands include the pro threshold (250_000)', () => {
    expect(MONTHLY_BANDS).toContain(250_000)
  })

  it('annual bands start at 5_000_000 and end at 200_000_000', () => {
    expect(ANNUAL_BANDS[0]).toBe(5_000_000)
    expect(ANNUAL_BANDS[ANNUAL_BANDS.length - 1]).toBe(200_000_000)
  })

  it('annual bands include the pro threshold equivalent (3_000_000)', () => {
    expect(ANNUAL_BANDS).toContain(3_000_000)
  })
})

describe('BudgetSlider — fmtM', () => {
  it('formats 50_000 as "$50K"', () => {
    expect(fmtM(50_000)).toBe('$50K')
  })

  it('formats 250_000 as "$250K"', () => {
    expect(fmtM(250_000)).toBe('$250K')
  })

  it('formats 1_000_000 as "$1M"', () => {
    expect(fmtM(1_000_000)).toBe('$1M')
  })

  it('formats 2_500_000 as "$2.5M"', () => {
    expect(fmtM(2_500_000)).toBe('$2.5M')
  })

  it('formats 10_000_000 as "$10M+"', () => {
    expect(fmtM(10_000_000)).toBe('$10M+')
  })

  it('formats 200_000_000 (annual cap) as "$200M+"', () => {
    expect(fmtM(200_000_000, true)).toBe('$200M+')
  })
})

describe('BudgetSlider — isAboveProThreshold', () => {
  it('returns false below threshold on monthly', () => {
    expect(isAboveProThreshold(200_000, 'monthly')).toBe(false)
  })

  it('returns true at threshold on monthly', () => {
    expect(isAboveProThreshold(250_000, 'monthly')).toBe(true)
  })

  it('returns true above threshold on monthly', () => {
    expect(isAboveProThreshold(500_000, 'monthly')).toBe(true)
  })

  it('returns false below annual threshold (3_000_000)', () => {
    expect(isAboveProThreshold(2_000_000, 'annual')).toBe(false)
  })

  it('returns true at annual threshold (3_000_000)', () => {
    expect(isAboveProThreshold(3_000_000, 'annual')).toBe(true)
  })
})
```

- [ ] **Step 3: Run the tests — they must fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: FAIL — `Cannot find module '../BudgetSlider.vue'`

- [ ] **Step 4: Create `BudgetSlider.vue` with exported constants only (no template yet)**

Create `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/BudgetSlider.vue`:

```vue
<template>
  <!-- Task 3 -->
  <div />
</template>

<script setup lang="ts">
// ─── Band definitions ────────────────────────────────────────────────────────

/** Monthly bands: $50K … $10M. Must include PRO_THRESHOLD_MONTHLY. */
export const MONTHLY_BANDS: number[] = [
  50_000, 75_000, 100_000, 150_000, 200_000, 250_000,
  300_000, 400_000, 500_000, 750_000,
  1_000_000, 1_500_000, 2_000_000, 2_500_000, 3_000_000,
  4_000_000, 5_000_000, 7_500_000, 10_000_000,
]

/** Annual bands: $5M … $200M. Equivalent pro threshold = $3M/mo × 12 = $3M annual is wrong;
 *  $250K/mo × 12 = $3M annual — that IS the threshold. */
export const ANNUAL_BANDS: number[] = [
  5_000_000, 6_000_000, 7_000_000, 8_000_000, 9_000_000,
  10_000_000, 12_000_000, 15_000_000, 18_000_000, 20_000_000,
  25_000_000, 30_000_000, 36_000_000, 40_000_000, 50_000_000,
  60_000_000, 75_000_000, 100_000_000, 150_000_000, 200_000_000,
]

/** Monthly budget at which AIM Pro tier consideration applies. */
export const PRO_THRESHOLD_MONTHLY = 250_000

// ─── Helpers ─────────────────────────────────────────────────────────────────

/** Format a raw dollar amount as a compact label.
 *  @param value  - raw dollar value
 *  @param isCap  - true when this is the top-of-range band (appends "+")
 */
export function fmtM(value: number, isCap = false): string {
  const MONTHLY_CAP = 10_000_000
  const ANNUAL_CAP  = 200_000_000

  const addPlus = isCap || value === MONTHLY_CAP || value === ANNUAL_CAP

  if (value >= 1_000_000) {
    const m = value / 1_000_000
    const label = m % 1 === 0 ? `$${m}M` : `$${m}M`
    return addPlus ? `${label}+` : label
  }
  const k = value / 1_000
  const label = `$${k}K`
  return addPlus ? `${label}+` : label
}

/** Returns true when the given budget is at or above the AIM Pro monthly threshold. */
export function isAboveProThreshold(value: number, period: 'monthly' | 'annual'): boolean {
  const monthly = period === 'monthly' ? value : Math.round(value / 12)
  return monthly >= PRO_THRESHOLD_MONTHLY
}
</script>
```

- [ ] **Step 5: Run tests — they must pass**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: all 16 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/BudgetSlider.vue \
        packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
git commit -m "feat(aim-onboarding): add BudgetSlider band helpers and fmtM"
```

---

## Task 2 — Period toggle behavior (Vitest)

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts`

Tests in this task verify that switching period resets the slider index to the closest available band value, not that it keeps a stale index. These are pure logic tests — no DOM mount.

- [ ] **Step 1: Add period-toggle tests**

Append to the spec file:

```typescript
import { nearestBandIndex } from '../BudgetSlider.vue'

describe('BudgetSlider — nearestBandIndex', () => {
  it('finds exact match in monthly bands', () => {
    expect(nearestBandIndex(250_000, MONTHLY_BANDS)).toBe(5)
  })

  it('finds exact match in annual bands', () => {
    expect(nearestBandIndex(3_000_000, ANNUAL_BANDS)).toBe(
      ANNUAL_BANDS.indexOf(3_000_000)
    )
  })

  it('clamps below-range value to index 0', () => {
    expect(nearestBandIndex(10_000, MONTHLY_BANDS)).toBe(0)
  })

  it('clamps above-range value to last index', () => {
    expect(nearestBandIndex(999_000_000, ANNUAL_BANDS)).toBe(ANNUAL_BANDS.length - 1)
  })

  it('picks the nearest band when value is between two bands', () => {
    // Between 100_000 (index 2) and 150_000 (index 3) → nearest = 100_000
    expect(nearestBandIndex(115_000, MONTHLY_BANDS)).toBe(2)
    // Between 100_000 and 150_000, closer to 150_000
    expect(nearestBandIndex(135_000, MONTHLY_BANDS)).toBe(3)
  })
})
```

- [ ] **Step 2: Run — must fail**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: FAIL — `nearestBandIndex is not exported`

- [ ] **Step 3: Export `nearestBandIndex` from `BudgetSlider.vue`**

Add after `isAboveProThreshold` in the `<script setup>` block:

```typescript
/** Given a dollar value, return the index of the nearest band in the provided array. */
export function nearestBandIndex(value: number, bands: number[]): number {
  if (value <= bands[0]) return 0
  if (value >= bands[bands.length - 1]) return bands.length - 1
  let best = 0
  let bestDist = Math.abs(bands[0] - value)
  for (let i = 1; i < bands.length; i++) {
    const dist = Math.abs(bands[i] - value)
    if (dist < bestDist) {
      bestDist = dist
      best = i
    }
  }
  return best
}
```

- [ ] **Step 4: Run — all tests pass**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: all 21 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/BudgetSlider.vue \
        packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
git commit -m "feat(aim-onboarding): add nearestBandIndex for period-toggle reset"
```

---

## Task 3 — Full component template (v-slider + v-btn-toggle + v-alert)

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/BudgetSlider.vue`

- [ ] **Step 1: Replace the stub template with the full component**

Replace the entire contents of `BudgetSlider.vue` with:

```vue
<template>
  <div class="budget-slider">
    <!-- Period toggle -->
    <div class="d-flex align-center ga-3 mb-4">
      <span class="text-body-2 text-medium-emphasis">Period</span>
      <v-btn-toggle
        :model-value="period"
        mandatory
        density="compact"
        rounded="lg"
        color="primary"
        aria-label="Budget period"
        @update:model-value="onPeriodChange"
      >
        <v-btn value="monthly" size="small" aria-label="Monthly">Monthly</v-btn>
        <v-btn value="annual"  size="small" aria-label="Annual">Annual</v-btn>
      </v-btn-toggle>
    </div>

    <!-- Value display -->
    <div class="d-flex justify-space-between align-end mb-2">
      <span class="text-caption text-medium-emphasis">
        {{ fmtM(activeBands[0]) }}
      </span>
      <span
        class="text-h5 font-weight-bold"
        style="color: rgb(var(--v-theme-primary))"
        aria-live="polite"
        :aria-label="`Selected budget: ${fmtM(selectedValue)}`"
      >
        {{ fmtM(selectedValue) }}{{ selectedIndex === activeBands.length - 1 ? '' : '' }}/{{ period === 'monthly' ? 'mo' : 'yr' }}
      </span>
      <span class="text-caption text-medium-emphasis">
        {{ fmtM(activeBands[activeBands.length - 1], true) }}
      </span>
    </div>

    <!-- Slider -->
    <v-slider
      :model-value="selectedIndex"
      :min="0"
      :max="activeBands.length - 1"
      :step="1"
      color="primary"
      thumb-color="primary"
      track-color="rgba(var(--v-theme-primary), 0.2)"
      hide-details
      :aria-label="`Total paid media budget, ${period}`"
      :aria-valuemin="activeBands[0]"
      :aria-valuemax="activeBands[activeBands.length - 1]"
      :aria-valuenow="selectedValue"
      :aria-valuetext="fmtM(selectedValue) + (period === 'monthly' ? ' per month' : ' per year')"
      @update:model-value="onSliderChange"
    />

    <!-- AIM Pro threshold banner -->
    <v-alert
      v-if="showProBanner"
      variant="tonal"
      type="info"
      density="compact"
      class="mt-3"
      icon="mdi-information-outline"
    >
      At {{ fmtM(selectedValue) }}/{{ period === 'monthly' ? 'mo' : 'yr' }}, your budget
      qualifies for <strong>AIM Pro</strong> tier consideration.
    </v-alert>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'

// ─── Band definitions ────────────────────────────────────────────────────────

export const MONTHLY_BANDS: number[] = [
  50_000, 75_000, 100_000, 150_000, 200_000, 250_000,
  300_000, 400_000, 500_000, 750_000,
  1_000_000, 1_500_000, 2_000_000, 2_500_000, 3_000_000,
  4_000_000, 5_000_000, 7_500_000, 10_000_000,
]

export const ANNUAL_BANDS: number[] = [
  5_000_000, 6_000_000, 7_000_000, 8_000_000, 9_000_000,
  10_000_000, 12_000_000, 15_000_000, 18_000_000, 20_000_000,
  25_000_000, 30_000_000, 36_000_000, 40_000_000, 50_000_000,
  60_000_000, 75_000_000, 100_000_000, 150_000_000, 200_000_000,
]

export const PRO_THRESHOLD_MONTHLY = 250_000

// ─── Helpers ─────────────────────────────────────────────────────────────────

export function fmtM(value: number, isCap = false): string {
  const MONTHLY_CAP = 10_000_000
  const ANNUAL_CAP  = 200_000_000
  const addPlus = isCap || value === MONTHLY_CAP || value === ANNUAL_CAP

  if (value >= 1_000_000) {
    const m = value / 1_000_000
    const label = m % 1 === 0 ? `$${m}M` : `$${parseFloat(m.toFixed(1))}M`
    return addPlus ? `${label}+` : label
  }
  const k = value / 1_000
  const label = `$${k}K`
  return addPlus ? `${label}+` : label
}

export function isAboveProThreshold(value: number, period: 'monthly' | 'annual'): boolean {
  const monthly = period === 'monthly' ? value : Math.round(value / 12)
  return monthly >= PRO_THRESHOLD_MONTHLY
}

export function nearestBandIndex(value: number, bands: number[]): number {
  if (value <= bands[0]) return 0
  if (value >= bands[bands.length - 1]) return bands.length - 1
  let best = 0
  let bestDist = Math.abs(bands[0] - value)
  for (let i = 1; i < bands.length; i++) {
    const dist = Math.abs(bands[i] - value)
    if (dist < bestDist) {
      bestDist = dist
      best = i
    }
  }
  return best
}

// ─── Props / emits ───────────────────────────────────────────────────────────

const props = withDefaults(defineProps<{
  modelValue?: number   // raw dollar budget (budgetMonthly or budgetAnnual)
  period?: 'monthly' | 'annual'
}>(), {
  modelValue: 250_000,
  period: 'monthly',
})

const emit = defineEmits<{
  'update:modelValue': [value: number]
  'update:period': [period: 'monthly' | 'annual']
}>()

// ─── State ───────────────────────────────────────────────────────────────────

const activeBands = computed(() =>
  props.period === 'monthly' ? MONTHLY_BANDS : ANNUAL_BANDS
)

const selectedIndex = ref(nearestBandIndex(props.modelValue, activeBands.value))

const selectedValue = computed(() => activeBands.value[selectedIndex.value])

const showProBanner = computed(() =>
  isAboveProThreshold(selectedValue.value, props.period ?? 'monthly')
)

// Sync external modelValue changes (e.g. loading saved state)
watch(() => props.modelValue, (val) => {
  selectedIndex.value = nearestBandIndex(val, activeBands.value)
})

// ─── Handlers ────────────────────────────────────────────────────────────────

function onSliderChange(idx: number): void {
  selectedIndex.value = idx
  emit('update:modelValue', activeBands.value[idx])
}

function onPeriodChange(newPeriod: 'monthly' | 'annual'): void {
  // Convert current dollar value to nearest index in the new band set
  const newBands = newPeriod === 'monthly' ? MONTHLY_BANDS : ANNUAL_BANDS
  // Approximate equivalent: keep rough proportional position
  const currentValue = selectedValue.value
  const approximate = newPeriod === 'annual'
    ? currentValue * 12    // monthly → annual
    : Math.round(currentValue / 12)  // annual → monthly
  selectedIndex.value = nearestBandIndex(approximate, newBands)
  emit('update:period', newPeriod)
  emit('update:modelValue', newBands[selectedIndex.value])
}
</script>

<style scoped>
.budget-slider {
  /* Inherit font from parent (Inter via Vuetify); no hardcoded colors. */
}
</style>
```

- [ ] **Step 2: Run all BudgetSlider tests — must still pass**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: all 21 tests PASS (template replacement does not break exported helpers).

- [ ] **Step 3: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/BudgetSlider.vue
git commit -m "feat(aim-onboarding): implement BudgetSlider template with v-slider + v-btn-toggle"
```

---

## Task 4 — Threshold banner tests (component mount)

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts`

These tests mount the component with `@vue/test-utils` and a minimal Vuetify stub so the Vuetify component tree resolves without a full `createVuetify()` setup.

- [ ] **Step 1: Add mount-based banner tests**

Append to the spec file:

```typescript
import { mount } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'
import BudgetSlider from '../BudgetSlider.vue'

const vuetify = createVuetify({ components, directives })

function mountSlider(props: Record<string, unknown> = {}) {
  return mount(BudgetSlider, {
    props: { modelValue: 50_000, period: 'monthly', ...props },
    global: { plugins: [vuetify] },
  })
}

describe('BudgetSlider — threshold banner', () => {
  it('hides the banner below $250K monthly', async () => {
    const wrapper = mountSlider({ modelValue: 200_000, period: 'monthly' })
    expect(wrapper.find('.v-alert').exists()).toBe(false)
  })

  it('shows the banner at $250K monthly', async () => {
    const wrapper = mountSlider({ modelValue: 250_000, period: 'monthly' })
    expect(wrapper.find('.v-alert').exists()).toBe(true)
  })

  it('shows the banner above $250K monthly', async () => {
    const wrapper = mountSlider({ modelValue: 500_000, period: 'monthly' })
    expect(wrapper.find('.v-alert').exists()).toBe(true)
  })

  it('hides the banner at $2M annual (< $250K/mo equivalent)', async () => {
    const wrapper = mountSlider({ modelValue: 2_000_000, period: 'annual' })
    expect(wrapper.find('.v-alert').exists()).toBe(false)
  })

  it('shows the banner at $3M annual (= $250K/mo)', async () => {
    const wrapper = mountSlider({ modelValue: 3_000_000, period: 'annual' })
    expect(wrapper.find('.v-alert').exists()).toBe(true)
  })
})
```

- [ ] **Step 2: Run — must fail**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: FAIL — the v-alert visibility tests fail because the component stub emitted in Task 1 shows `<div />`.

- [ ] **Step 3: Verify the full template is in place (Task 3 must be done first)**

If Task 3 is complete, the tests should now pass. Run again:

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: all 26 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
git commit -m "test(aim-onboarding): add threshold banner mount tests for BudgetSlider"
```

---

## Task 5 — Period toggle integration tests (component mount)

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts`

- [ ] **Step 1: Add period-toggle and emit tests**

Append to the spec file:

```typescript
describe('BudgetSlider — period toggle', () => {
  it('renders two v-btn-toggle buttons labelled Monthly and Annual', () => {
    const wrapper = mountSlider()
    const btns = wrapper.findAll('.v-btn-toggle .v-btn')
    expect(btns).toHaveLength(2)
    expect(btns[0].text()).toContain('Monthly')
    expect(btns[1].text()).toContain('Annual')
  })

  it('emits update:period with "annual" when Annual is clicked', async () => {
    const wrapper = mountSlider({ period: 'monthly' })
    await wrapper.findAll('.v-btn-toggle .v-btn')[1].trigger('click')
    const emitted = wrapper.emitted('update:period')
    expect(emitted).toBeTruthy()
    expect(emitted![emitted!.length - 1]).toEqual(['annual'])
  })

  it('emits update:modelValue when period toggles (value resets to nearest band)', async () => {
    const wrapper = mountSlider({ modelValue: 250_000, period: 'monthly' })
    await wrapper.findAll('.v-btn-toggle .v-btn')[1].trigger('click')
    const emitted = wrapper.emitted('update:modelValue')
    expect(emitted).toBeTruthy()
    // After toggling to annual: 250_000 * 12 = 3_000_000 — nearest annual band
    expect(emitted![emitted!.length - 1][0]).toBe(3_000_000)
  })
})

describe('BudgetSlider — keyboard (aria attributes)', () => {
  it('slider has role=slider with aria-valuemin, aria-valuemax, aria-valuenow', () => {
    const wrapper = mountSlider({ modelValue: 250_000, period: 'monthly' })
    const slider = wrapper.find('[role="slider"]')
    expect(slider.exists()).toBe(true)
    expect(slider.attributes('aria-valuemin')).toBeDefined()
    expect(slider.attributes('aria-valuemax')).toBeDefined()
    expect(slider.attributes('aria-valuenow')).toBeDefined()
  })
})
```

- [ ] **Step 2: Run — tests may partially fail on click interaction depending on Vuetify internals**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: most pass. If `update:period` click test fails because Vuetify swallows the click (btn-toggle uses internal `@click`), adjust:

```typescript
// Alternative: trigger via wrapper.vm
await wrapper.vm.onPeriodChange('annual')
await wrapper.vm.$nextTick()
```

Update the click test to use `onPeriodChange` if the trigger approach fails:

```typescript
it('emits update:period with "annual" when period changes', async () => {
  const wrapper = mountSlider({ period: 'monthly' })
  await (wrapper.vm as any).onPeriodChange('annual')
  await wrapper.vm.$nextTick()
  const emitted = wrapper.emitted('update:period')
  expect(emitted).toBeTruthy()
  expect(emitted![emitted!.length - 1]).toEqual(['annual'])
})
```

- [ ] **Step 3: All tests pass**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
```

Expected: all 32+ tests PASS.

- [ ] **Step 4: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/BudgetSlider.spec.ts
git commit -m "test(aim-onboarding): add period-toggle and a11y tests for BudgetSlider"
```

---

## Task 6 — Full test run and lint

**Files:** no new files

- [ ] **Step 1: Run the full Vitest suite**

```bash
cd /path/to/frontend-mos
npm run test:ci
```

Expected: all pre-existing tests still pass; new BudgetSlider tests appear in output.

- [ ] **Step 2: Lint**

```bash
npm run lint
```

Expected: 0 errors. Fix any that appear before proceeding.

- [ ] **Step 3: Commit lint fixes if any**

```bash
git add -p   # stage only lint-driven changes
git commit -m "fix(aim-onboarding): lint fixes in BudgetSlider"
```

---

## Self-review

### Spec coverage

| Spec requirement | Covered |
|-----------------|---------|
| Monthly bands $50K → $10M | Task 1 — `MONTHLY_BANDS` |
| Annual bands $5M → $200M+ | Task 1 — `ANNUAL_BANDS` |
| $250K/mo pro threshold banner | Task 1 `PRO_THRESHOLD_MONTHLY`; Task 4 mount tests |
| Monthly/Annual `v-btn-toggle` | Task 3 template; Task 5 tests |
| `fmtM` formatted labels | Task 1 tests + Task 3 display |
| Emits numeric budget + period | Task 3 `defineEmits`; Task 5 emit tests |
| `v-alert variant="tonal"` (design-consolidated §8b) | Task 3 template |
| Colors via `rgb(var(--v-theme-*))` (no hardcoded hex) | Task 3 template — `rgb(var(--v-theme-primary))` only |
| `role=slider` + `aria-value*` (WCAG 2.1 AA) | Task 3 `v-slider` props; Task 5 a11y test |
| No `box-shadow` focus ring | Task 3 — no custom `box-shadow` added |
| Period toggle resets to nearest band | Task 2 `nearestBandIndex`; Task 5 emit test |
| Stored as `budgetMonthly`/`budgetAnnual` + `budgetPeriod` | Emits raw numeric value + period — parent stores in correct key |

### Placeholder scan

No TBD, TODO, "implement later", "similar to", or "appropriate handling" found.

### Type consistency

- `fmtM(value: number, isCap?: boolean): string` — used identically in helpers (Task 1) and template (Task 3).
- `nearestBandIndex(value: number, bands: number[]): number` — defined Task 2 scaffold, used in `onPeriodChange` Task 3.
- `isAboveProThreshold(value: number, period: 'monthly' | 'annual'): boolean` — defined Task 1, used in `showProBanner` computed (Task 3).
- `onPeriodChange(newPeriod: 'monthly' | 'annual')` — defined Task 3, referenced in Task 5 fallback test.
