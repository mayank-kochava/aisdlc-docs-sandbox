---
id: plan-f6
title: "F6 — buildProvisionPlan Logic"
---

## Goal

Implement the framework-agnostic pure function `buildProvisionPlan(state)` in `frontend-mos` that
derives the Data Schema provision plan from wizard state. The function splits sources into two
buckets — **auto-collected** (Kochava direct API) and **client-must-provide** (file upload + CSV
template). Conditional rows are driven by wizard answers: offline media, LTV cohorts, web
attribution sources, iOS SKAN, and MMP-specific column names.

The output is **computed, never stored** (see `design-consolidated.md §8a` — Data Schema is in the
"Computed (never stored)" list). No I/O, no side effects. Input is a typed slice of wizard state;
output is a plain `ProvisionPlan` object.

CSV column sets per source are intentionally data-driven via a `COLUMN_REGISTRY` constant so that
Gary's pending column list (see Open Items below) can be filled without restructuring the function.
Tests assert branching behaviour and structure, not specific column-name strings, until the
registry is finalised.

---

## Files

**Create**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.test.ts`

**No other files modified by this plan.**

---

## Dependencies

- **F2** — `interfaces/aimOnboarding.ts` must exist and export `WizardState` (or a compatible
  named type) with at least the following fields used by this function:
  `platforms`, `mmp`, `mmpCollection`, `webAttrSources`, `spendCollection`, `usesOffline`,
  `wantsLtv`, `ltvCohortAvail`.

  Until F2 lands, stub the input type inline (see Task 1). Replace with the F2 import in Task 3.

---

## Open Items

> **OPEN — Gary / Data Engineering:** Exact CSV column sets per source (MMP file upload, web
> attribution per source, ad spend file, offline media, LTV cohort) are **not yet confirmed**.
> `COLUMN_REGISTRY` in the implementation uses placeholder columns that mirror the prototype
> (`mockup-k4a.html buildSchema()`). Gary or Anupam must supply the final column lists before O6
> (Data Schema tab) ships the CSV download. Update `COLUMN_REGISTRY` only — no structural change
> needed.
>
> Tests are written against branching behaviour (which rows appear under which conditions) and the
> shape of the registry, not against specific column-name strings, so they remain valid through the
> column update.

---

## Tasks

### Task 1 — Write the failing tests

**1.1** Create the test file:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.test.ts
```

with these contents:

```ts
import { describe, it, expect } from 'vitest'
import { buildProvisionPlan } from './buildProvisionPlan'
import type { ProvisionPlanInput } from './buildProvisionPlan'

// Minimal state factory — only fields this function reads
function state(overrides: Partial<ProvisionPlanInput> = {}): ProvisionPlanInput {
  return {
    platforms: [],
    mmp: null,
    mmpCollection: null,
    webAttrSources: [],
    spendCollection: null,
    usesOffline: false,
    wantsLtv: false,
    ltvCohortAvail: null,
    ...overrides,
  }
}

describe('buildProvisionPlan', () => {
  // ── MMP ──────────────────────────────────────────────────────────────────

  it('adds MMP to auto when mmpCollection is direct_api', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['iOS', 'Android'], mmp: 'appsflyer', mmpCollection: 'direct_api' }),
    )
    expect(plan.auto.some(r => r.source === 'mmp')).toBe(true)
    expect(plan.manual.some(r => r.source === 'mmp')).toBe(false)
  })

  it('adds MMP to manual when mmpCollection is file', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['iOS'], mmp: 'adjust', mmpCollection: 'file' }),
    )
    const row = plan.manual.find(r => r.source === 'mmp')
    expect(row).toBeDefined()
    expect(row!.columns).toBeInstanceOf(Array)
    expect(row!.columns.length).toBeGreaterThan(0)
  })

  it('omits MMP rows when no mobile platforms', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['Web'], mmp: 'appsflyer', mmpCollection: 'direct_api' }),
    )
    expect(plan.auto.some(r => r.source === 'mmp')).toBe(false)
    expect(plan.manual.some(r => r.source === 'mmp')).toBe(false)
  })

  it('omits MMP rows when mmp is other_none', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['iOS'], mmp: 'other_none', mmpCollection: null }),
    )
    expect(plan.auto.some(r => r.source === 'mmp')).toBe(false)
    expect(plan.manual.some(r => r.source === 'mmp')).toBe(false)
  })

  // ── Web attribution ───────────────────────────────────────────────────────

  it('adds GA4 to auto', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['Web'], webAttrSources: ['ga4'] }),
    )
    expect(plan.auto.some(r => r.source === 'web_attr_ga4')).toBe(true)
    expect(plan.manual.some(r => r.source === 'web_attr_ga4')).toBe(false)
  })

  it('adds non-GA4 web source to manual', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['Web'], webAttrSources: ['adobe_analytics'] }),
    )
    const row = plan.manual.find(r => r.source === 'web_attr_adobe_analytics')
    expect(row).toBeDefined()
    expect(row!.columns).toBeInstanceOf(Array)
  })

  it('handles multiple web sources — GA4 auto, other manual', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['Web'], webAttrSources: ['ga4', 'adobe_analytics'] }),
    )
    expect(plan.auto.some(r => r.source === 'web_attr_ga4')).toBe(true)
    expect(plan.manual.some(r => r.source === 'web_attr_adobe_analytics')).toBe(true)
  })

  it('omits web attr rows when no Web platform', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['iOS'], webAttrSources: ['ga4'] }),
    )
    expect(plan.auto.some(r => r.source === 'web_attr_ga4')).toBe(false)
  })

  // ── Ad spend ─────────────────────────────────────────────────────────────

  it('adds ad spend to auto when spendCollection is direct_api', () => {
    const plan = buildProvisionPlan(state({ spendCollection: 'direct_api' }))
    expect(plan.auto.some(r => r.source === 'ad_spend')).toBe(true)
    expect(plan.manual.some(r => r.source === 'ad_spend')).toBe(false)
  })

  it('adds ad spend to manual when spendCollection is file', () => {
    const plan = buildProvisionPlan(state({ spendCollection: 'file' }))
    const row = plan.manual.find(r => r.source === 'ad_spend')
    expect(row).toBeDefined()
    expect(row!.columns).toBeInstanceOf(Array)
  })

  // ── Offline media (conditional) ───────────────────────────────────────────

  it('adds offline media row to manual when usesOffline is true', () => {
    const plan = buildProvisionPlan(state({ usesOffline: true }))
    const row = plan.manual.find(r => r.source === 'offline_media')
    expect(row).toBeDefined()
    expect(row!.columns).toBeInstanceOf(Array)
  })

  it('omits offline media row when usesOffline is false', () => {
    const plan = buildProvisionPlan(state({ usesOffline: false }))
    expect(plan.manual.some(r => r.source === 'offline_media')).toBe(false)
  })

  // ── LTV cohorts (conditional) ─────────────────────────────────────────────

  it('adds LTV cohort row to manual when wantsLtv and ltvCohortAvail is yes', () => {
    const plan = buildProvisionPlan(state({ wantsLtv: true, ltvCohortAvail: 'yes' }))
    const row = plan.manual.find(r => r.source === 'ltv_cohort')
    expect(row).toBeDefined()
    expect(row!.columns).toBeInstanceOf(Array)
  })

  it('adds LTV cohort row to manual when wantsLtv and ltvCohortAvail is partial', () => {
    const plan = buildProvisionPlan(state({ wantsLtv: true, ltvCohortAvail: 'partial' }))
    expect(plan.manual.some(r => r.source === 'ltv_cohort')).toBe(true)
  })

  it('omits LTV cohort row when wantsLtv is false', () => {
    const plan = buildProvisionPlan(state({ wantsLtv: false, ltvCohortAvail: 'yes' }))
    expect(plan.manual.some(r => r.source === 'ltv_cohort')).toBe(false)
  })

  it('omits LTV cohort row when ltvCohortAvail is no', () => {
    const plan = buildProvisionPlan(state({ wantsLtv: true, ltvCohortAvail: 'no' }))
    expect(plan.manual.some(r => r.source === 'ltv_cohort')).toBe(false)
  })

  // ── allAuto flag ─────────────────────────────────────────────────────────

  it('sets allAuto true when manual bucket is empty', () => {
    const plan = buildProvisionPlan(
      state({
        platforms: ['iOS'],
        mmp: 'appsflyer',
        mmpCollection: 'direct_api',
        spendCollection: 'direct_api',
      }),
    )
    expect(plan.manual).toHaveLength(0)
    expect(plan.allAuto).toBe(true)
  })

  it('sets allAuto false when any manual row exists', () => {
    const plan = buildProvisionPlan(
      state({
        platforms: ['iOS'],
        mmp: 'adjust',
        mmpCollection: 'file',
        spendCollection: 'direct_api',
      }),
    )
    expect(plan.allAuto).toBe(false)
  })

  // ── AutoRow label includes MMP name ──────────────────────────────────────

  it('auto row label includes the MMP display name', () => {
    const plan = buildProvisionPlan(
      state({ platforms: ['Android'], mmp: 'singular', mmpCollection: 'direct_api' }),
    )
    const row = plan.auto.find(r => r.source === 'mmp')
    expect(row!.label).toMatch(/singular/i)
  })
})
```

