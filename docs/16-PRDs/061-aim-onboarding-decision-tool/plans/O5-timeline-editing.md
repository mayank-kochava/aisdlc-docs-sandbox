---
id: plan-o5
title: "O5 — Timeline Editing + Cascade"
---

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire interactive editing into the Timeline tab: any authenticated user (client or CSM) can change a milestone's status, add/edit notes, or trigger the delay flow (pick new date → preview downstream cascade → confirm → persist via `onboardingService.updateTimeline`).

**Architecture:** Three changes land together. (1) `TimelineTab.vue` (O4) drops its `isCsm`-based `disabled` gate on the status `v-select` (design-consolidated §7 overrides O4's accidental read-only gate — both client and CSM may edit) and emits `update:milestones` events. (2) A new `TimelineDelayPreviewDialog.vue` encapsulates the delay flow: accepts proposed `delayedToDate`, calls `computeEstDates` locally (no API call) to render old→new date shifts for every downstream milestone, then emits `confirm` with the chosen date. (3) `services/onboarding.ts` gains `updateTimeline(sessionId, dto)` which calls `PUT /{id}/timeline`. Author is never client-supplied — the backend stamps `CurrentUser.UserId` from the session token. No free-text name field anywhere in the UI.

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript, Vitest 4.1.2, @vue/test-utils 2.2.6, `v-data-table`, `v-select` (density compact), `v-dialog`, `v-date-picker` (Vuetify built-in).

---

## Spec override note

`product-spec-v2.md §4.2` states timeline status edits are CSM-only. **`design-consolidated.md §7` and §8a override this** — timeline editing (status, delay, notes) is editable by both client and CSM. O4 mistakenly gated on `isCsm`; this plan corrects that. The `design-consolidated` doc is the authority on all conflicts (see `plans/00-index.md` — "This wins on every conflict").

---

## Files

**Modify:**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue` — remove `isCsm` gate; wire `v-select` `update:modelValue`; open delay dialog on "Delayed" selection; emit `update:milestones`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts` — add editing + delay flow tests
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts` — add `updateTimeline(sessionId, dto)` method

**Create:**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.vue`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.test.ts`

---

## Dependencies

**O4** (`output/TimelineTab.vue`) — the base component this plan modifies. Must be merged or the component file must exist with the props/structure from O4.

**F7** (`logic/milestones.ts`) — exports `computeEstDates(milestones: Milestone[], anchorDate: Date): Record<string, EstDates>`. Used client-side in the delay preview to render downstream shifts without an API call. Must be merged or stubbed (see stub block below).

**B8** (`PUT /{id}/timeline`) — the backend endpoint. `onboardingService.updateTimeline` maps to this. Must be deployed or the service call must be mocked in tests.

**F2** (`services/onboarding.ts`, `interfaces/aimOnboarding.ts`) — the service file that gains `updateTimeline`; the `Milestone`/`TimelineEditDto` types.

---

## F2 + F7 Type Contract (reference — do not re-invent)

Import from the merged files. If not merged, use these stubs in test files only (remove once merged):

```ts
// Stub — remove once F2 + F7 are merged
export interface Milestone {
  key: string
  name: string
  responsible: string
  weekOffset: number
  durationLabel: string
  durationWeeks: number
  status: string
  delayedToDate: string | null
  notes: string
}

export interface EstDates {
  start: Date
  end: Date
}

export interface TimelineEditDto {
  MilestoneKey: string
  StatusUpdate?: string | null
  DelayedToDate?: string | null  // ISO date string
  Notes?: string | null
}
```

---

## `onboardingService.updateTimeline` signature

This method is added to `services/onboarding.ts` (F2). The backend contract is B8 (`PUT /{advertiserId}/onboarding-sessions/{id}/timeline`):

```ts
async updateTimeline(sessionId: string, dto: TimelineEditDto): Promise<void> {
  await mmmPortalApi.put(
    getPopulatedEndpoint(`mmm/advertisers/{advertiser_id}/onboarding-sessions/${sessionId}/timeline`),
    dto
  )
}
```

---

## Tasks

### Task 1 — Add `updateTimeline` to `services/onboarding.ts`

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts`

- [ ] **Step 1.1 — Write the failing test**

Open (or create if F2 is not yet merged) the service test file. Add a describe block for `updateTimeline`:

```ts
// onboarding.spec.ts (or onboarding.test.ts — match whichever F2 created)
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'

const { mockMmmPortalApi, mockCoreStore } = vi.hoisted(() => ({
  mockMmmPortalApi: { put: vi.fn() },
  mockCoreStore: {
    getPopulatedEndpoint: vi.fn((ep: string) => ep.replace('{advertiser_id}', 'adv-42')),
  },
}))

vi.mock('../../../../services/mmm', () => ({ mmmPortalApi: mockMmmPortalApi }))
vi.mock('@mos/core', () => ({
  endpoints: { mmm: {} },
  useMInsightsCoreStore: () => mockCoreStore,
}))

import { onboardingService } from '../services/onboarding'

describe('onboardingService.updateTimeline', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()
  })

  it('calls PUT /timeline with the correct URL and DTO', async () => {
    mockMmmPortalApi.put.mockResolvedValue({ data: {} })

    await onboardingService.updateTimeline('session-123', {
      MilestoneKey: 'kickoff',
      StatusUpdate: 'in_progress',
    })

    expect(mockMmmPortalApi.put).toHaveBeenCalledOnce()
    const [url, dto] = mockMmmPortalApi.put.mock.calls[0]
    expect(url).toContain('/adv-42/onboarding-sessions/session-123/timeline')
    expect(dto.MilestoneKey).toBe('kickoff')
    expect(dto.StatusUpdate).toBe('in_progress')
  })

  it('includes DelayedToDate in the payload when provided', async () => {
    mockMmmPortalApi.put.mockResolvedValue({ data: {} })

    await onboardingService.updateTimeline('session-456', {
      MilestoneKey: 'data_connect',
      StatusUpdate: 'delayed',
      DelayedToDate: '2026-07-06',
    })

    const [, dto] = mockMmmPortalApi.put.mock.calls[0]
    expect(dto.DelayedToDate).toBe('2026-07-06')
  })

  it('includes Notes in the payload when provided', async () => {
    mockMmmPortalApi.put.mockResolvedValue({ data: {} })

    await onboardingService.updateTimeline('session-789', {
      MilestoneKey: 'kickoff',
      Notes: 'Call rescheduled',
    })

    const [, dto] = mockMmmPortalApi.put.mock.calls[0]
    expect(dto.Notes).toBe('Call rescheduled')
  })
})
```

- [ ] **Step 1.2 — Run to confirm FAIL**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.spec.ts 2>&1 | tail -20
```

Expected: FAIL — `updateTimeline is not a function` or similar.

- [ ] **Step 1.3 — Implement `updateTimeline` in `services/onboarding.ts`**

Open `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts`. Add the method inside the service object (after whatever F2 already added):

```ts
async updateTimeline(sessionId: string, dto: TimelineEditDto): Promise<void> {
  const { useMInsightsCoreStore } = await import('@mos/core')
  const coreStore = useMInsightsCoreStore()
  const url = coreStore.getPopulatedEndpoint(
    `mmm/advertisers/{advertiser_id}/onboarding-sessions/${sessionId}/timeline`
  )
  await mmmPortalApi.put(url, dto)
},
```

Also ensure `TimelineEditDto` is imported at the top of the file:

```ts
import type { TimelineEditDto } from '../interfaces/aimOnboarding'
```

- [ ] **Step 1.4 — Run to confirm PASS**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.spec.ts 2>&1 | tail -20
```

Expected: all 3 `updateTimeline` tests pass.

- [ ] **Step 1.5 — Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts
git commit -m "feat(onboarding): add updateTimeline to onboardingService (O5)"
```

---

### Task 2 — Write failing tests for `TimelineDelayPreviewDialog.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.test.ts`

- [ ] **Step 2.1 — Create the test file**

```ts
// TimelineDelayPreviewDialog.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'
import TimelineDelayPreviewDialog from './TimelineDelayPreviewDialog.vue'
import type { Milestone } from '../interfaces/aimOnboarding'

// ── If F7 not yet merged, add the stubs from the Dependencies section ───────

const vuetify = createVuetify({ components, directives })

const ANCHOR = '2026-06-15' // Monday

function makeMilestones(): Milestone[] {
  return [
    { key: 'kickoff',      name: 'Kickoff',              responsible: 'Client + Kochava', weekOffset: 0, durationLabel: '90 min',    durationWeeks: 0, status: '',          delayedToDate: null, notes: '' },
    { key: 'sow_approval', name: 'SoW approval',         responsible: 'Client + Kochava', weekOffset: 0, durationLabel: '1 hour',    durationWeeks: 0, status: '',          delayedToDate: null, notes: '' },
    { key: 'data_connect', name: 'Data connection',      responsible: 'Client',           weekOffset: 1, durationLabel: '2-3 weeks', durationWeeks: 3, status: '',          delayedToDate: null, notes: '' },
    { key: 'data_qa',      name: 'Data QA',              responsible: 'Kochava',          weekOffset: 3, durationLabel: '~1 week',   durationWeeks: 1, status: '',          delayedToDate: null, notes: '' },
    { key: 'model_train',  name: 'Model training',       responsible: 'Kochava',          weekOffset: 4, durationLabel: '1-2 weeks', durationWeeks: 2, status: '',          delayedToDate: null, notes: '' },
    { key: 'first_insight',name: 'First insights',       responsible: 'Kochava',          weekOffset: 6, durationLabel: '1 hour',    durationWeeks: 0, status: '',          delayedToDate: null, notes: '' },
  ]
}

function mountDialog(props: Record<string, unknown> = {}) {
  return mount(TimelineDelayPreviewDialog, {
    global: { plugins: [vuetify] },
    props: {
      milestoneKey:  'kickoff',
      milestones:    makeMilestones(),
      anchorDate:    ANCHOR,
      modelValue:    true,   // dialog open
      ...props,
    },
  })
}

// ── Dialog renders ───────────────────────────────────────────────────────────

describe('TimelineDelayPreviewDialog — renders', () => {
  it('renders when modelValue is true', async () => {
    const w = mountDialog()
    await flushPromises()
    expect(w.findComponent({ name: 'VDialog' }).exists()).toBe(true)
  })

  it('shows the milestone name in the title', async () => {
    const w = mountDialog()
    await flushPromises()
    expect(w.text()).toContain('Kickoff')
  })

  it('renders a date picker or date input', async () => {
    const w = mountDialog()
    await flushPromises()
    // v-date-picker or v-text-field for date entry must exist
    const hasPicker = w.findComponent({ name: 'VDatePicker' }).exists()
    const hasField  = w.find('input[type="date"], .v-date-picker, [data-testid="delay-date-input"]').exists()
    expect(hasPicker || hasField).toBe(true)
  })

  it('shows a Confirm button', async () => {
    const w = mountDialog()
    await flushPromises()
    const btns = w.findAllComponents({ name: 'VBtn' })
    const labels = btns.map((b) => b.text().toLowerCase())
    expect(labels.some((l) => l.includes('confirm'))).toBe(true)
  })

  it('shows a Cancel button', async () => {
    const w = mountDialog()
    await flushPromises()
    const btns = w.findAllComponents({ name: 'VBtn' })
    const labels = btns.map((b) => b.text().toLowerCase())
    expect(labels.some((l) => l.includes('cancel'))).toBe(true)
  })
})

// ── Preview computation ──────────────────────────────────────────────────────

describe('TimelineDelayPreviewDialog — cascade preview', () => {
  it('shows old→new date shifts after a new date is entered', async () => {
    const w = mountDialog()
    await flushPromises()

    // Simulate selecting 2026-06-22 (one week after kickoff natural date 2026-06-15)
    // The component exposes pendingDate as a data/ref
    await w.setData({ pendingDate: '2026-06-22' })
    await flushPromises()

    // Preview table must show the shifted downstream dates
    // data_connect natural = 2026-06-22; shifted +7 days → 2026-06-29 ("WC 29 Jun")
    expect(w.text()).toContain('WC 29 Jun')
  })

  it('does NOT shift milestones before the edited one', async () => {
    // kickoff is the first milestone — nothing is before it.
    // Use sow_approval as target so kickoff is "before"
    const w = mountDialog({ milestoneKey: 'sow_approval' })
    await flushPromises()

    await w.setData({ pendingDate: '2026-06-22' })
    await flushPromises()

    // kickoff (weekOffset=0, anchor=2026-06-15) should still show "WC 15 Jun"
    expect(w.text()).toContain('WC 15 Jun')
  })

  it('shows zero-shift message when new date equals natural date', async () => {
    // kickoff natural = 2026-06-15; set pending date to same Monday
    const w = mountDialog()
    await flushPromises()

    await w.setData({ pendingDate: '2026-06-15' })
    await flushPromises()

    // No shift — all downstream dates unchanged
    // data_connect natural = WC 22 Jun
    expect(w.text()).toContain('WC 22 Jun')
  })
})

// ── Emits ────────────────────────────────────────────────────────────────────

describe('TimelineDelayPreviewDialog — emits', () => {
  it('emits "confirm" with the selected ISO date when Confirm clicked', async () => {
    const w = mountDialog()
    await flushPromises()

    await w.setData({ pendingDate: '2026-06-22' })
    await flushPromises()

    const btns = w.findAllComponents({ name: 'VBtn' })
    const confirmBtn = btns.find((b) => b.text().toLowerCase().includes('confirm'))
    await confirmBtn!.trigger('click')

    expect(w.emitted('confirm')).toBeTruthy()
    expect(w.emitted('confirm')![0][0]).toBe('2026-06-22')
  })

  it('emits "update:modelValue" with false when Cancel clicked', async () => {
    const w = mountDialog()
    await flushPromises()

    const btns = w.findAllComponents({ name: 'VBtn' })
    const cancelBtn = btns.find((b) => b.text().toLowerCase().includes('cancel'))
    await cancelBtn!.trigger('click')

    expect(w.emitted('update:modelValue')).toBeTruthy()
    expect(w.emitted('update:modelValue')![0][0]).toBe(false)
  })

  it('Confirm button is disabled when no date is selected', async () => {
    const w = mountDialog()
    await flushPromises()

    // pendingDate is null initially
    const btns = w.findAllComponents({ name: 'VBtn' })
    const confirmBtn = btns.find((b) => b.text().toLowerCase().includes('confirm'))
    expect(confirmBtn!.props('disabled')).toBe(true)
  })
})
```

- [ ] **Step 2.2 — Run to confirm FAIL**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.test.ts 2>&1 | tail -20
```

Expected: FAIL — `Cannot find module './TimelineDelayPreviewDialog.vue'`.

---

### Task 3 — Implement `TimelineDelayPreviewDialog.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.vue`

- [ ] **Step 3.1 — Create the component**

```vue
<template>
  <!-- design-consolidated §8a: delay flow = pick date → preview shifts → confirm -->
  <v-dialog
    :model-value="modelValue"
    max-width="640"
    @update:model-value="$emit('update:modelValue', $event)"
  >
    <v-card class="aim-delay-dialog">
      <v-card-title class="aim-delay-dialog__title">
        Delay milestone: <strong>{{ targetMilestone?.name }}</strong>
      </v-card-title>

      <v-card-text>
        <!-- ── Date picker ───────────────────────────────────────────────── -->
        <p class="mb-2 text-body-2">Select the new estimated start date:</p>
        <v-date-picker
          v-model="pendingDateObj"
          :min="minDate"
          color="primary"
          show-adjacent-months
          class="aim-delay-dialog__picker"
        />

        <!-- ── Cascade preview table ─────────────────────────────────────── -->
        <template v-if="pendingDate && previewRows.length">
          <p class="mt-4 mb-2 text-body-2 font-weight-medium">
            Downstream date shifts preview:
          </p>
          <v-table density="compact" class="aim-delay-dialog__preview-table">
            <thead>
              <tr>
                <th>Milestone</th>
                <th>Current est. start</th>
                <th>New est. start</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="row in previewRows"
                :key="row.key"
                :class="{ 'aim-delay-dialog__shifted': row.shifted }"
              >
                <td>{{ row.name }}</td>
                <td>{{ row.oldStart }}</td>
                <td>
                  <span
                    v-if="row.shifted"
                    class="text-warning font-weight-medium"
                  >{{ row.newStart }}</span>
                  <span v-else>{{ row.newStart }}</span>
                </td>
              </tr>
            </tbody>
          </v-table>
        </template>

        <v-alert
          v-if="!pendingDate"
          type="info"
          variant="tonal"
          density="compact"
          class="mt-4"
        >
          Choose a new date above to preview the impact on subsequent milestones.
        </v-alert>
      </v-card-text>

      <v-card-actions class="justify-end">
        <v-btn
          variant="text"
          @click="$emit('update:modelValue', false)"
        >
          Cancel
        </v-btn>
        <v-btn
          color="primary"
          variant="flat"
          :disabled="!pendingDate"
          @click="handleConfirm"
        >
          Confirm delay
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { computeEstDates } from '../logic/milestones'
import type { Milestone } from '../interfaces/aimOnboarding'

// ── Props / emits ─────────────────────────────────────────────────────────────

const props = defineProps<{
  /** Controls v-dialog open state. */
  modelValue: boolean
  /** Key of the milestone being delayed. */
  milestoneKey: string
  /** Full milestone array (order matters for cascade). */
  milestones: Milestone[]
  /** ISO date string for the anchor (approvedAt or today). */
  anchorDate: string
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', val: boolean): void
  /** Emitted when the user confirms — payload is the chosen ISO date string. */
  (e: 'confirm', newDate: string): void
}>()

// ── State ─────────────────────────────────────────────────────────────────────

/** v-date-picker uses a Date object; we derive the ISO string from it. */
const pendingDateObj = ref<Date | null>(null)

/** ISO date string (YYYY-MM-DD) derived from pendingDateObj. */
const pendingDate = computed<string | null>(() => {
  if (!pendingDateObj.value) return null
  const d = pendingDateObj.value
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
})

/** Reset picker when dialog opens. */
watch(() => props.modelValue, (open) => {
  if (open) pendingDateObj.value = null
})

// ── Helpers ──────────────────────────────────────────────────────────────────

/** Format a Date as "WC D Mon", e.g. "WC 22 Jun" */
function fmtWC(date: Date): string {
  return `WC ${date.getDate()} ${date.toLocaleString('en-GB', { month: 'short' })}`
}

// ── Computed ──────────────────────────────────────────────────────────────────

const targetMilestone = computed(() =>
  props.milestones.find((m) => m.key === props.milestoneKey) ?? null
)

/** The earliest selectable date — tomorrow (can't delay to today or the past). */
const minDate = computed(() => {
  const d = new Date()
  d.setDate(d.getDate() + 1)
  return d.toISOString().slice(0, 10)
})

/**
 * Current est-dates (no pending delay applied) — the "before" column.
 * Uses the anchor from props and the existing milestone array as-is.
 */
const currentEstDates = computed(() =>
  computeEstDates(props.milestones, new Date(props.anchorDate))
)

/**
 * Preview est-dates — the "after" column.
 * Clones the milestone array, applies the proposed delayedToDate to the target
 * milestone, then recomputes.  This is a pure local calculation (no API call).
 */
const previewEstDates = computed(() => {
  if (!pendingDate.value) return null
  const cloned: Milestone[] = props.milestones.map((m) => {
    if (m.key === props.milestoneKey) {
      return { ...m, delayedToDate: pendingDate.value! }
    }
    return { ...m }
  })
  return computeEstDates(cloned, new Date(props.anchorDate))
})

/**
 * Rows for the preview table.
 * Shows all milestones; highlights rows where the new start differs from the current.
 */
const previewRows = computed(() => {
  if (!previewEstDates.value) return []
  return props.milestones.map((m) => {
    const currentStart = currentEstDates.value[m.key]?.start
    const newStart     = previewEstDates.value![m.key]?.start
    const oldFmt = currentStart ? fmtWC(currentStart) : '—'
    const newFmt = newStart     ? fmtWC(newStart)     : '—'
    return {
      key:      m.key,
      name:     m.name,
      oldStart: oldFmt,
      newStart: newFmt,
      shifted:  oldFmt !== newFmt,
    }
  })
})

// ── Handlers ──────────────────────────────────────────────────────────────────

function handleConfirm() {
  if (!pendingDate.value) return
  emit('confirm', pendingDate.value)
  emit('update:modelValue', false)
}
</script>
```

- [ ] **Step 3.2 — Run to confirm PASS**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.test.ts 2>&1 | tail -30
```

Expected: all tests pass.

- [ ] **Step 3.3 — Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineDelayPreviewDialog.test.ts
git commit -m "feat(onboarding): add TimelineDelayPreviewDialog with cascade preview (O5)"
```

---

### Task 4 — Wire editing into `TimelineTab.vue`

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue`

- [ ] **Step 4.1 — Write the failing tests (add to `TimelineTab.test.ts`)**

Open `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts`. Append these describe blocks after the existing tests:

```ts
// ── Editing — status change (O5 additions) ───────────────────────────────────

describe('TimelineTab — status editing (O5)', () => {
  it('v-selects are NOT disabled — editing open to all users (no isCsm gate)', () => {
    // design-consolidated §7 overrides O4's isCsm gate
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    selects.forEach((sel) => {
      expect(sel.props('disabled')).toBe(false)
    })
  })

  it('emits "update:milestones" when a non-delay status is selected', async () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    // simulate selecting 'in_progress' on the first row (kickoff)
    selects[0].vm.$emit('update:modelValue', 'in_progress')
    await flushPromises()

    expect(w.emitted('update:milestones')).toBeTruthy()
    const emitted = w.emitted('update:milestones')![0][0] as Milestone[]
    const kickoff = emitted.find((m) => m.key === 'kickoff')
    expect(kickoff?.status).toBe('in_progress')
  })

  it('emits "update:milestones" with "complete" status', async () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    selects[0].vm.$emit('update:modelValue', 'complete')
    await flushPromises()

    const emitted = w.emitted('update:milestones')![0][0] as Milestone[]
    expect(emitted.find((m) => m.key === 'kickoff')?.status).toBe('complete')
  })
})

