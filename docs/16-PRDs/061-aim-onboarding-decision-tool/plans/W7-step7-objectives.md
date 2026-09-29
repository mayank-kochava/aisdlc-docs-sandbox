---
id: plan-w7
title: "W7 — Step 7: Objectives"
---

## W7 — Step 7: Objectives

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development`
> (recommended) or `superpowers:executing-plans` to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `Step7Objectives.vue` — an optional wizard step with three free-text
textareas (`objectivesGoal`, `objectivesSuccess`, `objectivesMarketing`) and a tonal
info banner, bound to `useAimOnboarding`.

**Architecture:** Pure presentational step component using Vuetify's `v-textarea` for
each field and `v-alert variant="tonal"` for the optional-step banner (the established
pattern in `AimOnboardingTab`). All three fields map directly to state keys in the
`useAimOnboarding` composable (F3). No validation logic — this step never blocks.
Component communicates only via the composable; no props or emits needed beyond what the
step wrapper already provides.

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript, Vitest + `@vue/test-utils` (jsdom).

---

## Dependencies

- **F2** (`services/onboarding.ts` + `interfaces/aimOnboarding.ts`) — `objectivesGoal`,
  `objectivesSuccess`, `objectivesMarketing` are defined in the `AimOnboardingWizard`
  interface. Must be merged before this plan.
- **F3** (`useAimOnboarding.ts`) — composable exposes `state` (reactive) and `setState`
  (patch helper). Must be merged before this plan.

---

## File map

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step7Objectives.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step7Objectives.spec.ts` |

All paths relative to `/Users/mukey/Documents/kochava-projects/k4a/frontend-mos/`.

---

## Token reference (from `packages/app/src/plugins/vuetify.ts`)

Use `rgb(var(--v-theme-TOKEN))` in CSS — never hardcode hex.

| Semantic role | Token | Light hex |
|---|---|---|
| Primary | `primary` | `#1E4B97` |
| Info/tonal banner | `info` | `#095ABD` |
| Muted text | `grey-2` | `#6D757B` |
| Page background | `surface-2` | `#F8F7F7` |
| Border | `border` | `#E1E1E1` |

---

## Task 1: `Step7Objectives.spec.ts` — write failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step7Objectives.spec.ts`

### Context for the tests

`Step7Objectives.vue` must:

1. Render a tonal `v-alert` info banner stating all fields are optional.
2. Render three `<textarea>` elements (via `v-textarea`) bound to `objectivesGoal`,
   `objectivesSuccess`, and `objectivesMarketing`.
3. Call `setState` with the correct key when a textarea value changes.
4. Display the current state values inside each textarea.
5. Never emit a validation error — the step never blocks.

The component uses `useAimOnboarding` internally. The test stubs that composable via
`vi.mock` so there is no dependency on the store or backend.

- [ ] **Step 1.1: Create the `__tests__` directory**

```bash
mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__
```

- [ ] **Step 1.2: Write the failing tests**

