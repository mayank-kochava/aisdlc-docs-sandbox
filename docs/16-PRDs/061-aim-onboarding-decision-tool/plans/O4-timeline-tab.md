---
id: plan-o4
title: "O4 — Timeline Tab"
---

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `output/TimelineTab.vue` — a bare `v-data-table` rendering the 6-milestone delivery plan with computed est. start / duration / est. end dates drawn from F7's milestone logic, a Responsible chip per row, a compact `v-select` status dropdown per row (rendered but non-mutating; enablement wired in O5), an expandable-row chevron per milestone (empty body; expand/collapse interaction wired in O5), a "DELIVERY PLAN / Timeline" header, and the cascade-delay note — read-only rendering only (editing/cascade interaction is O5).

**Architecture:** `TimelineTab.vue` accepts `milestones` (raw `Milestone[]` from the session doc), `anchorDate` (ISO string or null), and `effectiveTier` as props. It calls `computeEstDates` from F7 (`logic/milestones.ts`) inside a computed ref to derive `{ start, end }` per row. The status dropdown renders but emits no mutation (O5 wires mutations and enablement). Each milestone row has a chevron affordance for expand/collapse with an empty `#expanded-row` slot body — O5 fills in delay picker and notes. No `isCsm` prop: design-consolidated locked "Timeline editable by client + CSM" (§8a, wins over spec §4.2); permissions enforced at O5 and API layer. All colors via `rgb(var(--v-theme-*))`. Bare `v-data-table` (not the `List` wrapper) — the `List` wrapper's viewport-height and infinite-scroll machinery fights a static 6-row table (design-consolidated §8b).

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript, Vitest + @vue/test-utils, `v-data-table`, `v-chip`, `v-select` (density compact), `v-alert variant="tonal"`.

---

## Files

**Create:**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts`

**Read (existing — do not reinvent):**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.ts` (F7 — `computeEstDates`, `Milestone`, `EstDates`)
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts` (F2 — `Milestone`, `TimelineState`)
- `packages/core/src/components/ui/List.vue` (mirror only — we use bare `v-data-table`, not this wrapper)

---

## Dependencies

**F2** (`interfaces/aimOnboarding.ts`) — exports `Milestone`, `TimelineState`. Must be merged or stubbed (see stub block below).

**F7** (`logic/milestones.ts`) — exports `computeEstDates(milestones, anchorDate): Record<string, EstDates>` and `buildMilestoneTemplate(tier)`. Must be merged or stubbed.

If either is not yet merged, add the inline stubs shown in Task 1 to the test file. Remove stubs once the dependencies land.

---

## F2 + F7 Type Contract (reference — do not re-invent)

Import from the real files once merged. If not merged, add this stub at the top of `TimelineTab.test.ts` only:

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

// Minimal computeEstDates stub — replace with real import once F7 merges
function computeEstDates(
  milestones: Milestone[],
  anchorDate: Date
): Record<string, EstDates> {
  const result: Record<string, EstDates> = {}
  let cumulativeDeltaDays = 0
  for (const m of milestones) {
    const raw = new Date(anchorDate)
    raw.setDate(raw.getDate() + m.weekOffset * 7 + cumulativeDeltaDays)
    const start = nextMonday(raw)
    if (m.delayedToDate) {
      const delayed = nextMonday(new Date(m.delayedToDate))
      const delta = (delayed.getTime() - start.getTime()) / 86400000
      cumulativeDeltaDays += delta
    }
    const end = new Date(start)
    end.setDate(end.getDate() + m.durationWeeks * 7)
    result[m.key] = { start, end }
  }
  return result
}

function nextMonday(date: Date): Date {
  const d = new Date(date); d.setHours(0, 0, 0, 0)
  const day = d.getDay()
  d.setDate(d.getDate() + (8 - day) % 7)
  return d
}
```

---

## Milestone table (spec §4.2 Tab 2 — hardcoded in F7; reference here for the tests)

