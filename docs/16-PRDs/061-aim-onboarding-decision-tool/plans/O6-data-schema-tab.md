---
id: plan-o6
title: "O6 — Data Schema Tab"
---

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `output/DataSchemaTab.vue` — the "DATA PROVISION / What Kochava collects vs what you provide" tab (Tab 3 of the output screen) — consuming the `ProvisionPlan` returned by `buildProvisionPlan` (F6) and rendering: a green all-clear banner when every source is direct API, an auto-collected card, and one you-provide card per manual source with a schema column table and a CSV template download button.

**Architecture:** A single `.vue` SFC receiving a `ProvisionPlan` prop (typed from F6). No internal data fetching, no store access. CSV download is a thin side-effect wrapper around a pure `columnsToCsv(columns, filename)` helper (co-located in a `csvDownload.ts` utility) — the helper is unit-tested independently; the Blob/anchor click is tested by spying on `URL.createObjectURL`. The component test mounts with the global Vuetify plugin already configured by `tests.config.ts` and stubs Vuetify elements that have scoped-slot complexity (`v-table` is rendered real; `v-alert` is rendered real — no stubs needed for flat components).

**Tech Stack:** Vue 3.5 (Composition API, `<script setup>`), Vuetify 3.12 (`v-alert`, `v-table`, `v-btn`, `v-icon`), TypeScript, Vitest + `@vue/test-utils` (jsdom).

---

## Files

**Create**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.vue`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.test.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.test.ts`

**Read (F6 contract — must exist before Task 1)**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.ts`

---

## Dependencies

**F6** (`logic/buildProvisionPlan.ts`) must be merged and export `ProvisionPlan`, `AutoRow`, and `ManualRow`. These types are the sole prop contract for this component.

If F6 has not landed yet, copy the type stubs from the F6 plan (Task 2 stub) inline at the top of the test file and remove them once F6 merges.

---

## Open Item

> **OPEN — Gary / Data Engineering:** Exact CSV column sets per source are **not yet confirmed**.
> `buildProvisionPlan` (F6) uses placeholder columns in `COLUMN_REGISTRY`. The `DataSchemaTab`
> renders whatever columns F6 supplies — no hardcoded column knowledge here. Once Gary confirms the
> final columns, update `COLUMN_REGISTRY` in F6 (`buildProvisionPlan.ts`) only. No change to this
> component or its tests is required.

---

## F6 Type Contract (reference)

```ts
// From logic/buildProvisionPlan.ts — import, do NOT re-declare
export interface AutoRow {
  source: string
  label: string
  note: string
}

export interface ManualRow {
  source: string
  label: string
  columns: string[]
}

export interface ProvisionPlan {
  auto: AutoRow[]
  manual: ManualRow[]
  allAuto: boolean
}
```

---

## Tasks

### Task 1 — Write the failing tests for `csvDownload.ts`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.test.ts`

- [ ] **Step 1: Create the test file**

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { columnsToCsv, downloadCsv } from './csvDownload'

describe('columnsToCsv', () => {
  it('returns a header-only CSV row for a single column', () => {
    expect(columnsToCsv(['date'])).toBe('date')
  })

  it('joins multiple columns with commas', () => {
    expect(columnsToCsv(['date', 'channel', 'spend'])).toBe('date,channel,spend')
  })

  it('returns empty string for empty array', () => {
    expect(columnsToCsv([])).toBe('')
  })
})