// ── Editing — delay flow triggers dialog ─────────────────────────────────────

describe('TimelineTab — delay dialog trigger (O5)', () => {
  it('opens TimelineDelayPreviewDialog when "delayed" is selected', async () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    selects[0].vm.$emit('update:modelValue', 'delayed')
    await flushPromises()

    const dialog = w.findComponent({ name: 'TimelineDelayPreviewDialog' })
    expect(dialog.exists()).toBe(true)
    expect(dialog.props('modelValue')).toBe(true)
    expect(dialog.props('milestoneKey')).toBe('kickoff')
  })

  it('passes milestones and anchorDate to the dialog', async () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    selects[0].vm.$emit('update:modelValue', 'delayed')
    await flushPromises()

    const dialog = w.findComponent({ name: 'TimelineDelayPreviewDialog' })
    expect(dialog.props('milestones')).toBeTruthy()
    expect(dialog.props('anchorDate')).toBe('2026-06-15')
  })
})

// ── Editing — confirm delay updates milestones ────────────────────────────────

describe('TimelineTab — confirm delay (O5)', () => {
  it('emits "update:milestones" with delayedToDate and status=delayed on confirm', async () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    selects[0].vm.$emit('update:modelValue', 'delayed')
    await flushPromises()

    const dialog = w.findComponent({ name: 'TimelineDelayPreviewDialog' })
    dialog.vm.$emit('confirm', '2026-06-22')
    await flushPromises()

    expect(w.emitted('update:milestones')).toBeTruthy()
    const updated = w.emitted('update:milestones')![0][0] as Milestone[]
    const kickoff = updated.find((m) => m.key === 'kickoff')
    expect(kickoff?.status).toBe('delayed')
    expect(kickoff?.delayedToDate).toBe('2026-06-22')
  })

  it('closes the dialog after confirm', async () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    selects[0].vm.$emit('update:modelValue', 'delayed')
    await flushPromises()

    const dialog = w.findComponent({ name: 'TimelineDelayPreviewDialog' })
    dialog.vm.$emit('confirm', '2026-06-22')
    await flushPromises()

    expect(dialog.props('modelValue')).toBe(false)
  })
})