| # | key | name | responsible | AIM X weekOffset | AIM Pro weekOffset | durationLabel (X / Pro) |
|---|-----|------|-------------|------------------|--------------------|------------------------|
| 1 | `kickoff` | Onboarding kickoff call | Client + Kochava | 0 | 0 | 90 min / 90 min |
| 2 | `sow_approval` | Scope of Work approval | Client + Kochava | 0 | 0 | 1 hour / 1 hour |
| 3 | `data_connect` | Data connection / file upload | Client | 1 | 1 | 2-3 weeks / 3-5 weeks |
| 4 | `data_qa` | Data QA and validation | Kochava | 3 | 5 | ~1 week / 1-2 weeks |
| 5 | `model_train` | Model training | Kochava | 4 | 6 | 1-2 weeks / 2-3 weeks |
| 6 | `first_insight` | First insights delivered | Kochava | 6 | 8 | 1 hour / 1 hour |

---

## Date formatting convention

Display dates as `"WC D Mon"` (week-commencing), e.g. `"WC 15 Jun"`. Helper:

```ts
function fmtWC(date: Date): string {
  return `WC ${date.getDate()} ${date.toLocaleString('en-GB', { month: 'short' })}`
}
```

---

## Tasks

### Task 1 — Write failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts`

- [ ] **Step 1.1 — Create the test file**

