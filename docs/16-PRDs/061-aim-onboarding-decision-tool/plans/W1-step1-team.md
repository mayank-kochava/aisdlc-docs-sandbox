---
id: plan-w1
title: "W1 — Step 1: Your Team"
---

## W1 — Step 1: Your Team

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `Step1Team.vue` — the first wizard step that collects company name, project lead(s), and data lead(s), with a "Same as project lead" sync checkbox, bound to `useAimOnboarding` state.

**Architecture:** A single `.vue` SFC under `AimOnboardingTab/steps/`. Uses `InputText` and `InputCheckbox` from `@mos/core` (wraps Vuetify's `v-text-field`/`v-checkbox`). Binds directly to `wizard.companyName`, `wizard.projectLeads[]`, `wizard.dataLeads[]`, and a new `wizard.dataLeadsSameAsProject` boolean via `patchWizard()`. The "same as" sync is component-local logic — no backend concern. Required markers use asterisk + `aria-required`. Validation state is read from the composable's `stepStatus`; no field-level regex is added here (type="email" is sufficient per §8b). Tests mock the composable and assert render + behavior without Pinia or router.

**Tech Stack:** Vue 3.5 / TypeScript 5, Vuetify 3, `@mos/core` (`InputText`, `InputCheckbox`), Vitest + @vue/test-utils.

---

## Files

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step1Team.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step1Team.spec.ts` |

**Depends on (must be merged first):**

- `F2` — `interfaces/aimOnboarding.ts` must define `Lead`, `WizardState` (with `companyName`, `projectLeads`, `dataLeads`, `dataLeadsSameAsProject`).
- `F3` — `composables/useAimOnboarding.ts` must expose `{ wizard, patchWizard }`.

**Interface addition required (coordinate with F2 before implementing):**
`WizardState` must include `dataLeadsSameAsProject: boolean`. Default value: `false`. This key is wizard-local state (never sent to the backend as a meaningful business field — it drives the sync UX only; the data leads array is what persists). Add to `WizardState` in `interfaces/aimOnboarding.ts` and to `defaultWizard()` in F3. Mongo doc shape (`§5`) does not need a new top-level key — `dataLeadsSameAsProject` lives only in the frontend wizard state (it is not stored; the synced `dataLeads` array is what is saved). **Note this in a comment in `aimOnboarding.ts`.**

---

## Task 1: Write the failing component tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step1Team.spec.ts`

- [ ] **Step 1: Create the spec file**

```ts
// steps/__tests__/Step1Team.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import { InputText, InputCheckbox } from '@mos/core'
import Step1Team from '../Step1Team.vue'

// ── Mock useAimOnboarding ────────────────────────────────────────────────────
// The composable is a module-scope singleton. We control its returned refs here.
import { ref } from 'vue'

const mockWizard = ref({
  companyName: '',
  projectLeads: [{ name: '', email: '' }],
  dataLeads: [{ name: '', email: '' }],
  dataLeadsSameAsProject: false,
})

const mockPatchWizard = vi.fn((patch: Record<string, unknown>) => {
  Object.assign(mockWizard.value, patch)
})

vi.mock('../../composables/useAimOnboarding', () => ({
  useAimOnboarding: () => ({
    wizard: mockWizard,
    patchWizard: mockPatchWizard,
  }),
}))

// ── Helpers ──────────────────────────────────────────────────────────────────

function mountComponent() {
  return mount(Step1Team, {
    global: {
      plugins: [createVuetify()],
      stubs: { InputText, InputCheckbox },
    },
  })
}

// ── Tests ────────────────────────────────────────────────────────────────────

describe('Step1Team.vue', () => {
  beforeEach(() => {
    mockWizard.value = {
      companyName: '',
      projectLeads: [{ name: '', email: '' }],
      dataLeads: [{ name: '', email: '' }],
      dataLeadsSameAsProject: false,
    }
    mockPatchWizard.mockClear()
  })

  // ── Render ─────────────────────────────────────────────────────────────────

  it('renders the step heading', () => {
    const wrapper = mountComponent()
    expect(wrapper.text()).toContain('Your Team')
  })

  it('renders the Company name InputText with required marker', () => {
    const wrapper = mountComponent()
    const inputs = wrapper.findAllComponents(InputText)
    // First InputText is company name
    expect(inputs.length).toBeGreaterThanOrEqual(1)
    // Required asterisk present somewhere in the label area
    expect(wrapper.html()).toContain('aria-required')
  })

  it('renders one project lead row on mount', () => {
    const wrapper = mountComponent()
    // Each row has a Name and an Email InputText; 1 lead = 2 fields + 1 company = 3 total minimum
    const inputs = wrapper.findAllComponents(InputText)
    expect(inputs.length).toBeGreaterThanOrEqual(3)
  })

  it('renders the "Same as project lead" InputCheckbox', () => {
    const wrapper = mountComponent()
    const checkbox = wrapper.findComponent(InputCheckbox)
    expect(checkbox.exists()).toBe(true)
  })

  it('renders one data lead row when sameAsProject is false', () => {
    const wrapper = mountComponent()
    // data leads section is visible (not hidden)
    expect(wrapper.find('[data-testid="data-leads-section"]').exists()).toBe(true)
  })

  // ── Add / remove rows ──────────────────────────────────────────────────────

  it('clicking "+ Add another project lead" appends a blank row', async () => {
    const wrapper = mountComponent()
    const addBtn = wrapper.find('[data-testid="add-project-lead"]')
    expect(addBtn.exists()).toBe(true)
    await addBtn.trigger('click')
    expect(mockWizard.value.projectLeads.length).toBe(2)
  })

  it('clicking × on a project lead row removes it (when length > 1)', async () => {
    mockWizard.value.projectLeads = [
      { name: 'Alice', email: 'alice@example.com' },
      { name: 'Bob', email: 'bob@example.com' },
    ]
    const wrapper = mountComponent()
    const removeBtn = wrapper.find('[data-testid="remove-project-lead-1"]')
    expect(removeBtn.exists()).toBe(true)
    await removeBtn.trigger('click')
    expect(mockWizard.value.projectLeads.length).toBe(1)
    expect(mockWizard.value.projectLeads[0].name).toBe('Alice')
  })

  it('× button on a project lead is disabled when only one row remains', () => {
    const wrapper = mountComponent()
    // Only one row: button should be disabled or not rendered
    const removeBtn = wrapper.find('[data-testid="remove-project-lead-0"]')
    // Either not present or disabled
    if (removeBtn.exists()) {
      expect((removeBtn.element as HTMLButtonElement).disabled).toBe(true)
    } else {
      expect(removeBtn.exists()).toBe(false)
    }
  })

  it('clicking "+ Add another data lead" appends a blank data lead row', async () => {
    const wrapper = mountComponent()
    const addBtn = wrapper.find('[data-testid="add-data-lead"]')
    expect(addBtn.exists()).toBe(true)
    await addBtn.trigger('click')
    expect(mockWizard.value.dataLeads.length).toBe(2)
  })

  it('clicking × on a data lead row removes it (when length > 1)', async () => {
    mockWizard.value.dataLeads = [
      { name: 'Carol', email: 'carol@example.com' },
      { name: 'Dan', email: 'dan@example.com' },
    ]
    const wrapper = mountComponent()
    const removeBtn = wrapper.find('[data-testid="remove-data-lead-1"]')
    await removeBtn.trigger('click')
    expect(mockWizard.value.dataLeads.length).toBe(1)
    expect(mockWizard.value.dataLeads[0].name).toBe('Carol')
  })

  // ── "Same as project lead" sync ───────────────────────────────────────────

  it('checking "Same as project lead" hides the data leads section', async () => {
    const wrapper = mountComponent()
    const checkbox = wrapper.findComponent(InputCheckbox)
    checkbox.vm.$emit('update:modelValue', true)
    await wrapper.vm.$nextTick()
    expect(mockWizard.value.dataLeadsSameAsProject).toBe(true)
    // data leads section hidden
    const section = wrapper.find('[data-testid="data-leads-section"]')
    expect(section.exists()).toBe(false)
  })

  it('when synced, dataLeads mirrors projectLeads in state', async () => {
    mockWizard.value.projectLeads = [{ name: 'Eve', email: 'eve@example.com' }]
    mockWizard.value.dataLeadsSameAsProject = true
    // Simulate the component applying sync on projectLeads change
    const wrapper = mountComponent()
    await wrapper.vm.$nextTick()
    expect(mockWizard.value.dataLeads).toEqual([{ name: 'Eve', email: 'eve@example.com' }])
  })

  it('unchecking "Same as project lead" restores the data leads section', async () => {
    mockWizard.value.dataLeadsSameAsProject = true
    const wrapper = mountComponent()
    const checkbox = wrapper.findComponent(InputCheckbox)
    checkbox.vm.$emit('update:modelValue', false)
    await wrapper.vm.$nextTick()
    expect(wrapper.find('[data-testid="data-leads-section"]').exists()).toBe(true)
  })

  // ── Accessibility ─────────────────────────────────────────────────────────

  it('remove buttons have an aria-label', () => {
    mockWizard.value.projectLeads = [
      { name: 'Alice', email: 'alice@example.com' },
      { name: 'Bob', email: 'bob@example.com' },
    ]
    const wrapper = mountComponent()
    const removeBtns = wrapper.findAll('[data-testid^="remove-project-lead-"]')
    removeBtns.forEach(btn => {
      expect(btn.attributes('aria-label')).toBeTruthy()
    })
  })
})
```

- [ ] **Step 2: Run the tests to confirm they fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step1Team.spec.ts
```

Expected: FAIL — `Cannot find module '../Step1Team.vue'`

---

## Task 2: Implement `Step1Team.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step1Team.vue`

- [ ] **Step 3: Create the component**

```vue
<template>
  <div class="step-1-team">
    <!-- Eyebrow + heading -->
    <div class="step-eyebrow">Step <strong>1</strong> of 8</div>
    <h2 class="step-title">Your Team</h2>
    <p class="step-lead">
      Tell us who is leading this AIM implementation so we can address the Scope
      of Work to the right people.
    </p>

    <!-- 1. Company name ───────────────────────────────────────────────────── -->
    <div class="qcard">
      <div class="qhead">
        <span class="qnum">1.1</span>
        <span class="qtitle">
          Company name
          <span class="req" aria-hidden="true"> *</span>
        </span>
      </div>
      <InputText
        :model-value="wizard.companyName"
        label="Company name"
        placeholder="e.g. Acme Corp"
        aria-required="true"
        :aria-label="'Company name (required)'"
        @update:model-value="(v: string) => patchWizard({ companyName: v ?? '' })"
      />
    </div>

    <!-- 2. Project lead(s) ────────────────────────────────────────────────── -->
    <div class="qcard">
      <div class="qhead">
        <span class="qnum">1.2</span>
        <span class="qtitle">
          Project lead(s)
          <span class="req" aria-hidden="true"> *</span>
        </span>
      </div>
      <p class="qhelp">At least one required. This person will sign the Scope of Work.</p>

      <div
        v-for="(lead, idx) in wizard.projectLeads"
        :key="idx"
        class="lead-row"
      >
        <div class="lead-row-fields">
          <InputText
            :model-value="lead.name"
            :label="`Name${idx === 0 ? ' (required)' : ''}`"
            placeholder="Full name"
            :aria-label="`Project lead ${idx + 1} name`"
            aria-required="true"
            @update:model-value="(v: string) => updateLead('projectLeads', idx, 'name', v ?? '')"
          />
          <InputText
            :model-value="lead.email"
            :label="`Email${idx === 0 ? ' (required)' : ''}`"
            placeholder="name@company.com"
            type="email"
            :aria-label="`Project lead ${idx + 1} email`"
            aria-required="true"
            @update:model-value="(v: string) => updateLead('projectLeads', idx, 'email', v ?? '')"
          />
        </div>
        <v-btn
          v-if="wizard.projectLeads.length > 1"
          :data-testid="`remove-project-lead-${idx}`"
          :aria-label="`Remove project lead ${idx + 1}`"
          icon
          variant="text"
          density="compact"
          :disabled="wizard.projectLeads.length <= 1"
          @click="removeLead('projectLeads', idx)"
        >
          <v-icon>mdi-close</v-icon>
        </v-btn>
        <v-btn
          v-else
          :data-testid="`remove-project-lead-${idx}`"
          :aria-label="`Remove project lead ${idx + 1}`"
          icon
          variant="text"
          density="compact"
          disabled
          style="visibility: hidden"
        >
          <v-icon>mdi-close</v-icon>
        </v-btn>
      </div>

      <v-btn
        data-testid="add-project-lead"
        variant="outlined"
        size="small"
        class="add-lead-btn mt-3"
        prepend-icon="mdi-plus"
        @click="addLead('projectLeads')"
      >
        Add another project lead
      </v-btn>
    </div>

    <!-- 3. Data lead(s) ───────────────────────────────────────────────────── -->
    <div class="qcard">
      <div class="qhead">
        <span class="qnum">1.3</span>
        <span class="qtitle">
          Data lead(s)
          <span class="req" aria-hidden="true"> *</span>
        </span>
      </div>
      <p class="qhelp">
        The person responsible for supplying data files or API credentials.
        At least one required.
      </p>

      <InputCheckbox
        :model-value="wizard.dataLeadsSameAsProject"
        label="Same as project lead"
        class="mb-2"
        @update:model-value="onSameAsToggle"
      />

      <div
        v-if="!wizard.dataLeadsSameAsProject"
        data-testid="data-leads-section"
      >
        <div
          v-for="(lead, idx) in wizard.dataLeads"
          :key="idx"
          class="lead-row"
        >
          <div class="lead-row-fields">
            <InputText
              :model-value="lead.name"
              :label="`Name${idx === 0 ? ' (required)' : ''}`"
              placeholder="Full name"
              :aria-label="`Data lead ${idx + 1} name`"
              aria-required="true"
              @update:model-value="(v: string) => updateLead('dataLeads', idx, 'name', v ?? '')"
            />
            <InputText
              :model-value="lead.email"
              :label="`Email${idx === 0 ? ' (required)' : ''}`"
              placeholder="name@company.com"
              type="email"
              :aria-label="`Data lead ${idx + 1} email`"
              aria-required="true"
              @update:model-value="(v: string) => updateLead('dataLeads', idx, 'email', v ?? '')"
            />
          </div>
          <v-btn
            v-if="wizard.dataLeads.length > 1"
            :data-testid="`remove-data-lead-${idx}`"
            :aria-label="`Remove data lead ${idx + 1}`"
            icon
            variant="text"
            density="compact"
            :disabled="wizard.dataLeads.length <= 1"
            @click="removeLead('dataLeads', idx)"
          >
            <v-icon>mdi-close</v-icon>
          </v-btn>
          <v-btn
            v-else
            :data-testid="`remove-data-lead-${idx}`"
            :aria-label="`Remove data lead ${idx + 1}`"
            icon
            variant="text"
            density="compact"
            disabled
            style="visibility: hidden"
          >
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </div>

        <v-btn
          data-testid="add-data-lead"
          variant="outlined"
          size="small"
          class="add-lead-btn mt-3"
          prepend-icon="mdi-plus"
          @click="addLead('dataLeads')"
        >
          Add another data lead
        </v-btn>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { watch } from 'vue'
import { InputText, InputCheckbox } from '@mos/core'
import { useAimOnboarding } from '../composables/useAimOnboarding'
import type { Lead } from '../interfaces/aimOnboarding'

const { wizard, patchWizard } = useAimOnboarding()

// ── Lead array helpers ────────────────────────────────────────────────────────

function addLead(field: 'projectLeads' | 'dataLeads') {
  const leads = [...wizard.value[field], { name: '', email: '' }]
  patchWizard({ [field]: leads })
}

function removeLead(field: 'projectLeads' | 'dataLeads', idx: number) {
  if (wizard.value[field].length <= 1) return
  const leads = wizard.value[field].filter((_: Lead, i: number) => i !== idx)
  patchWizard({ [field]: leads })
}

function updateLead(
  field: 'projectLeads' | 'dataLeads',
  idx: number,
  key: 'name' | 'email',
  value: string,
) {
  const leads = wizard.value[field].map((lead: Lead, i: number) =>
    i === idx ? { ...lead, [key]: value } : lead,
  )
  patchWizard({ [field]: leads })
}

// ── "Same as project lead" sync ───────────────────────────────────────────────

function onSameAsToggle(checked: boolean) {
  patchWizard({ dataLeadsSameAsProject: checked })
  if (checked) {
    // Immediately mirror project leads into data leads
    patchWizard({ dataLeads: wizard.value.projectLeads.map((l: Lead) => ({ ...l })) })
  }
}

// Keep data leads in sync while the checkbox is on and project leads change
watch(
  () => wizard.value.projectLeads,
  (leads) => {
    if (wizard.value.dataLeadsSameAsProject) {
      patchWizard({ dataLeads: leads.map((l: Lead) => ({ ...l })) })
    }
  },
  { deep: true },
)
</script>

<style scoped>
.step-1-team {
  max-width: 680px;
}

.step-eyebrow {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.07em;
  text-transform: uppercase;
  color: rgb(var(--v-theme-text-secondary, 92, 94, 96));
  margin-bottom: 6px;
}

.step-title {
  font-size: 24px;
  font-weight: 700;
  letter-spacing: -0.015em;
  margin: 0 0 6px;
}

.step-lead {
  color: rgb(var(--v-theme-text-secondary, 92, 94, 96));
  font-size: 14px;
  margin: 0 0 4px;
}

.qcard {
  background: rgb(var(--v-theme-surface));
  border: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
  border-radius: 8px;
  padding: 22px 24px;
  margin-top: 20px;
}

.qhead {
  display: flex;
  align-items: center;
  gap: 9px;
  margin-bottom: 12px;
}

.qnum {
  font-size: 10px;
  font-weight: 600;
  color: rgb(var(--v-theme-on-surface-variant, 92, 94, 96));
  background: rgb(var(--v-theme-surface-variant, 251, 251, 251));
  border: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
  border-radius: 5px;
  padding: 1px 6px;
}

.qtitle {
  font-size: 15.5px;
  font-weight: 600;
}

.req {
  color: rgb(var(--v-theme-error));
}

.qhelp {
  font-size: 13px;
  color: rgb(var(--v-theme-on-surface-variant, 92, 94, 96));
  margin: 0 0 16px;
}

.lead-row {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin-bottom: 12px;
}

.lead-row-fields {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  flex: 1;
}

.add-lead-btn {
  /* v-btn handles its own styling */
}
</style>
```

- [ ] **Step 4: Run the tests — confirm they pass**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step1Team.spec.ts
```

Expected output:

```text
✓ Step1Team.vue > renders the step heading
✓ Step1Team.vue > renders the Company name InputText with required marker
✓ Step1Team.vue > renders one project lead row on mount
✓ Step1Team.vue > renders the "Same as project lead" InputCheckbox
✓ Step1Team.vue > renders one data lead row when sameAsProject is false
✓ Step1Team.vue > clicking "+ Add another project lead" appends a blank row
✓ Step1Team.vue > clicking × on a project lead row removes it (when length > 1)
✓ Step1Team.vue > × button on a project lead is disabled when only one row remains
✓ Step1Team.vue > clicking "+ Add another data lead" appends a blank data lead row
✓ Step1Team.vue > clicking × on a data lead row removes it (when length > 1)
✓ Step1Team.vue > checking "Same as project lead" hides the data leads section
✓ Step1Team.vue > when synced, dataLeads mirrors projectLeads in state
✓ Step1Team.vue > unchecking "Same as project lead" restores the data leads section
✓ Step1Team.vue > remove buttons have an aria-label

Test Files  1 passed (1)
Tests       14 passed (14)
```

If a test fails due to a missing `data-testid` attribute, verify the attribute name in the template matches the test selector exactly (e.g. `data-testid="remove-project-lead-0"`).

- [ ] **Step 5: Run the full lint check**

```bash
npx eslint packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step1Team.vue packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step1Team.spec.ts
```

Expected: no errors or warnings. Fix any `no-explicit-any`, unused-import, or template lint issues before proceeding.

- [ ] **Step 6: Commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step1Team.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step1Team.spec.ts
git commit -m "feat(onboarding): add Step1Team wizard step with lead repeaters and same-as sync"
```

---

## Task 3: Wire `dataLeadsSameAsProject` into F2 interfaces and F3 composable

> This task is a coordination task — it produces changes to two files already owned by F2 and F3 plans. If those plans are already merged, apply these changes directly. If not, add the field to their respective task lists before merging.

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts`
- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts`

- [ ] **Step 7: Add `dataLeadsSameAsProject` to `WizardState`**

In `interfaces/aimOnboarding.ts`, locate the `WizardState` interface and add:

```ts
// UI-only sync flag — not stored to MongoDB; only the dataLeads array persists.
dataLeadsSameAsProject: boolean
```

- [ ] **Step 8: Add default value in `defaultWizard()`**

In `composables/useAimOnboarding.ts`, locate the `defaultWizard()` function and add:

```ts
dataLeadsSameAsProject: false,
```

- [ ] **Step 9: Run all onboarding composable tests to confirm no regression**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding.spec.ts
```

Expected: all tests pass (unchanged from F3 baseline).

- [ ] **Step 10: Commit the interface/composable update**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts
git commit -m "feat(onboarding): add dataLeadsSameAsProject to WizardState for Step1 sync"
```

---

## Self-review

### Spec coverage

| §4.1 requirement | Covered by |
|---|---|
| Company name — required text input | Task 2, qcard 1.1 |
| Project lead(s) — repeating name+email, min 1, +Add/× | Task 2, lead-row loop; addLead/removeLead helpers |
| × disabled when only 1 row | `disabled` when `projectLeads.length <= 1` |
| Data lead(s) — same repeating pattern, min 1 | Task 2, data leads section |
| "Same as project lead" checkbox, syncs in real time | `onSameAsToggle` + `watch` on projectLeads |
| Required markers | Asterisk spans + `aria-required` attributes |
| Binds to `useAimOnboarding` state (`companyName`, `projectLeads[]`, `dataLeads[]`) | `patchWizard` calls throughout |
| Feeds output: company name + lead names populate SoW | Stored in wizard state; SoW tab reads these keys |
| WCAG 2.1 AA — aria-required, aria-label on remove buttons | `aria-required="true"`, `aria-label` on all × buttons |
| Colors via CSS custom properties, no hardcoded hex | Style block uses `rgb(var(--v-theme-*))` |
| `InputText`/`InputCheckbox` from `@mos/core` | Import line in `<script setup>` |
| No email-format regex (type="email" sufficient) | `type="email"` on email fields only |

### Placeholder scan

No TBDs, no "implement later", no "similar to Task N" — every step contains real code.

### Type consistency

- `Lead` interface imported from F2 (`{ name: string; email: string }`); used in `updateLead`, `removeLead`, `addLead`, and the `watch` callback.
- `patchWizard` called with `Partial<WizardState>` keys; key names (`companyName`, `projectLeads`, `dataLeads`, `dataLeadsSameAsProject`) match F3 `wizard.value` access in this same file.
- `data-testid` attribute names in template match selector strings in tests exactly.