// ── Editing — notes ──────────────────────────────────────────────────────────

describe('TimelineTab — notes editing (O5)', () => {
  it('renders a notes textarea or text-field for each row', () => {
    const w = mountTab()
    // Each row has a notes input; check at least one exists
    const inputs = w.findAll('[data-testid^="notes-input-"]')
    expect(inputs.length).toBe(6)
  })

  it('emits "update:milestones" with updated notes when notes field changes', async () => {
    const w = mountTab()
    const noteInput = w.find('[data-testid="notes-input-kickoff"]')
    await noteInput.setValue('Follow-up needed')
    await flushPromises()

    expect(w.emitted('update:milestones')).toBeTruthy()
    const updated = w.emitted('update:milestones')![0][0] as Milestone[]
    expect(updated.find((m) => m.key === 'kickoff')?.notes).toBe('Follow-up needed')
  })
})
```

Also add the import at the top if not already present:

```ts
import { flushPromises } from '@vue/test-utils'
import type { Milestone } from '../interfaces/aimOnboarding'
```

- [ ] **Step 4.2 — Run to confirm FAIL**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts 2>&1 | tail -30
```

Expected: all O5 test blocks FAIL (update:milestones not emitted, dialog not opened, etc.).

- [ ] **Step 4.3 — Update `TimelineTab.vue`**