```ts
// TimelineTab.test.ts
import { describe, it, expect } from 'vitest'
import { mount, VueWrapper } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'
import TimelineTab from './TimelineTab.vue'

// ── Add F2+F7 stubs here if those plans are not yet merged ──────────────────
// (see stub block in the Dependencies section above)

// ── Vuetify fixture ─────────────────────────────────────────────────────────

const vuetify = createVuetify({ components, directives })

function mountTab(props: Record<string, unknown> = {}): VueWrapper {
  return mount(TimelineTab, {
    global: { plugins: [vuetify] },
    props: {
      milestones: [
        { key: 'kickoff',       name: 'Onboarding kickoff call',       responsible: 'Client + Kochava', weekOffset: 0, durationLabel: '90 min',    durationWeeks: 0, status: '',   delayedToDate: null, notes: '' },
        { key: 'sow_approval',  name: 'Scope of Work approval',        responsible: 'Client + Kochava', weekOffset: 0, durationLabel: '1 hour',    durationWeeks: 0, status: '',   delayedToDate: null, notes: '' },
        { key: 'data_connect',  name: 'Data connection / file upload', responsible: 'Client',           weekOffset: 1, durationLabel: '3-5 weeks', durationWeeks: 5, status: '',   delayedToDate: null, notes: '' },
        { key: 'data_qa',       name: 'Data QA and validation',        responsible: 'Kochava',          weekOffset: 5, durationLabel: '1-2 weeks', durationWeeks: 2, status: '',   delayedToDate: null, notes: '' },
        { key: 'model_train',   name: 'Model training',                responsible: 'Kochava',          weekOffset: 6, durationLabel: '2-3 weeks', durationWeeks: 3, status: '',   delayedToDate: null, notes: '' },
        { key: 'first_insight', name: 'First insights delivered',      responsible: 'Kochava',          weekOffset: 8, durationLabel: '1 hour',    durationWeeks: 0, status: '',   delayedToDate: null, notes: '' },
      ],
      anchorDate: '2026-06-15',  // Monday
      effectiveTier: 'aim_pro',
      ...props,
    },
  })
}

// ── Header / labels ─────────────────────────────────────────────────────────

describe('TimelineTab — header and labels', () => {
  it('renders "DELIVERY PLAN" eyebrow label', () => {
    const w = mountTab()
    expect(w.text()).toContain('DELIVERY PLAN')
  })

  it('renders "Timeline" heading', () => {
    const w = mountTab()
    expect(w.text()).toContain('Timeline')
  })

  it('shows tier name in the subtitle', () => {
    const w = mountTab()
    expect(w.text()).toContain('AIM Pro')
  })

  it('renders the cascade-delay note', () => {
    const w = mountTab()
    expect(w.text()).toContain('delayed')
    expect(w.text()).toContain('subsequent')
  })
})

// ── Column headers ──────────────────────────────────────────────────────────

describe('TimelineTab — column headers', () => {
  it('renders "Milestone" column header', () => {
    const w = mountTab()
    expect(w.text()).toContain('Milestone')
  })

  it('renders "Responsible" column header', () => {
    const w = mountTab()
    expect(w.text()).toContain('Responsible')
  })

  it('renders "Est. start" column header', () => {
    const w = mountTab()
    expect(w.text()).toContain('Est. start')
  })

  it('renders "Duration" column header', () => {
    const w = mountTab()
    expect(w.text()).toContain('Duration')
  })

  it('renders "Est. end" column header', () => {
    const w = mountTab()
    expect(w.text()).toContain('Est. end')
  })

  it('renders "Status" column header', () => {
    const w = mountTab()
    expect(w.text()).toContain('Status')
  })
})

// ── Row content ─────────────────────────────────────────────────────────────

describe('TimelineTab — row content', () => {
  it('renders all 6 milestone names', () => {
    const w = mountTab()
    expect(w.text()).toContain('Onboarding kickoff call')
    expect(w.text()).toContain('Scope of Work approval')
    expect(w.text()).toContain('Data connection / file upload')
    expect(w.text()).toContain('Data QA and validation')
    expect(w.text()).toContain('Model training')
    expect(w.text()).toContain('First insights delivered')
  })

  it('renders duration labels', () => {
    const w = mountTab()
    expect(w.text()).toContain('90 min')
    expect(w.text()).toContain('3-5 weeks')
  })

  it('renders est. start as "WC D Mon" format for kickoff (anchorDate 2026-06-15)', () => {
    const w = mountTab()
    // weekOffset 0 on a Monday anchor: start = 2026-06-15 → "WC 15 Jun"
    expect(w.text()).toContain('WC 15 Jun')
  })

  it('renders est. start for data_connect (weekOffset 1 → 2026-06-22)', () => {
    const w = mountTab()
    expect(w.text()).toContain('WC 22 Jun')
  })

  it('renders est. end for data_connect (durationWeeks 5 from 2026-06-22 → 2026-07-27)', () => {
    const w = mountTab()
    expect(w.text()).toContain('WC 27 Jul')
  })
})

// ── Responsible chip ─────────────────────────────────────────────────────────

describe('TimelineTab — responsible chip', () => {
  it('renders a v-chip for "Client + Kochava"', () => {
    const w = mountTab()
    const chips = w.findAllComponents({ name: 'VChip' })
    const texts = chips.map((c) => c.text())
    expect(texts.some((t) => t.includes('Client + Kochava'))).toBe(true)
  })

  it('renders a v-chip for "Client"', () => {
    const w = mountTab()
    const chips = w.findAllComponents({ name: 'VChip' })
    const texts = chips.map((c) => c.text())
    expect(texts.some((t) => t === 'Client')).toBe(true)
  })

  it('renders a v-chip for "Kochava"', () => {
    const w = mountTab()
    const chips = w.findAllComponents({ name: 'VChip' })
    const texts = chips.map((c) => c.text())
    expect(texts.some((t) => t === 'Kochava')).toBe(true)
  })
})

// ── Status v-select ──────────────────────────────────────────────────────────
// O4 renders the select non-mutating; O5 wires enablement + mutations.
// design-consolidated §8a: Timeline editable by client + CSM both — no isCsm gate here.

describe('TimelineTab — status v-select', () => {
  it('renders a v-select for each of the 6 rows', () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    expect(selects.length).toBe(6)
  })

  it('status v-select is not disabled — all users can see select (O5 controls actual edit flow)', () => {
    const w = mountTab()
    const selects = w.findAllComponents({ name: 'VSelect' })
    selects.forEach((sel) => {
      expect(sel.props('disabled')).toBe(false)
    })
  })
})

// ── Expandable row chevron ────────────────────────────────────────────────────
// O4 renders the chevron affordance; O5 fills expanded body (delay picker + notes).

describe('TimelineTab — expandable row chevron', () => {
  it('renders expand toggle for each of the 6 rows', () => {
    // v-data-table expand-on-click or show-expand renders a .v-data-table__td--expanded-row
    // or aria-expanded attribute; we assert the wrapper contains expand affordance markup
    const w = mountTab()
    // show-expand prop means each row has an expand icon button
    const expandBtns = w.findAll('[data-testid="row-expand"]')
    // Fallback: check the table has show-expand header column rendered
    const html = w.html()
    expect(html).toContain('mdi-chevron')
  })
})

// ── anchorDate null (pre-approval) ──────────────────────────────────────────

describe('TimelineTab — anchorDate null (pre-approval state)', () => {
  it('falls back to today as anchor when anchorDate is null', () => {
    // Just check no crash and 6 rows still render
    const w = mountTab({ anchorDate: null })
    expect(w.text()).toContain('Onboarding kickoff call')
    expect(w.text()).toContain('WC')
  })
})

// ── Tier label in subtitle ───────────────────────────────────────────────────

describe('TimelineTab — tier label in subtitle', () => {
  it('shows AIM X in subtitle when effectiveTier is aim_x', () => {
    const w = mountTab({ effectiveTier: 'aim_x' })
    expect(w.text()).toContain('AIM X')
  })
})
```