Create `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step7Objectives.spec.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount } from '@vue/test-utils'
import { reactive } from 'vue'
import Step7Objectives from '../Step7Objectives.vue'

// Stub the composable so no store / backend is needed.
const mockSetState = vi.fn()
const mockState = reactive({
  objectivesGoal: '',
  objectivesSuccess: '',
  objectivesMarketing: '',
})

vi.mock(
  '../../composables/useAimOnboarding',
  () => ({
    useAimOnboarding: () => ({
      state: mockState,
      setState: mockSetState,
    }),
  })
)

function mountStep() {
  return mount(Step7Objectives, {
    global: {
      // tests.config.ts registers Vuetify globally; include it here too.
      plugins: [],
    },
  })
}

describe('Step7Objectives', () => {
  beforeEach(() => {
    mockSetState.mockClear()
    mockState.objectivesGoal = ''
    mockState.objectivesSuccess = ''
    mockState.objectivesMarketing = ''
  })

  // ── Banner ──────────────────────────────────────────────────────────────

  describe('optional banner', () => {
    it('renders a v-alert with variant tonal', () => {
      const wrapper = mountStep()
      // Vuetify renders v-alert as a div with class v-alert
      const alert = wrapper.find('.v-alert')
      expect(alert.exists()).toBe(true)
    })

    it('banner text communicates all fields are optional', () => {
      const wrapper = mountStep()
      const alert = wrapper.find('.v-alert')
      expect(alert.text().toLowerCase()).toContain('optional')
    })
  })

  // ── Textarea presence ────────────────────────────────────────────────────

  describe('textarea rendering', () => {
    it('renders a textarea for objectivesGoal', () => {
      const wrapper = mountStep()
      expect(wrapper.find('textarea[data-field="objectivesGoal"]').exists()).toBe(true)
    })

    it('renders a textarea for objectivesSuccess', () => {
      const wrapper = mountStep()
      expect(wrapper.find('textarea[data-field="objectivesSuccess"]').exists()).toBe(true)
    })

    it('renders a textarea for objectivesMarketing', () => {
      const wrapper = mountStep()
      expect(wrapper.find('textarea[data-field="objectivesMarketing"]').exists()).toBe(true)
    })
  })

  // ── Initial value binding ────────────────────────────────────────────────

  describe('initial value display', () => {
    it('pre-fills objectivesGoal from state', () => {
      mockState.objectivesGoal = 'Grow ROAS by 20%'
      const wrapper = mountStep()
      const ta = wrapper.find('textarea[data-field="objectivesGoal"]')
      expect((ta.element as HTMLTextAreaElement).value).toBe('Grow ROAS by 20%')
    })

    it('pre-fills objectivesSuccess from state', () => {
      mockState.objectivesSuccess = 'CPA < $10'
      const wrapper = mountStep()
      const ta = wrapper.find('textarea[data-field="objectivesSuccess"]')
      expect((ta.element as HTMLTextAreaElement).value).toBe('CPA < $10')
    })

    it('pre-fills objectivesMarketing from state', () => {
      mockState.objectivesMarketing = 'Brand awareness in APAC'
      const wrapper = mountStep()
      const ta = wrapper.find('textarea[data-field="objectivesMarketing"]')
      expect((ta.element as HTMLTextAreaElement).value).toBe('Brand awareness in APAC')
    })
  })

  // ── setState on input ────────────────────────────────────────────────────

  describe('setState on input', () => {
    it('calls setState with objectivesGoal on input', async () => {
      const wrapper = mountStep()
      const ta = wrapper.find('textarea[data-field="objectivesGoal"]')
      await ta.setValue('Increase market share')
      expect(mockSetState).toHaveBeenCalledWith(
        expect.objectContaining({ objectivesGoal: 'Increase market share' })
      )
    })

    it('calls setState with objectivesSuccess on input', async () => {
      const wrapper = mountStep()
      const ta = wrapper.find('textarea[data-field="objectivesSuccess"]')
      await ta.setValue('50 conversions/day')
      expect(mockSetState).toHaveBeenCalledWith(
        expect.objectContaining({ objectivesSuccess: '50 conversions/day' })
      )
    })

    it('calls setState with objectivesMarketing on input', async () => {
      const wrapper = mountStep()
      const ta = wrapper.find('textarea[data-field="objectivesMarketing"]')
      await ta.setValue('Retarget lapsed users')
      expect(mockSetState).toHaveBeenCalledWith(
        expect.objectContaining({ objectivesMarketing: 'Retarget lapsed users' })
      )
    })
  })

  // ── No validation ────────────────────────────────────────────────────────

  describe('no blocking validation', () => {
    it('does not render any required asterisk or error message', () => {
      const wrapper = mountStep()
      // Vuetify renders required asterisks inside .v-label or via :required attribute
      const textareas = wrapper.findAll('textarea')
      textareas.forEach((ta) => {
        expect(ta.attributes('required')).toBeUndefined()
        expect(ta.attributes('aria-required')).toBeUndefined()
      })
    })
  })
})
```

- [ ] **Step 1.3: Run the tests — expect them to fail (module does not exist)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step7Objectives.spec.ts 2>&1 | tail -20
```

Expected output: `FAIL` — "Cannot find module '../Step7Objectives.vue'"

---

## Task 2: `Step7Objectives.vue` — implement until tests pass

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step7Objectives.vue`

- [ ] **Step 2.1: Create `steps/` directory if it does not yet exist**

```bash
mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps
```

- [ ] **Step 2.2: Create the component**