**1.2** Run the tests (expect module-not-found failure — the implementation file does not exist yet):

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.test.ts 2>&1 | tail -30
```

---

### Task 2 — Stub the implementation (tests still fail, but compile)

Create the implementation file with correct exports so the test file can import:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.ts
```

with this stub:

```ts
export interface ProvisionPlanInput {
  platforms: string[]
  mmp: string | null
  mmpCollection: string | null
  webAttrSources: string[]
  spendCollection: string | null
  usesOffline: boolean
  wantsLtv: boolean
  ltvCohortAvail: string | null
}

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

export function buildProvisionPlan(_state: ProvisionPlanInput): ProvisionPlan {
  return { auto: [], manual: [], allAuto: true }
}
```

Run the tests (expect failures — assertions are unmet, but no compile errors):

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.test.ts 2>&1 | tail -30
```

---

### Task 3 — Implement `buildProvisionPlan`

Replace the stub with the real implementation:

```ts
// NOTE: Once F2 (interfaces/aimOnboarding.ts) lands, replace this inline type with:
//   import type { WizardState } from '../interfaces/aimOnboarding'
//   export type ProvisionPlanInput = Pick<WizardState,
//     'platforms' | 'mmp' | 'mmpCollection' | 'webAttrSources' |
//     'spendCollection' | 'usesOffline' | 'wantsLtv' | 'ltvCohortAvail'>

export interface ProvisionPlanInput {
  platforms: string[]
  mmp: string | null
  mmpCollection: string | null
  webAttrSources: string[]
  spendCollection: string | null
  usesOffline: boolean
  wantsLtv: boolean
  ltvCohortAvail: string | null
}

export interface AutoRow {
  source: string
  label: string
  note: string
}

export interface ManualRow {
  source: string
  label: string
  /** Placeholder columns — update COLUMN_REGISTRY when Gary confirms final column lists. */
  columns: string[]
}

export interface ProvisionPlan {
  auto: AutoRow[]
  manual: ManualRow[]
  /** True when every configured source is direct-API (green confirmation banner in Data Schema tab). */
  allAuto: boolean
}

// ── Display labels ────────────────────────────────────────────────────────────

const MMP_LABEL: Record<string, string> = {
  appsflyer: 'AppsFlyer',
  adjust: 'Adjust',
  singular: 'Singular',
  branch: 'Branch',
  kochava: 'Kochava',
}

const WEB_ATTR_LABEL: Record<string, string> = {
  ga4: 'Google Analytics 4',
  adobe_analytics: 'Adobe Analytics',
  amplitude: 'Amplitude',
  mixpanel: 'Mixpanel',
}