describe('downloadCsv', () => {
  let createObjectURLSpy: ReturnType<typeof vi.spyOn>
  let revokeObjectURLSpy: ReturnType<typeof vi.spyOn>
  let appendChildSpy: ReturnType<typeof vi.spyOn>
  let removeChildSpy: ReturnType<typeof vi.spyOn>
  let clickSpy: ReturnType<typeof vi.fn>

  beforeEach(() => {
    clickSpy = vi.fn()
    // jsdom does not implement createObjectURL — stub it
    createObjectURLSpy = vi.spyOn(URL, 'createObjectURL').mockReturnValue('blob:mock-url')
    revokeObjectURLSpy = vi.spyOn(URL, 'revokeObjectURL').mockImplementation(() => undefined)
    // Intercept anchor creation so we can verify download attribute
    vi.spyOn(document, 'createElement').mockImplementation((tag: string) => {
      if (tag === 'a') {
        const anchor = Object.assign(document.createElementNS('http://www.w3.org/1999/xhtml', 'a') as HTMLAnchorElement, {
          click: clickSpy,
        })
        return anchor
      }
      return document.createElementNS('http://www.w3.org/1999/xhtml', tag) as HTMLElement
    })
    appendChildSpy = vi.spyOn(document.body, 'appendChild').mockImplementation((node) => node)
    removeChildSpy = vi.spyOn(document.body, 'removeChild').mockImplementation((node) => node)
  })

  afterEach(() => {
    vi.restoreAllMocks()
  })

  it('calls createObjectURL with a Blob of type text/csv', () => {
    downloadCsv('date,channel,spend', 'ad-spend.csv')
    expect(createObjectURLSpy).toHaveBeenCalledOnce()
    const blob = createObjectURLSpy.mock.calls[0][0] as Blob
    expect(blob.type).toBe('text/csv')
  })

  it('triggers a click on the anchor element', () => {
    downloadCsv('date,channel,spend', 'ad-spend.csv')
    expect(clickSpy).toHaveBeenCalledOnce()
  })

  it('calls revokeObjectURL after click', () => {
    downloadCsv('date,channel,spend', 'ad-spend.csv')
    expect(revokeObjectURLSpy).toHaveBeenCalledWith('blob:mock-url')
  })
})
```

- [ ] **Step 2: Run the test to verify it fails (module not found)**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.test.ts 2>&1 | tail -20
```

Expected: `Error: Cannot find module './csvDownload'`

---

### Task 2 — Implement `csvDownload.ts`, make tests pass

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.ts`

- [ ] **Step 1: Create the implementation**

```ts
/**
 * Pure helper — converts an array of column-name strings to a single CSV header line.
 * Tests assert this function directly; no DOM side-effects.
 */
export function columnsToCsv(columns: string[]): string {
  return columns.join(',')
}

/**
 * Thin side-effect wrapper — creates a Blob from csvContent and triggers a browser download.
 * Not unit-tested beyond spy verification; logic lives in columnsToCsv.
 */