Open `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue`.

**Replace the `<template #item.status>` slot** (the read-only v-select block from O4) with:

```vue
<!-- Status v-select — editable by client AND CSM (design-consolidated §7) -->
<template #item.status="{ item }">
  <v-select
    :model-value="item.status || null"
    :items="STATUS_OPTIONS"
    item-title="label"
    item-value="value"
    density="compact"
    variant="outlined"
    hide-details
    placeholder="— Status —"
    class="aim-timeline-tab__status-sel"
    style="min-width: 140px;"
    @update:model-value="handleStatusChange(item.key, $event)"
  />
</template>

<!-- Notes text-field per row -->
<template #item.notes="{ item }">
  <v-text-field
    :model-value="item.notes"
    density="compact"
    variant="outlined"
    hide-details
    placeholder="Add notes…"
    :data-testid="`notes-input-${item.key}`"
    class="aim-timeline-tab__notes-field"
    style="min-width: 180px;"
    @update:model-value="handleNotesChange(item.key, $event)"
  />
</template>
```

**Add the Notes column to `headers`:**

```ts
const headers = [
  { title: 'Milestone',   key: 'name',        sortable: false },
  { title: 'Responsible', key: 'responsible',  sortable: false },
  { title: 'Est. start',  key: 'estStart',     sortable: false },
  { title: 'Duration',    key: 'duration',     sortable: false },
  { title: 'Est. end',    key: 'estEnd',       sortable: false },
  { title: 'Status',      key: 'status',       sortable: false },
  { title: 'Notes',       key: 'notes',        sortable: false },
]
```