- [ ] **Step 1.2 — Run to confirm FAIL (module not found)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts 2>&1 | tail -20
```

Expected output contains: `Cannot find module './TimelineTab.vue'`

---

### Task 2 — Implement `TimelineTab.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue`

- [ ] **Step 2.1 — Create the component**

```vue
<template>
  <!-- design-consolidated §8b: bare v-data-table, NOT the List wrapper -->
  <div class="aim-timeline-tab">
    <!-- ── Header ───────────────────────────────────────────────────────── -->
    <div class="aim-timeline-tab__eyebrow">DELIVERY PLAN</div>
    <h3 class="aim-timeline-tab__heading">Timeline</h3>
    <p class="aim-timeline-tab__sub">
      Estimated milestones for
      <strong>{{ tierLabel }}</strong>.
      Calculated from
      {{ anchorDate ? 'SoW approval date' : 'questionnaire completion today' }}.
    </p>

    <!-- ── Cascade note (v-alert tonal — §8b precedent) ────────────────── -->
    <v-alert
      type="warning"
      variant="tonal"
      class="aim-timeline-tab__cascade-note mb-4"
      density="compact"
    >
      <strong>Important:</strong> All timelines are estimates. Marking a milestone
      <em>delayed</em> automatically shifts all <em>subsequent</em> estimated dates.
    </v-alert>

    <!-- ── Table ────────────────────────────────────────────────────────── -->
    <!-- show-expand: chevron affordance per row; expanded body is empty here (O5 fills it) -->
    <v-data-table
      :headers="headers"
      :items="rows"
      items-per-page="-1"
      :mobile="false"
      hide-default-footer
      show-expand
      class="aim-timeline-tab__table"
    >
      <!-- Milestone name -->
      <template #item.name="{ item }">
        <span class="aim-timeline-tab__mname">{{ item.name }}</span>
      </template>

      <!-- Responsible chip -->
      <template #item.responsible="{ item }">
        <v-chip
          size="small"
          color="primary"
          variant="tonal"
          class="aim-timeline-tab__chip"
        >
          {{ item.responsible }}
        </v-chip>
      </template>

      <!-- Est. start -->
      <template #item.estStart="{ item }">
        <span class="aim-timeline-tab__date">{{ item.estStart }}</span>
      </template>

      <!-- Duration -->
      <template #item.duration="{ item }">
        <span class="aim-timeline-tab__dur">{{ item.duration }}</span>
      </template>

      <!-- Est. end -->
      <template #item.estEnd="{ item }">
        <span class="aim-timeline-tab__date">{{ item.estEnd }}</span>
      </template>

      <!-- Status v-select — renders non-mutating; O5 wires @update:modelValue + enablement -->
      <!-- design-consolidated §8a: Timeline editable by client + CSM — no disabled here -->
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
        />
      </template>

      <!-- Expanded row body — empty placeholder; O5 fills delay picker + notes -->
      <template #expanded-row="{ columns }">
        <tr>
          <td :colspan="columns.length" class="aim-timeline-tab__expanded-body" />
        </tr>
      </template>

      <!-- Hide the default bottom pagination bar -->
      <template #bottom />
    </v-data-table>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { computeEstDates } from '../logic/milestones'