```vue
<template>
  <div class="step7-objectives">
    <!-- Optional-step info banner -->
    <v-alert
      variant="tonal"
      color="info"
      icon="mdi-information-outline"
      class="step7-objectives__banner mb-6"
    >
      All fields on this step are <strong>optional</strong>. Your answers feed the
      Objectives section of the Scope of Work. You can leave these blank and add
      them later in the SoW notes field.
    </v-alert>

    <!-- Objectives: primary goal -->
    <div class="step7-objectives__field mb-5">
      <label class="step7-objectives__label" :for="ids.goal">
        What is the primary goal of this AIM engagement?
      </label>
      <p class="step7-objectives__hint" :id="ids.goalHint">
        For example: improve ROAS, reduce CPA, model incrementality across channels.
      </p>
      <v-textarea
        :id="ids.goal"
        :model-value="state.objectivesGoal"
        :aria-describedby="ids.goalHint"
        data-field="objectivesGoal"
        rows="3"
        auto-grow
        variant="outlined"
        density="comfortable"
        hide-details
        placeholder="Describe your primary goal…"
        @update:model-value="(v) => setState({ objectivesGoal: v ?? '' })"
      />
    </div>

    <!-- Objectives: success criteria -->
    <div class="step7-objectives__field mb-5">
      <label class="step7-objectives__label" :for="ids.success">
        How will you measure success?
      </label>
      <p class="step7-objectives__hint" :id="ids.successHint">
        For example: CPA below $15, ROAS above 4×, incremental conversions up 10%.
      </p>
      <v-textarea
        :id="ids.success"
        :model-value="state.objectivesSuccess"
        :aria-describedby="ids.successHint"
        data-field="objectivesSuccess"
        rows="3"
        auto-grow
        variant="outlined"
        density="comfortable"
        hide-details
        placeholder="Describe your success criteria…"
        @update:model-value="(v) => setState({ objectivesSuccess: v ?? '' })"
      />
    </div>

    <!-- Objectives: marketing context -->
    <div class="step7-objectives__field">
      <label class="step7-objectives__label" :for="ids.marketing">
        Any broader marketing context we should know?
      </label>
      <p class="step7-objectives__hint" :id="ids.marketingHint">
        For example: upcoming product launch, seasonal peak, brand campaign running
        alongside performance.
      </p>
      <v-textarea
        :id="ids.marketing"
        :model-value="state.objectivesMarketing"
        :aria-describedby="ids.marketingHint"
        data-field="objectivesMarketing"
        rows="3"
        auto-grow
        variant="outlined"
        density="comfortable"
        hide-details
        placeholder="Add any relevant marketing context…"
        @update:model-value="(v) => setState({ objectivesMarketing: v ?? '' })"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { useAimOnboarding } from '../composables/useAimOnboarding'

const { state, setState } = useAimOnboarding()

// Stable IDs for label/aria-describedby wiring (WCAG §8b).
const ids = {
  goal: 'aim-obj-goal',
  goalHint: 'aim-obj-goal-hint',
  success: 'aim-obj-success',
  successHint: 'aim-obj-success-hint',
  marketing: 'aim-obj-marketing',
  marketingHint: 'aim-obj-marketing-hint',
}
</script>

<style scoped>
.step7-objectives {
  max-width: 680px;
}

.step7-objectives__label {
  display: block;
  font-size: 14px;
  font-weight: 600;
  color: rgb(var(--v-theme-on-surface));
  margin-bottom: 4px;
}

.step7-objectives__hint {
  font-size: 13px;
  color: rgb(var(--v-theme-grey-2));
  margin: 0 0 8px;
  line-height: 1.5;
}
</style>
```

- [ ] **Step 2.3: Run the tests — expect them to pass**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step7Objectives.spec.ts 2>&1 | tail -20
```

Expected output: all tests `PASS` — 0 failures.

- [ ] **Step 2.4: Run the full advertiser test suite to confirm no regressions**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx vitest run 2>&1 | tail -30
```

Expected output: no new failures (pre-existing skipped tests are fine).

- [ ] **Step 2.5: Lint both files**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser && npx eslint \
  src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step7Objectives.vue \
  src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step7Objectives.spec.ts \
  2>&1 | tail -20
```

Expected output: no errors. Fix any warnings before the commit.

- [ ] **Step 2.6: Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step7Objectives.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step7Objectives.spec.ts
git commit -m "feat(aim-onboarding): add Step7Objectives with 3 optional textareas, tonal banner, and Vitest tests"
```

---

## Self-review checklist

**Spec coverage (product-spec-v2.md §4.1 Step 7 + design-consolidated.md §5/§8a):**

| Requirement | Task |
|---|---|
| Three optional free-text fields: `objectivesGoal`, `objectivesSuccess`, `objectivesMarketing` | Tasks 1 + 2 |
| All fields optional — no validation, never blocks | Tasks 1 + 2 |
| Info banner communicating all fields are optional | Tasks 1 + 2 |
| Fields bind to `useAimOnboarding` composable state | Task 2 |
| `setState` called on input with the correct key | Tasks 1 + 2 |
| Pre-fills from existing state (e.g. restored from localStorage) | Task 1 (binding tests) |
| Helper text / hint per field | Task 2 |
| Feeds SoW (O1) — state keys match `wizard.objectivesGoal/Success/Marketing` in design-consolidated §5 | Task 2 (key names) |
| No required asterisk, no `aria-required`, no `required` attribute | Task 1 (no-validation test) |
| Labels wired to textareas via `for`/`id` (WCAG 2.1 AA §8b) | Task 2 |
| Hint text wired via `aria-describedby` (WCAG 2.1 AA §8b) | Task 2 |
| Colors via `rgb(var(--v-theme-*))` — no hardcoded hex | Task 2 |
| `v-alert variant="tonal"` pattern (design-consolidated §8b precedent) | Task 2 |
| Uses `v-textarea` (Vuetify — not a styled `<div>`) per §8b | Task 2 |

**Placeholder scan:** No TBD / TODO / "similar to" language present.

**Type consistency:** `objectivesGoal`, `objectivesSuccess`, `objectivesMarketing` match
the state keys defined in `design-consolidated.md §5` and the `AimOnboardingWizard`
interface in F2. `setState` signature matches F3's patch helper.