**Update the `rows` computed** to include `notes`:

```ts
return props.milestones.map((m) => {
  const est = estMap[m.key]
  return {
    key:         m.key,
    name:        m.name,
    responsible: m.responsible,
    estStart:    est ? fmtWC(est.start) : '—',
    duration:    m.durationLabel,
    estEnd:      est ? fmtWC(est.end)   : '—',
    status:      m.status,
    notes:       m.notes,
  }
})
```

**Remove the `isCsm` prop** from `defineProps` (or keep it for CSM-only features elsewhere, but stop using it to gate status editing).

**Add `emit` and delay dialog state to `<script setup>`:**

```ts
import { computed, ref } from 'vue'
import TimelineDelayPreviewDialog from './TimelineDelayPreviewDialog.vue'

const emit = defineEmits<{
  (e: 'update:milestones', milestones: Milestone[]): void
}>()

// ── Delay dialog state ────────────────────────────────────────────────────────

/** Key of the milestone currently being delayed; null = dialog closed. */
const delayingMilestoneKey = ref<string | null>(null)

const delayDialogOpen = computed({
  get: () => delayingMilestoneKey.value !== null,
  set: (val: boolean) => { if (!val) delayingMilestoneKey.value = null },
})

// ── Handlers ──────────────────────────────────────────────────────────────────

/**
 * Called when the status v-select changes for a row.
 * If the new status is 'delayed', opens the delay preview dialog instead of
 * emitting immediately — the emit happens only after the user confirms.
 */
function handleStatusChange(milestoneKey: string, newStatus: string) {
  if (newStatus === 'delayed') {
    delayingMilestoneKey.value = milestoneKey
    return
  }
  emitMilestoneUpdate(milestoneKey, { status: newStatus })
}

/**
 * Called when the delay dialog confirms a new date.
 * Sets both status='delayed' and delayedToDate on the target milestone.
 */
function handleDelayConfirm(newDate: string) {
  if (!delayingMilestoneKey.value) return
  emitMilestoneUpdate(delayingMilestoneKey.value, {
    status: 'delayed',
    delayedToDate: newDate,
  })
  delayingMilestoneKey.value = null
}

function handleNotesChange(milestoneKey: string, notes: string) {
  emitMilestoneUpdate(milestoneKey, { notes })
}

/**
 * Clones props.milestones, applies the patch to the target key, emits.
 */
function emitMilestoneUpdate(key: string, patch: Partial<Milestone>) {
  const updated = props.milestones.map((m) =>
    m.key === key ? { ...m, ...patch } : { ...m }
  )
  emit('update:milestones', updated)
}
```