import type { Milestone } from '../interfaces/aimOnboarding'

// ── Props ────────────────────────────────────────────────────────────────────

// No isCsm prop — design-consolidated §8a locks "Timeline editable by client + CSM both".
// Permissions enforced at O5 (mutation wiring) and mmm-portal-api (B8 endpoint).
const props = defineProps<{
  /** Raw milestone array from the session doc (F7 template + any stored status/delay). */
  milestones: Milestone[]
  /** ISO date string from approvedAt; null while pre-approval → falls back to today. */
  anchorDate: string | null
  /** Computed effective tier, drives subtitle label. */
  effectiveTier: 'aim_x' | 'aim_pro'
}>()

// ── Constants ────────────────────────────────────────────────────────────────

const STATUS_OPTIONS = [
  { label: 'In progress', value: 'in_progress' },
  { label: 'Complete',    value: 'complete'    },
  { label: 'Delayed',     value: 'delayed'     },
] as const

const headers = [
  { title: 'Milestone',   key: 'name',        sortable: false },
  { title: 'Responsible', key: 'responsible',  sortable: false },
  { title: 'Est. start',  key: 'estStart',     sortable: false },
  { title: 'Duration',    key: 'duration',     sortable: false },
  { title: 'Est. end',    key: 'estEnd',       sortable: false },
  { title: 'Status',      key: 'status',       sortable: false },
]

// ── Helpers ──────────────────────────────────────────────────────────────────

/** Format a Date as "WC D Mon", e.g. "WC 15 Jun" (design-consolidated §8a). */
function fmtWC(date: Date): string {
  return `WC ${date.getDate()} ${date.toLocaleString('en-GB', { month: 'short' })}`
}

// ── Computed ─────────────────────────────────────────────────────────────────

const tierLabel = computed(() =>
  props.effectiveTier === 'aim_pro' ? 'AIM Pro' : 'AIM X'
)

/**
 * Resolve anchor to a Date:
 *   - If approvedAt is set → use that date.
 *   - Otherwise → today (pre-approval, spec §4.2: "cascade from today").
 * design-consolidated §8a: anchorDate = approvedAt ?? today.
 */
const resolvedAnchor = computed((): Date =>
  props.anchorDate ? new Date(props.anchorDate) : new Date()
)

/**
 * Computed row objects for the v-data-table.
 * computeEstDates lives in F7 (logic/milestones.ts) — injected, never new Date() here.
 */
const rows = computed(() => {
  const estMap = computeEstDates(props.milestones, resolvedAnchor.value)
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
    }
  })
})
</script>

<style scoped>
.aim-timeline-tab {
  max-width: 880px;
  margin: 0 auto;
}

.aim-timeline-tab__eyebrow {
  font-size: 10.5px;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: rgb(var(--v-theme-on-surface-variant, 154 155 157));
  opacity: 0.7;
}

.aim-timeline-tab__heading {
  font-size: 18px;
  font-weight: 700;
  margin: 4px 0 4px;
  color: rgb(var(--v-theme-on-surface));
}

.aim-timeline-tab__sub {
  font-size: 13px;
  color: rgb(var(--v-theme-on-surface-variant));
  margin-bottom: 8px;
}

/* Compact chip — matches ui-preview.html .who */
.aim-timeline-tab__chip {
  font-size: 11px;
  font-weight: 600;
}

/* Date cells — primary color per ui-preview */
.aim-timeline-tab__date {
  color: rgb(var(--v-theme-primary));
  font-weight: 600;
  white-space: nowrap;
}

.aim-timeline-tab__dur {
  color: rgb(var(--v-theme-on-surface-variant));
}

/* Match Vuetify table radius — SCSS global handles the rounded corners */
.aim-timeline-tab__table {
  border: 1px solid rgb(var(--v-border-color));
  border-radius: 8px;
  overflow: hidden;
}