// ── Column registry ───────────────────────────────────────────────────────────
// OPEN ITEM: Gary / Data Engineering to confirm exact column lists per source.
// Replace placeholder arrays below; function structure is unchanged.

const COLUMN_REGISTRY: Record<string, string[]> = {
  mmp_file: ['date', 'campaign_id', 'campaign_name', 'installs', 'events', 'revenue'],
  web_attr_file: ['date', 'source', 'medium', 'campaign', 'sessions', 'conversions'],
  ad_spend_file: ['date', 'channel', 'campaign_id', 'spend', 'impressions', 'clicks'],
  offline_media: ['date', 'channel', 'market', 'spend', 'impressions'],
  ltv_cohort: ['install_date', 'day_7_rev', 'day_30_rev', 'day_90_rev'],
}

// ── Core function ─────────────────────────────────────────────────────────────

export function buildProvisionPlan(state: ProvisionPlanInput): ProvisionPlan {
  const auto: AutoRow[] = []
  const manual: ManualRow[] = []

  const hasMobile = state.platforms.some(p => p === 'iOS' || p === 'Android')
  const hasWeb = state.platforms.includes('Web')

  // ── MMP ──
  if (hasMobile && state.mmp && state.mmp !== 'other_none') {
    const mmpLabel = MMP_LABEL[state.mmp] ?? state.mmp
    if (state.mmpCollection === 'direct_api') {
      auto.push({ source: 'mmp', label: `${mmpLabel} (MMP)`, note: 'Direct API' })
    } else {
      manual.push({
        source: 'mmp',
        label: `${mmpLabel} mobile events`,
        columns: COLUMN_REGISTRY.mmp_file,
      })
    }
  }

  // ── Web attribution ──
  if (hasWeb) {
    for (const src of state.webAttrSources) {
      const sourceKey = `web_attr_${src}`
      const label = WEB_ATTR_LABEL[src] ?? `Web attribution (${src})`
      if (src === 'ga4') {
        auto.push({ source: sourceKey, label, note: 'Direct API' })
      } else {
        manual.push({
          source: sourceKey,
          label,
          columns: COLUMN_REGISTRY.web_attr_file,
        })
      }
    }
  }

  // ── Ad spend ──
  if (state.spendCollection === 'direct_api') {
    auto.push({ source: 'ad_spend', label: 'Ad spend', note: 'Direct API' })
  } else if (state.spendCollection) {
    manual.push({
      source: 'ad_spend',
      label: 'Ad spend data',
      columns: COLUMN_REGISTRY.ad_spend_file,
    })
  }

  // ── Offline media (conditional) ──
  if (state.usesOffline) {
    manual.push({
      source: 'offline_media',
      label: 'Offline media spend',
      columns: COLUMN_REGISTRY.offline_media,
    })
  }

  // ── LTV cohorts (conditional) ──
  if (state.wantsLtv && state.ltvCohortAvail && state.ltvCohortAvail !== 'no') {
    manual.push({
      source: 'ltv_cohort',
      label: 'LTV cohort revenue',
      columns: COLUMN_REGISTRY.ltv_cohort,
    })
  }

  return { auto, manual, allAuto: manual.length === 0 }
}
```

---

### Task 4 — Run tests (expect all pass)

```bash
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.test.ts 2>&1 | tail -30
```

All tests listed in Task 1 must pass. Then run the full suite to confirm no regressions:

```bash
npm run test:ci 2>&1 | tail -20
```

---

### Task 5 — Type-check and lint

```bash
npx tsc --noEmit 2>&1 | grep -i "buildProvisionPlan\|error" | head -20
npm run lint -- packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.ts 2>&1 | tail -20
```

Both must exit clean.

---

### Task 6 — Commit

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/buildProvisionPlan.test.ts

git commit -m "feat(onboarding): add buildProvisionPlan logic + Vitest (F6)"
```