**Add `TimelineDelayPreviewDialog` to the template** (just before the closing `</div>`):

```vue
<!-- Delay preview dialog (O5) -->
<TimelineDelayPreviewDialog
  v-model="delayDialogOpen"
  :milestone-key="delayingMilestoneKey ?? ''"
  :milestones="props.milestones"
  :anchor-date="anchorDate ?? new Date().toISOString().slice(0, 10)"
  @confirm="handleDelayConfirm"
/>
```

- [ ] **Step 4.4 — Run to confirm PASS**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts 2>&1 | tail -30
```

Expected: all tests pass (both O4 originals and O5 additions).

- [ ] **Step 4.5 — Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts
git commit -m "feat(onboarding): wire timeline editing — status, delay dialog, notes (O5)"
```

---

### Task 5 — Wire `updateTimeline` in the parent (call the API on milestone update)

The `update:milestones` event from `TimelineTab.vue` lands in whichever parent mounts it (`OutputTab.vue` or `AimOnboardingTab.vue` — O7 creates the shell). This task adds the API call handler so the emit results in a backend call.

**Files:**

- Modify: the parent component that mounts `TimelineTab.vue` (created by O7; stub handler if O7 is not yet merged)

- [ ] **Step 5.1 — Write the failing test**

In the parent component's test file (or create a minimal integration test if O7 is not yet merged):