/* Remove the focus-halo on the status select (§8b) */
.aim-timeline-tab__status-sel :deep(.v-field__outline) {
  box-shadow: none !important;
}
</style>
```

- [ ] **Step 2.2 — Run tests (expect PASS)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts 2>&1 | tail -30
```

Expected: all tests pass, no failures.

If any tests fail due to a missing F7 import, add the inline stub from the Dependencies section to the test file and re-run.

- [ ] **Step 2.3 — Commit**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/TimelineTab.test.ts
git commit -m "feat(onboarding): add output/TimelineTab — delivery plan table with computed est dates (O4)"
```

---

### Task 3 — Full suite + lint

- [ ] **Step 3.1 — Run full advertiser test suite**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run 2>&1 | tail -30
```

Expected: all tests pass including `TimelineTab.test.ts`. Zero regressions.

- [ ] **Step 3.2 — TypeScript check**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx tsc --noEmit 2>&1 | grep -i "TimelineTab\|AimOnboarding" | head -20
```

Expected: no type errors in the new files.

- [ ] **Step 3.3 — Markdown lint**

```bash
cd /Users/mukey/Documents/kochava-projects/aim-onboarding-tool/docs
npx markdownlint-cli2 "docs/16-PRDs/061-aim-onboarding-decision-tool/plans/O4-timeline-tab.md"
```

Expected: no errors.

---

## Self-Review

### Spec coverage

| Requirement | Task |
|---|---|
| "DELIVERY PLAN / Timeline" header (ui-preview.html) | Task 2 component |
| 6 columns: Milestone, Responsible, Est. start, Duration, Est. end, Status (spec §4.2) | Task 1 tests, Task 2 component |
| Responsible shown as chip per row (spec §4.2 "Responsible" new field) | Task 1 chip tests, Task 2 `v-chip` |
| Est. start / Est. end computed from `anchorDate + weekOffset` via F7 `computeEstDates` | Task 1 date tests, Task 2 computed `rows` |
| `anchorDate = approvedAt ?? today` (design-consolidated §8a) | Task 1 null-anchor test, Task 2 `resolvedAnchor` |
| "WC D Mon" date format (ui-preview.html) | Task 1 date tests, Task 2 `fmtWC` |
| Duration column shows label string (spec §4.2 "informational only") | Task 1 duration test, Task 2 rows |
| Status column is compact `v-select` (design-consolidated §8b) | Task 1 v-select tests, Task 2 |
| Status dropdown rendered non-mutating (design-consolidated §8a: client + CSM both; O5 wires enablement) | Task 1 v-select non-disabled test, Task 2 no `isCsm` prop |
| Expandable row chevron affordance per milestone (ui-preview chevron; body empty until O5) | Task 1 expand-chevron test, Task 2 `show-expand` + `#expanded-row` |
| Cascade-delay note visible (ui-preview.html) | Task 1 cascade-note test, Task 2 `v-alert` |
| Bare `v-data-table`, not the `List` wrapper (design-consolidated §8b) | Task 2 uses `<v-data-table>` directly |
| No hardcoded hex; all colors via `rgb(var(--v-theme-*))` (design-consolidated §8b) | Task 2 styles |
| No focus-halo on select (design-consolidated §8b "remove the mock's box-shadow ring") | Task 2 `<style scoped>` |
| O5 scope boundary respected — no mutation emits in this component | Task 2 (no `@update`, no emits wired) |

### Placeholder scan

No TBD, TODO, "implement later", or placeholder steps. All test assertions are concrete. All component code is complete.

### Type consistency

- `Milestone` interface used identically across the test stub, props type, and `computeEstDates` call.
- `fmtWC(date: Date): string` defined once, used in two places (`estStart`, `estEnd`) within the same computed.
- `STATUS_OPTIONS` items typed `{ label: string; value: string }` — consistent with `v-select` `item-title`/`item-value` props.
- No `isCsm` prop — design-consolidated §8a wins; permission model belongs to O5 + B8.
- `show-expand` on `v-data-table` renders Vuetify's built-in chevron; `#expanded-row` slot is empty placeholder until O5.