export function downloadCsv(csvContent: string, filename: string): void {
  const blob = new Blob([csvContent], { type: 'text/csv' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}
```

- [ ] **Step 2: Run the csvDownload tests — expect all pass**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.test.ts 2>&1 | tail -20
```

Expected: all tests pass, no failures.

- [ ] **Step 3: Commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.test.ts
git commit -m "feat(onboarding): add csvDownload helper + Vitest (O6)"
```

---

### Task 3 — Write the failing component tests for `DataSchemaTab.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.test.ts`

Note: `tests.config.ts` globally registers `[vuetify, pinia, i18n]` on `config.global.plugins` — no per-test Vuetify setup needed. `URL.createObjectURL` must be spied per-test (not globally mocked).

- [ ] **Step 1: Create the test file**

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { mount } from '@vue/test-utils'
import DataSchemaTab from './DataSchemaTab.vue'
import type { ProvisionPlan } from '../logic/buildProvisionPlan'

// ── Factory helpers ──────────────────────────────────────────────────────────

function allAutoplan(): ProvisionPlan {
  return {
    allAuto: true,
    auto: [
      { source: 'mmp', label: 'AppsFlyer (MMP)', note: 'Direct API' },
      { source: 'ad_spend', label: 'Ad spend', note: 'Direct API' },
    ],
    manual: [],
  }
}

function mixedPlan(): ProvisionPlan {
  return {
    allAuto: false,
    auto: [
      { source: 'mmp', label: 'AppsFlyer (MMP)', note: 'Direct API' },
    ],
    manual: [
      { source: 'ad_spend', label: 'Ad spend data', columns: ['date', 'channel', 'spend'] },
      { source: 'offline_media', label: 'Offline media spend', columns: ['date', 'channel', 'market', 'spend'] },
    ],
  }
}

function manualOnlyPlan(): ProvisionPlan {
  return {
    allAuto: false,
    auto: [],
    manual: [
      { source: 'ad_spend', label: 'Ad spend data', columns: ['date', 'channel', 'spend'] },
    ],
  }
}

// ── Tests ────────────────────────────────────────────────────────────────────

describe('DataSchemaTab', () => {
  describe('allAuto state', () => {
    it('renders the green all-clear alert when allAuto is true', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: allAutoplan() } })
      const alert = wrapper.find('[data-testid="all-auto-alert"]')
      expect(alert.exists()).toBe(true)
      expect(alert.text()).toContain('No file uploads needed')
    })

    it('does not render the you-provide section when allAuto is true', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: allAutoplan() } })
      expect(wrapper.find('[data-testid="manual-section"]').exists()).toBe(false)
    })

    it('still renders the auto-collected card when allAuto is true', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: allAutoplan() } })
      expect(wrapper.find('[data-testid="auto-collected-card"]').exists()).toBe(true)
    })
  })

  describe('auto-collected card', () => {
    it('renders a row for each auto source', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      const rows = wrapper.findAll('[data-testid="auto-row"]')
      expect(rows).toHaveLength(1) // mixedPlan has 1 auto source
    })

    it('shows the auto row label and Direct API note', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      const row = wrapper.find('[data-testid="auto-row"]')
      expect(row.text()).toContain('AppsFlyer (MMP)')
      expect(row.text()).toContain('Direct API')
    })

    it('does not render the auto-collected card when auto array is empty', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: manualOnlyPlan() } })
      expect(wrapper.find('[data-testid="auto-collected-card"]').exists()).toBe(false)
    })
  })

  describe('you-provide cards', () => {
    it('renders one card per manual source', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      const cards = wrapper.findAll('[data-testid="manual-card"]')
      expect(cards).toHaveLength(2)
    })

    it('shows the manual source label as the card heading', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      const firstCard = wrapper.findAll('[data-testid="manual-card"]')[0]
      expect(firstCard.text()).toContain('Ad spend data')
    })

    it('renders a v-table with one th per column', async () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      const firstCard = wrapper.findAll('[data-testid="manual-card"]')[0]
      const headers = firstCard.findAll('th')
      // ad_spend columns: date, channel, spend
      expect(headers).toHaveLength(3)
      expect(headers[0].text()).toBe('date')
      expect(headers[1].text()).toBe('channel')
      expect(headers[2].text()).toBe('spend')
    })

    it('shows a "File upload required" badge on each manual card', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      const badges = wrapper.findAll('[data-testid="upload-badge"]')
      expect(badges).toHaveLength(2)
    })

    it('renders a CSV download button on each manual card', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      const buttons = wrapper.findAll('[data-testid="csv-download-btn"]')
      expect(buttons).toHaveLength(2)
    })
  })

  describe('CSV download', () => {
    let createObjectURLSpy: ReturnType<typeof vi.spyOn>
    let revokeObjectURLSpy: ReturnType<typeof vi.spyOn>

    beforeEach(() => {
      createObjectURLSpy = vi.spyOn(URL, 'createObjectURL').mockReturnValue('blob:mock')
      revokeObjectURLSpy = vi.spyOn(URL, 'revokeObjectURL').mockImplementation(() => undefined)
      vi.spyOn(document, 'createElement').mockImplementation((tag: string) => {
        if (tag === 'a') {
          return Object.assign(
            document.createElementNS('http://www.w3.org/1999/xhtml', 'a') as HTMLAnchorElement,
            { click: vi.fn() },
          )
        }
        return document.createElementNS('http://www.w3.org/1999/xhtml', tag) as HTMLElement
      })
      vi.spyOn(document.body, 'appendChild').mockImplementation((node) => node)
      vi.spyOn(document.body, 'removeChild').mockImplementation((node) => node)
    })

    afterEach(() => {
      vi.restoreAllMocks()
    })

    it('calls createObjectURL when CSV download button is clicked', async () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      await wrapper.find('[data-testid="csv-download-btn"]').trigger('click')
      expect(createObjectURLSpy).toHaveBeenCalledOnce()
    })

    it('calls revokeObjectURL after download click', async () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      await wrapper.find('[data-testid="csv-download-btn"]').trigger('click')
      expect(revokeObjectURLSpy).toHaveBeenCalledWith('blob:mock')
    })

    it('passes a Blob with type text/csv', async () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      await wrapper.find('[data-testid="csv-download-btn"]').trigger('click')
      const blob = createObjectURLSpy.mock.calls[0][0] as Blob
      expect(blob.type).toBe('text/csv')
    })
  })

  describe('does not render the all-clear alert when manual sources exist', () => {
    it('hides the all-auto-alert when allAuto is false', () => {
      const wrapper = mount(DataSchemaTab, { props: { plan: mixedPlan() } })
      expect(wrapper.find('[data-testid="all-auto-alert"]').exists()).toBe(false)
    })
  })
})
```

- [ ] **Step 2: Run the test to verify it fails (module not found)**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.test.ts 2>&1 | tail -20
```

Expected: `Error: Cannot find module './DataSchemaTab.vue'`

---

### Task 4 — Implement `DataSchemaTab.vue`, make tests pass

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.vue`

- [ ] **Step 1: Create the component**

```vue
<template>
  <div class="aim-data-schema-tab">

    <!-- ── All-auto green banner ─────────────────────────────────────────── -->
    <v-alert
      v-if="plan.allAuto"
      data-testid="all-auto-alert"
      type="success"
      variant="tonal"
      class="mb-5"
      :icon="false"
    >
      <div class="d-flex align-center ga-2">
        <v-icon icon="mdi-check-circle-outline" />
        <span class="font-weight-medium">No file uploads needed</span>
      </div>
      <div class="mt-1 text-body-2">
        All your data sources are connected via Direct API — Kochava collects everything
        automatically.
      </div>
    </v-alert>

    <!-- ── Auto-collected card ───────────────────────────────────────────── -->
    <div
      v-if="plan.auto.length > 0"
      data-testid="auto-collected-card"
      class="aim-schema-card mb-4"
    >
      <div class="aim-schema-card__header">
        <v-icon icon="mdi-cloud-check-outline" color="success" size="18" />
        <span class="text-body-2 font-weight-semibold">Kochava collects automatically</span>
      </div>
      <div
        v-for="row in plan.auto"
        :key="row.source"
        data-testid="auto-row"
        class="aim-schema-card__auto-row"
      >
        <span class="text-body-2">{{ row.label }}</span>
        <span class="aim-schema-card__api-note">
          <v-icon icon="mdi-api" size="14" />
          {{ row.note }}
        </span>
      </div>
    </div>

    <!-- ── You-provide cards ─────────────────────────────────────────────── -->
    <template v-if="plan.manual.length > 0">
      <div data-testid="manual-section">
        <div
          v-for="row in plan.manual"
          :key="row.source"
          data-testid="manual-card"
          class="aim-schema-card mb-4"
        >
          <!-- Card header: label + badges + download button -->
          <div class="aim-schema-card__header aim-schema-card__header--space-between">
            <span class="text-body-2 font-weight-semibold">{{ row.label }}</span>
            <div class="d-flex align-center ga-3">
              <span
                data-testid="upload-badge"
                class="aim-schema-card__upload-badge"
              >
                <v-icon icon="mdi-tray-arrow-up" size="14" />
                File upload required
              </span>
              <v-btn
                data-testid="csv-download-btn"
                variant="outlined"
                density="compact"
                size="small"
                prepend-icon="mdi-download"
                @click="handleDownload(row)"
              >
                CSV template
              </v-btn>
            </div>
          </div>

          <!-- Schema column table -->
          <v-table
            class="aim-schema-table mt-3"
            density="compact"
          >
            <thead>
              <tr>
                <th
                  v-for="col in row.columns"
                  :key="col"
                  class="text-left"
                >
                  {{ col }}
                </th>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td
                  v-for="col in row.columns"
                  :key="col"
                  class="text-medium-emphasis text-body-2"
                >
                  <!-- Sample value placeholder shown as em dash until Gary confirms columns -->
                  —
                </td>
              </tr>
            </tbody>
          </v-table>
        </div>
      </div>
    </template>

  </div>
</template>

<script setup lang="ts">
import { columnsToCsv, downloadCsv } from './csvDownload'
import type { ProvisionPlan, ManualRow } from '../logic/buildProvisionPlan'

const props = defineProps<{
  plan: ProvisionPlan
}>()

function handleDownload(row: ManualRow): void {
  const csvContent = columnsToCsv(row.columns)
  const filename = `${row.source.replace(/_/g, '-')}-template.csv`
  downloadCsv(csvContent, filename)
}
</script>

<style scoped lang="scss">
.aim-data-schema-tab {
  max-width: 820px;
  margin: 0 auto;
}

.aim-schema-card {
  background: rgb(var(--v-theme-surface));
  border: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
  border-radius: 8px;
  padding: 18px 20px;
  box-shadow: 0 1px 2px rgba(16, 24, 40, 0.04);

  &__header {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 12px;

    &--space-between {
      justify-content: space-between;
    }
  }

  &__auto-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 0;
    border-bottom: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
    font-size: 13.5px;

    &:last-child {
      border-bottom: none;
    }
  }

  &__api-note {
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 12px;
    color: rgb(var(--v-theme-success));
    font-weight: 500;
  }

  &__upload-badge {
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 12px;
    color: rgb(var(--v-theme-warning));
    font-weight: 500;
  }
}

.aim-schema-table {
  :deep(th) {
    font-family: 'JetBrains Mono', monospace;
    font-size: 11px;
    font-weight: 600;
    background: rgba(var(--v-theme-surface-variant), 0.4);
    color: rgb(var(--v-theme-on-surface-variant));
  }

  :deep(td) {
    font-family: 'JetBrains Mono', monospace;
    font-size: 11.5px;
    color: rgb(var(--v-theme-on-surface-variant));
  }
}
</style>
```

- [ ] **Step 2: Run the component tests — expect all pass**

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.test.ts 2>&1 | tail -30
```

Expected: all tests listed in Task 3 pass.

- [ ] **Step 3: Run the full suite to confirm no regressions**

```bash
npm run test:ci 2>&1 | tail -20
```

Expected: passing count increases; no regressions.

---

### Task 5 — Type-check and lint

**Files:** all 4 new files.

- [ ] **Step 1: Type-check**

```bash
cd /path/to/frontend-mos
npx tsc --noEmit 2>&1 | grep -i "DataSchemaTab\|csvDownload\|error" | head -20
```

Expected: no errors referencing the new files.

- [ ] **Step 2: Lint**

```bash
npm run lint -- \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.test.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/csvDownload.test.ts \
  2>&1 | tail -20
```

Expected: no errors, no warnings.

---

### Task 6 — Commit

- [ ] **Step 1: Stage and commit all new files**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/DataSchemaTab.test.ts
git commit -m "feat(onboarding): add DataSchemaTab component + Vitest (O6)"
```

---

## Self-review

**Spec coverage check:**

| Requirement (product-spec-v2 §4.2 Tab 3) | Task |
|---|---|
| "Kochava collects automatically" auto-collected card | Task 4 `.aim-schema-card` auto block |
| "You need to provide" you-provide cards per manual source | Task 4 `manual-card` block |
| Green confirmation banner when all sources are direct API | Task 4 `v-alert[data-testid=all-auto-alert]` |
| Full schema column table per client-supplied item | Task 4 `v-table` with `th` per column |
| "File upload required" indicator per manual card | Task 4 `upload-badge` span |
| CSV template download per schema type | Task 4 `csv-download-btn` → `handleDownload` → `downloadCsv` |
| CSV download = Blob pattern (mirroring SmartLinksTrackerWidget) | `csvDownload.ts` — `new Blob(…)` + `URL.createObjectURL` + anchor click |
| Computed not stored (design-consolidated §8a) | Component receives `ProvisionPlan` prop; no store write, no persistence |
| `buildProvisionPlan` output consumed (F6) | `ProvisionPlan` prop type imported from F6 |
| OPEN: exact CSV columns pending Gary | Note in template comment; `COLUMN_REGISTRY` update is F6-only |

**Placeholder scan:** no TBD, no TODO, no "similar to," no steps without code. Sample table row uses `—` with a code comment flagging the Gary dependency — intentional, not a placeholder.

**Type consistency:** `ProvisionPlan`, `AutoRow`, `ManualRow` are imported from F6, not re-declared. `handleDownload(row: ManualRow)` matches the `v-for="row in plan.manual"` item type. `columnsToCsv` and `downloadCsv` exported from `csvDownload.ts` match their import in the `.vue` file.