```ts
// Integration test: TimelineTab → parent → onboardingService.updateTimeline
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'

const { mockUpdateTimeline } = vi.hoisted(() => ({
  mockUpdateTimeline: vi.fn(),
}))

vi.mock('../services/onboarding', () => ({
  onboardingService: {
    updateTimeline: mockUpdateTimeline,
  },
}))

import TimelineTab from '../output/TimelineTab.vue'
import type { Milestone } from '../interfaces/aimOnboarding'

const vuetify = createVuetify({ components, directives })

const SESSION_ID = 'session-abc'

function makeMilestones(): Milestone[] {
  return [
    { key: 'kickoff', name: 'Kickoff', responsible: 'Client + Kochava', weekOffset: 0, durationLabel: '90 min', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    { key: 'data_connect', name: 'Data connection', responsible: 'Client', weekOffset: 1, durationLabel: '2-3 weeks', durationWeeks: 3, status: '', delayedToDate: null, notes: '' },
  ]
}

// Minimal parent wrapper that handles update:milestones and calls updateTimeline
const ParentStub = {
  template: `
    <TimelineTab
      :milestones="milestones"
      anchorDate="2026-06-15"
      effectiveTier="aim_x"
      @update:milestones="onUpdate"
    />
  `,
  components: { TimelineTab },
  data: () => ({ milestones: makeMilestones(), sessionId: SESSION_ID }),
  methods: {
    async onUpdate(updated: Milestone[]) {
      // Find which milestone changed and what changed
      const { onboardingService } = await import('../services/onboarding')
      const orig = this.milestones as Milestone[]
      for (const m of updated) {
        const old = orig.find((o) => o.key === m.key)!
        if (m.status !== old.status) {
          await onboardingService.updateTimeline(this.sessionId, {
            MilestoneKey: m.key,
            StatusUpdate: m.status,
            DelayedToDate: m.delayedToDate,
          })
        }
        if (m.notes !== old.notes) {
          await onboardingService.updateTimeline(this.sessionId, {
            MilestoneKey: m.key,
            Notes: m.notes,
          })
        }
      }
      this.milestones = updated
    },
  },
}

describe('TimelineTab → onboardingService.updateTimeline integration', () => {
  beforeEach(() => { vi.clearAllMocks() })

  it('calls updateTimeline when status changes to in_progress', async () => {
    mockUpdateTimeline.mockResolvedValue(undefined)
    const w = mount(ParentStub, { global: { plugins: [vuetify] } })
    await flushPromises()

    const selects = w.findAllComponents({ name: 'VSelect' })
    selects[0].vm.$emit('update:modelValue', 'in_progress')
    await flushPromises()

    expect(mockUpdateTimeline).toHaveBeenCalledWith(SESSION_ID, expect.objectContaining({
      MilestoneKey: 'kickoff',
      StatusUpdate: 'in_progress',
    }))
  })

  it('calls updateTimeline with DelayedToDate when delay confirmed', async () => {
    mockUpdateTimeline.mockResolvedValue(undefined)
    const w = mount(ParentStub, { global: { plugins: [vuetify] } })
    await flushPromises()

    const selects = w.findAllComponents({ name: 'VSelect' })
    selects[0].vm.$emit('update:modelValue', 'delayed')
    await flushPromises()

    const dialog = w.findComponent({ name: 'TimelineDelayPreviewDialog' })
    dialog.vm.$emit('confirm', '2026-06-22')
    await flushPromises()

    expect(mockUpdateTimeline).toHaveBeenCalledWith(SESSION_ID, expect.objectContaining({
      MilestoneKey: 'kickoff',
      StatusUpdate: 'delayed',
      DelayedToDate: '2026-06-22',
    }))
  })
})
```

- [ ] **Step 5.2 — Run to confirm FAIL**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts 2>&1 | grep -E "FAIL|PASS|Error" | tail -20
```

The integration tests will FAIL (no parent wires the call yet).

- [ ] **Step 5.3 — Add the `update:milestones` handler to the real parent**

When O7 lands, `OutputShell.vue` (or `AimOnboardingTab.vue`) mounts `TimelineTab`. Add the handler there:

```ts
// In the parent component
import { onboardingService } from '../services/onboarding'

async function onTimelineMilestonesUpdate(updated: Milestone[]) {
  const orig = timelineState.value.milestones   // current milestones from state
  for (const m of updated) {
    const old = orig.find((o) => o.key === m.key)
    if (!old) continue
    if (m.status !== old.status || m.delayedToDate !== old.delayedToDate) {
      await onboardingService.updateTimeline(props.sessionId, {
        MilestoneKey:  m.key,
        StatusUpdate:  m.status !== old.status ? m.status : null,
        DelayedToDate: m.delayedToDate !== old.delayedToDate ? m.delayedToDate : null,
      })
    }
    if (m.notes !== old.notes) {
      await onboardingService.updateTimeline(props.sessionId, {
        MilestoneKey: m.key,
        Notes: m.notes,
      })
    }
  }
  // Update local state so cascade renders immediately (optimistic)
  timelineState.value = { ...timelineState.value, milestones: updated }
}
```

Wire it in the template:

```vue
<TimelineTab
  :milestones="timelineState.milestones"
  :anchor-date="timelineState.anchorDate"
  :effective-tier="effectiveTier"
  @update:milestones="onTimelineMilestonesUpdate"
/>
```

If O7 is not yet merged, add the `ParentStub` from Step 5.1 as a temporary integration test stand-in and skip this step until O7 lands.

- [ ] **Step 5.4 — Run full test pass**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/ 2>&1 | tail -30
```

Expected: all tests in the `AimOnboardingTab` subtree pass. Zero regressions.

- [ ] **Step 5.5 — Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/
git commit -m "feat(onboarding): wire updateTimeline API call from timeline edit events (O5)"
```

---

### Task 6 — Lint + type-check

- [ ] **Step 6.1 — Markdown lint**

```bash
cd /Users/mukey/Documents/kochava-projects/aim-onboarding-tool/docs
npx markdownlint-cli2 "docs/16-PRDs/061-aim-onboarding-decision-tool/plans/O5-timeline-editing.md"
```

Expected: no errors.

- [ ] **Step 6.2 — TypeScript type-check**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx tsc --noEmit 2>&1 | grep -i "AimOnboarding\|TimelineTab\|TimelineDelay\|onboarding" | head -20
```

Expected: no type errors in the new files.

- [ ] **Step 6.3 — Full advertiser test suite**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run 2>&1 | tail -20
```

Expected: all tests pass, zero regressions across the package.

---

## Self-Review Checklist

| Requirement | Task |
|---|---|
| Status editable by client AND CSM — no `isCsm` gate (design-consolidated §7) | Task 4 impl + test |
| Selecting "Delayed" triggers date picker → preview dialog (not an immediate emit) | Task 4 impl + test |
| Cascade preview uses `computeEstDates` locally — no API call until confirm | Task 3 impl (previewEstDates computed) |
| Preview shows old start → new start per downstream milestone (shifted highlighted) | Task 3 impl + test |
| Confirm emits date; parent calls `updateTimeline` with `StatusUpdate="delayed"` + `DelayedToDate` | Task 4 + 5 test |
| Non-delay status changes (`in_progress`, `complete`) call `updateTimeline` directly | Task 5 test |
| Notes editable per row; `updateTimeline` called with `Notes` field | Task 4 impl + test |
| Author is NEVER client-supplied — no free-text name field anywhere | No namerow in any component; author stamped server-side (B8) |
| `updateTimeline` maps to `PUT /{id}/timeline` with `TimelineEditDto` | Task 1 impl + test |
| `TimelineDelayPreviewDialog` resets picker when dialog reopens | Task 3 impl (watcher) |
| Confirm button disabled until a date is chosen | Task 2 test + Task 3 impl |
| O4 existing tests still pass after modifications | Task 4.4 full test run |
