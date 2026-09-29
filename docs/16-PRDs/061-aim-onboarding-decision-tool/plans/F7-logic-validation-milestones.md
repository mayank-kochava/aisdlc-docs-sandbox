---
id: plan-f7
title: "F7 — Validation + Milestones Logic"
---

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement two pure TypeScript modules — `logic/validation.ts` (per-step required-field validation and output-screen completeness summary) and `logic/milestones.ts` (6-milestone template + est-date computation with week-Monday rounding + cascade recompute when a milestone is delayed) — both with full Vitest coverage.

**Architecture:** Both modules are pure functions with no framework dependency. They consume the `WizardState` and `Milestone[]` types defined by F2 (`interfaces/aimOnboarding.ts`). Validation derives per-step `done | error | dimmed` state and a list of offending fields for the output-screen approve gate. Milestones computes `est.start` / `est.end` from an injected anchor date (never `new Date()` inside) plus any accumulated delay deltas, always rounding to the next Monday on or after the computed date. Tests live co-located as `*.test.ts` files per the Vitest include glob `src/**/*.{test,spec}.{js,ts}`.

**Tech Stack:** TypeScript, Vitest (jsdom environment, `globals: true`), no Vue/Vuetify — these are pure-logic modules.

---

## Files

**Create**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.test.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.test.ts`

**Read (F2 contract — does not exist yet; create stub types if F2 is not merged)**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts`

---

## Dependencies

**F2** — `interfaces/aimOnboarding.ts` must exist and export `WizardState`, `Milestone`, and `TimelineState`.
If F2 has not landed yet, stub the types inline at the top of each `*.test.ts` file using the exact §5 contract from `design-consolidated.md`. Remove the stubs once F2 merges.

---

## F2 Type Contract (reference — do not reinvent)

The tests import these types. If F2 is merged, use the import; otherwise use this inline stub:

```ts
// Stub — remove once F2 is merged
export interface WizardState {
  companyName: string
  projectLeads: Array<{ name: string; email: string }>
  dataLeads: Array<{ name: string; email: string }>
  appName: string
  platforms: Array<'iOS' | 'Android' | 'Web'>
  platformShares: Partial<Record<'iOS' | 'Android' | 'Web', number>>
  modelling: string
  region: string
  regionOther: string
  budgetMonthly: number
  budgetAnnual: number
  budgetPeriod: 'monthly' | 'annual'
  usesOffline: boolean | null
  offlineSplitPct: number
  digitalMediaTypes: string[]
  hasAttrGaps: 'yes' | 'no' | 'not_sure' | ''
  attrGapCategories: string[]
  coverageConfidence: 'yes' | 'no' | 'not_sure' | ''
  paidSplit: Partial<Record<'iOS' | 'Android' | 'Web', number>>
  ua: boolean; uaShare: string
  ue: boolean; ueShare: string
  brand: boolean; brandShare: string
  campaignGrouping: boolean | null
  campaignGroupingChoice: string
  business: 'subscription' | 'ecommerce' | 'gaming' | ''
  funnel: { items: string[]; kpi: string; kpiConfirmed: boolean; names: Record<string, string> }
  webFunnel: { items: string[]; kpi: string }
  wantsLtv: boolean | null
  ltvCohortAvail: string
  ltvPartialChoice: string
  mmp: string
  mmpCollection: string
  mmpFileStorage: string
  appsflyerCohortAccess: boolean | null
  webAttrSources: string[]
  spendCollection: string
  adSpendSources: string[]
  history: Partial<Record<'iOS' | 'Android' | 'Web', string>>
  externalFactors: string[]
  externalFactorsNotes: string
  objectivesGoal: string
  objectivesSuccess: string
  objectivesMarketing: string
  uaUeRoutingFlag: string
  brandRoutingFlag: boolean
}

export interface Milestone {
  key: string
  name: string
  responsible: string
  weekOffset: number
  durationLabel: string   // display string: "90 min" | "1 hour" | "2-3 weeks" etc.
  durationWeeks: number   // upper bound in weeks (0 for sub-day events)
  status: string
  delayedToDate: string | null  // ISO date string or null
  notes: string
}

export interface TimelineState {
  anchorDate: string | null  // ISO date string; null until SoW approved
  milestones: Milestone[]
}

export interface EstDates {
  start: Date
  end: Date
}
```

---

## Convention: weekOffset indexing

"Week N" in the milestone table means the Nth week from kickoff. The formula uses zero-based `weekOffset`: `weekOffset = weekNumber - 1`.

| Milestone | AIM X weekNumber | AIM X weekOffset | AIM Pro weekNumber | AIM Pro weekOffset |
|-----------|-----------------|------------------|--------------------|-------------------|
| 1 Kickoff | 1 | 0 | 1 | 0 |
| 2 SoW approval | 1 | 0 | 1 | 0 |
| 3 Data connection | 2 | 1 | 2 | 1 |
| 4 Data QA | 4 | 3 | 6 | 5 |
| 5 Model training | 5 | 4 | 7 | 6 |
| 6 First insights | 7 | 6 | 9 | 8 |

Both milestones 1 and 2 have `weekOffset = 0` — they land in the same anchor week.

---

## Tasks

### Task 1 — Write failing milestone tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.test.ts`

- [ ] **Step 1.1 — Create the test file**

```ts
// milestones.test.ts
import { describe, it, expect } from 'vitest'
import {
  nextMonday,
  computeEstDates,
  buildMilestoneTemplate,
  MILESTONES_AIM_X,
  MILESTONES_AIM_PRO,
} from './milestones'
import type { Milestone } from '../interfaces/aimOnboarding'

// ── nextMonday ─────────────────────────────────────────────────────────────

describe('nextMonday', () => {
  it('returns the date unchanged when already Monday', () => {
    const monday = new Date('2026-06-08') // known Monday
    expect(nextMonday(monday)).toEqual(new Date('2026-06-08'))
  })

  it('advances Tuesday by 6 days to next Monday', () => {
    expect(nextMonday(new Date('2026-06-09'))).toEqual(new Date('2026-06-15'))
  })

  it('advances Wednesday by 5 days', () => {
    expect(nextMonday(new Date('2026-06-10'))).toEqual(new Date('2026-06-15'))
  })

  it('advances Thursday by 4 days', () => {
    expect(nextMonday(new Date('2026-06-11'))).toEqual(new Date('2026-06-15'))
  })

  it('advances Friday by 3 days', () => {
    expect(nextMonday(new Date('2026-06-12'))).toEqual(new Date('2026-06-15'))
  })

  it('advances Saturday by 2 days', () => {
    expect(nextMonday(new Date('2026-06-13'))).toEqual(new Date('2026-06-15'))
  })

  it('advances Sunday by 1 day', () => {
    expect(nextMonday(new Date('2026-06-14'))).toEqual(new Date('2026-06-15'))
  })
})

// ── computeEstDates — no delays ───────────────────────────────────────────

describe('computeEstDates — no delays', () => {
  // anchor = Monday 2026-06-01
  const anchor = new Date('2026-06-01')

  const milestones: Milestone[] = [
    { key: 'm1', name: '', responsible: '', weekOffset: 0, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    { key: 'm2', name: '', responsible: '', weekOffset: 2, durationLabel: '', durationWeeks: 1, status: '', delayedToDate: null, notes: '' },
    { key: 'm3', name: '', responsible: '', weekOffset: 4, durationLabel: '', durationWeeks: 2, status: '', delayedToDate: null, notes: '' },
    { key: 'm4', name: '', responsible: '', weekOffset: 8, durationLabel: '', durationWeeks: 3, status: '', delayedToDate: null, notes: '' },
  ]

  it('m1: anchor + 0 weeks = 2026-06-01 (Monday stays)', () => {
    const { start } = computeEstDates(milestones, anchor)['m1']
    expect(start).toEqual(new Date('2026-06-01'))
  })

  it('m2: anchor + 2 weeks = 2026-06-15', () => {
    const { start } = computeEstDates(milestones, anchor)['m2']
    expect(start).toEqual(new Date('2026-06-15'))
  })

  it('m3: anchor + 4 weeks = 2026-06-29', () => {
    const { start } = computeEstDates(milestones, anchor)['m3']
    expect(start).toEqual(new Date('2026-06-29'))
  })

  it('est.end = est.start + durationWeeks weeks', () => {
    const { start, end } = computeEstDates(milestones, anchor)['m2']
    const expectedEnd = new Date(start)
    expectedEnd.setDate(expectedEnd.getDate() + 1 * 7)
    expect(end).toEqual(expectedEnd)
  })

  it('sub-day milestone (durationWeeks=0): est.end equals est.start', () => {
    const { start, end } = computeEstDates(milestones, anchor)['m1']
    expect(end).toEqual(start)
  })

  it('all starts are Mondays', () => {
    const results = computeEstDates(milestones, anchor)
    for (const key of Object.keys(results)) {
      expect(results[key].start.getDay()).toBe(1) // 1 = Monday
    }
  })
})

// ── computeEstDates — anchor not Monday ───────────────────────────────────

describe('computeEstDates — anchor on non-Monday', () => {
  it('Wednesday anchor + weekOffset 0 → rounds up to next Monday', () => {
    const anchor = new Date('2026-06-10') // Wednesday
    const ms: Milestone[] = [
      { key: 'm1', name: '', responsible: '', weekOffset: 0, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    ]
    const { start } = computeEstDates(ms, anchor)['m1']
    expect(start).toEqual(new Date('2026-06-15'))
    expect(start.getDay()).toBe(1)
  })

  it('Sunday anchor + weekOffset 1 → rounds up (Sunday + 7 days = Sunday; then +1 to Monday)', () => {
    const anchor = new Date('2026-06-07') // Sunday
    const ms: Milestone[] = [
      { key: 'm1', name: '', responsible: '', weekOffset: 1, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    ]
    // Sunday + 7 days = 2026-06-14 (Sunday); nextMonday = 2026-06-15
    const { start } = computeEstDates(ms, anchor)['m1']
    expect(start).toEqual(new Date('2026-06-15'))
  })
})

// ── computeEstDates — cascade single delay ────────────────────────────────

describe('computeEstDates — cascade: single delay', () => {
  const anchor = new Date('2026-06-01') // Monday

  it('delay on m1 shifts m2, m3 by the same delta', () => {
    // m1 natural = 2026-06-01; delayed to 2026-06-08 (+7 days = +1 week)
    // m2 natural = 2026-06-15; shifted → 2026-06-22
    // m3 natural = 2026-06-29; shifted → 2026-07-06
    const ms: Milestone[] = [
      { key: 'm1', name: '', responsible: '', weekOffset: 0, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: '2026-06-08', notes: '' },
      { key: 'm2', name: '', responsible: '', weekOffset: 2, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
      { key: 'm3', name: '', responsible: '', weekOffset: 4, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    ]
    const result = computeEstDates(ms, anchor)
    expect(result['m1'].start).toEqual(new Date('2026-06-08'))
    expect(result['m2'].start).toEqual(new Date('2026-06-22'))
    expect(result['m3'].start).toEqual(new Date('2026-07-06'))
  })

  it('milestone before the delay is NOT shifted', () => {
    // m1 has no delay; m2 is delayed
    const ms: Milestone[] = [
      { key: 'm1', name: '', responsible: '', weekOffset: 0, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
      { key: 'm2', name: '', responsible: '', weekOffset: 2, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: '2026-06-22', notes: '' },
      { key: 'm3', name: '', responsible: '', weekOffset: 4, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    ]
    const result = computeEstDates(ms, anchor)
    // m1 unchanged
    expect(result['m1'].start).toEqual(new Date('2026-06-01'))
    // m2 = delayed date
    expect(result['m2'].start).toEqual(new Date('2026-06-22'))
    // m3 natural 2026-06-29; shifted by +7 days → 2026-07-06
    expect(result['m3'].start).toEqual(new Date('2026-07-06'))
  })

  it('delayed date on a non-Monday is rounded to next Monday before computing delta', () => {
    // m1 natural 2026-06-01; delayed to Wednesday 2026-06-10 → rounded to 2026-06-15
    const ms: Milestone[] = [
      { key: 'm1', name: '', responsible: '', weekOffset: 0, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: '2026-06-10', notes: '' },
      { key: 'm2', name: '', responsible: '', weekOffset: 2, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    ]
    const result = computeEstDates(ms, anchor)
    // m1 rounded → 2026-06-15 (delta = +14 days)
    expect(result['m1'].start).toEqual(new Date('2026-06-15'))
    expect(result['m1'].start.getDay()).toBe(1)
    // m2 natural 2026-06-15; shifted by +14 → 2026-06-29
    expect(result['m2'].start).toEqual(new Date('2026-06-29'))
  })
})

// ── computeEstDates — cascade cumulative (two delays) ─────────────────────

describe('computeEstDates — cascade: cumulative two delays', () => {
  const anchor = new Date('2026-06-01') // Monday

  it('two independent delays accumulate correctly', () => {
    // m1 natural 2026-06-01; delayed to 2026-06-08 (+7 days)
    // m2 natural 2026-06-15; after m1 delta → 2026-06-22; delayed to 2026-06-29 (+7 more)
    // total delta for m3+ = +14 days
    // m3 natural 2026-06-29; shifted by +14 → 2026-07-13
    const ms: Milestone[] = [
      { key: 'm1', name: '', responsible: '', weekOffset: 0, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: '2026-06-08', notes: '' },
      { key: 'm2', name: '', responsible: '', weekOffset: 2, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: '2026-06-29', notes: '' },
      { key: 'm3', name: '', responsible: '', weekOffset: 4, durationLabel: '', durationWeeks: 0, status: '', delayedToDate: null, notes: '' },
    ]
    const result = computeEstDates(ms, anchor)
    expect(result['m1'].start).toEqual(new Date('2026-06-08'))
    expect(result['m2'].start).toEqual(new Date('2026-06-29'))
    expect(result['m3'].start).toEqual(new Date('2026-07-13'))
  })
})

// ── buildMilestoneTemplate ────────────────────────────────────────────────

describe('buildMilestoneTemplate', () => {
  it('AIM X: returns 6 milestones', () => {
    expect(buildMilestoneTemplate('aim_x')).toHaveLength(6)
  })

  it('AIM Pro: returns 6 milestones', () => {
    expect(buildMilestoneTemplate('aim_pro')).toHaveLength(6)
  })

  it('AIM X: first two milestones both have weekOffset 0', () => {
    const ms = buildMilestoneTemplate('aim_x')
    expect(ms[0].weekOffset).toBe(0)
    expect(ms[1].weekOffset).toBe(0)
  })

  it('AIM X: milestone 6 has weekOffset 6', () => {
    const ms = buildMilestoneTemplate('aim_x')
    expect(ms[5].weekOffset).toBe(6)
  })

  it('AIM Pro: milestone 4 (Data QA) has weekOffset 5', () => {
    const ms = buildMilestoneTemplate('aim_pro')
    expect(ms[3].weekOffset).toBe(5)
  })

  it('AIM Pro: milestone 6 has weekOffset 8', () => {
    const ms = buildMilestoneTemplate('aim_pro')
    expect(ms[5].weekOffset).toBe(8)
  })

  it('all milestones have a non-empty key and name', () => {
    for (const tier of ['aim_x', 'aim_pro'] as const) {
      buildMilestoneTemplate(tier).forEach((m) => {
        expect(m.key).toBeTruthy()
        expect(m.name).toBeTruthy()
      })
    }
  })

  it('durationWeeks is 0 for sub-day milestones (kickoff and SoW approval)', () => {
    const ms = buildMilestoneTemplate('aim_x')
    expect(ms[0].durationWeeks).toBe(0) // "90 min"
    expect(ms[1].durationWeeks).toBe(0) // "1 hour"
  })

  it('MILESTONES_AIM_X constant matches buildMilestoneTemplate("aim_x")', () => {
    expect(MILESTONES_AIM_X).toEqual(buildMilestoneTemplate('aim_x'))
  })

  it('MILESTONES_AIM_PRO constant matches buildMilestoneTemplate("aim_pro")', () => {
    expect(MILESTONES_AIM_PRO).toEqual(buildMilestoneTemplate('aim_pro'))
  })
})
```

- [ ] **Step 1.2 — Run to confirm FAIL (module not found)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.test.ts 2>&1 | tail -20
```

Expected output contains: `Cannot find module './milestones'`

---

### Task 2 — Implement `milestones.ts`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.ts`

- [ ] **Step 2.1 — Create the file**

```ts
// milestones.ts
// Pure functions — no Vue, no Vuetify, no side effects.
// design-consolidated §8a: est.start = nextMonday(anchorDate + weekOffset × 7 days + cumulativeDeltaDays)
//                          est.end   = est.start + durationWeeks × 7 days
// Inject `anchorDate` — never call new Date() here.

import type { Milestone, EstDates } from '../interfaces/aimOnboarding'

// ── nextMonday ──────────────────────────────────────────────────────────────
// Returns the date itself if already Monday; otherwise advances to the
// next Monday (on or after — design-consolidated §4.2 "next Monday on or after").
// Returns a new Date object at midnight local time.

export function nextMonday(date: Date): Date {
  const d = new Date(date)
  d.setHours(0, 0, 0, 0)
  const day = d.getDay() // 0=Sun, 1=Mon … 6=Sat
  const daysUntilMonday = (8 - day) % 7  // 0 if already Monday
  d.setDate(d.getDate() + daysUntilMonday)
  return d
}

// ── computeEstDates ─────────────────────────────────────────────────────────
// Returns a map of milestone key → { start, end }.
// Process milestones in array order so delays accumulate correctly.
// A delayedToDate on milestone N adds (delayedMonday − naturalMonday) days
// to all subsequent natural dates.

export function computeEstDates(
  milestones: Milestone[],
  anchorDate: Date
): Record<string, EstDates> {
  const result: Record<string, EstDates> = {}
  let cumulativeDeltaDays = 0

  for (const m of milestones) {
    const naturalRaw = new Date(anchorDate)
    naturalRaw.setDate(naturalRaw.getDate() + m.weekOffset * 7 + cumulativeDeltaDays)
    const naturalMonday = nextMonday(naturalRaw)

    let start: Date

    if (m.delayedToDate !== null && m.delayedToDate !== undefined) {
      const delayedMonday = nextMonday(new Date(m.delayedToDate))
      const delta = (delayedMonday.getTime() - naturalMonday.getTime()) / (1000 * 60 * 60 * 24)
      cumulativeDeltaDays += delta
      start = delayedMonday
    } else {
      start = naturalMonday
    }

    const end = new Date(start)
    end.setDate(end.getDate() + m.durationWeeks * 7)

    result[m.key] = { start, end }
  }

  return result
}

// ── milestone template ──────────────────────────────────────────────────────
// 6 milestones — product-spec-v2 §4.2 Tab 2 / design-consolidated.
// weekOffset = weekNumber - 1 (zero-based).
// durationWeeks: upper bound in weeks; 0 for sub-day events.

interface MilestoneTemplate extends Omit<Milestone, 'status' | 'delayedToDate' | 'notes'> {
  status: string
  delayedToDate: null
  notes: string
}

function makeMilestone(
  key: string,
  name: string,
  responsible: string,
  weekOffset: number,
  durationLabel: string,
  durationWeeks: number
): MilestoneTemplate {
  return { key, name, responsible, weekOffset, durationLabel, durationWeeks, status: 'not_started', delayedToDate: null, notes: '' }
}

export function buildMilestoneTemplate(tier: 'aim_x' | 'aim_pro'): MilestoneTemplate[] {
  if (tier === 'aim_x') {
    return [
      makeMilestone('kickoff',      'Onboarding kickoff call',      'Client + Kochava', 0, '90 min',    0),
      makeMilestone('sow_approval', 'Scope of Work approval',       'Client + Kochava', 0, '1 hour',    0),
      makeMilestone('data_connect', 'Data connection / file upload', 'Client',           1, '2-3 weeks', 3),
      makeMilestone('data_qa',      'Data QA and validation',        'Kochava',          3, '~1 week',   1),
      makeMilestone('model_train',  'Model training',                'Kochava',          4, '1-2 weeks', 2),
      makeMilestone('first_insight','First insights delivered',      'Kochava',          6, '1 hour',    0),
    ]
  }
  // aim_pro — longer durations on milestones 3-5
  return [
    makeMilestone('kickoff',      'Onboarding kickoff call',      'Client + Kochava', 0, '90 min',    0),
    makeMilestone('sow_approval', 'Scope of Work approval',       'Client + Kochava', 0, '1 hour',    0),
    makeMilestone('data_connect', 'Data connection / file upload', 'Client',           1, '3-5 weeks', 5),
    makeMilestone('data_qa',      'Data QA and validation',        'Kochava',          5, '1-2 weeks', 2),
    makeMilestone('model_train',  'Model training',                'Kochava',          6, '2-3 weeks', 3),
    makeMilestone('first_insight','First insights delivered',      'Kochava',          8, '1 hour',    0),
  ]
}

export const MILESTONES_AIM_X   = buildMilestoneTemplate('aim_x')
export const MILESTONES_AIM_PRO = buildMilestoneTemplate('aim_pro')
```

- [ ] **Step 2.2 — Run milestone tests (expect PASS)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.test.ts 2>&1 | tail -30
```

Expected: all tests pass, no failures.

- [ ] **Step 2.3 — Commit milestones**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.ts \
        packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/milestones.test.ts
git commit -m "feat(onboarding): add logic/milestones — est-date formula + cascade + template (F7)"
```

---

### Task 3 — Write failing validation tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.test.ts`

- [ ] **Step 3.1 — Create the test file**

```ts
// validation.test.ts
import { describe, it, expect } from 'vitest'
import {
  validateStep,
  buildValidationSummary,
} from './validation'
import type { WizardState } from '../interfaces/aimOnboarding'

// ── helpers ────────────────────────────────────────────────────────────────

function base(): WizardState {
  return {
    companyName: 'Acme',
    projectLeads: [{ name: 'Alice', email: 'alice@acme.com' }],
    dataLeads: [{ name: 'Bob', email: 'bob@acme.com' }],
    appName: 'AcmeApp',
    platforms: ['iOS'],
    platformShares: {},
    modelling: '',
    region: 'US',
    regionOther: '',
    budgetMonthly: 300000,
    budgetAnnual: 0,
    budgetPeriod: 'monthly',
    usesOffline: false,
    offlineSplitPct: 20,
    digitalMediaTypes: ['self_attributed'],
    hasAttrGaps: 'no',
    attrGapCategories: [],
    coverageConfidence: 'yes',
    paidSplit: { iOS: 65 },
    ua: true, uaShare: '60',
    ue: false, ueShare: '',
    brand: false, brandShare: '',
    campaignGrouping: false,
    campaignGroupingChoice: '',
    business: 'subscription',
    funnel: { items: ['install', 'trial_start', 'subscription_start'], kpi: 'trial_start', kpiConfirmed: true, names: {} },
    webFunnel: { items: [], kpi: '' },
    wantsLtv: false,
    ltvCohortAvail: '',
    ltvPartialChoice: '',
    mmp: 'appsflyer',
    mmpCollection: 'api',
    mmpFileStorage: '',
    appsflyerCohortAccess: true,
    webAttrSources: [],
    spendCollection: 'api',
    adSpendSources: ['meta'],
    history: { iOS: '24_36' },
    externalFactors: [],
    externalFactorsNotes: '',
    objectivesGoal: '',
    objectivesSuccess: '',
    objectivesMarketing: '',
    uaUeRoutingFlag: '',
    brandRoutingFlag: false,
  }
}

// ── Step 1 validation ─────────────────────────────────────────────────────

describe('validateStep — step 1 (Team)', () => {
  it('done when companyName, ≥1 projectLead, ≥1 dataLead', () => {
    expect(validateStep(1, base())).toBe('done')
  })

  it('error when companyName is empty', () => {
    const s = { ...base(), companyName: '' }
    expect(validateStep(1, s)).toBe('error')
  })

  it('error when projectLeads is empty', () => {
    const s = { ...base(), projectLeads: [] }
    expect(validateStep(1, s)).toBe('error')
  })

  it('error when projectLead has empty name', () => {
    const s = { ...base(), projectLeads: [{ name: '', email: 'a@b.com' }] }
    expect(validateStep(1, s)).toBe('error')
  })

  it('error when dataLeads is empty', () => {
    const s = { ...base(), dataLeads: [] }
    expect(validateStep(1, s)).toBe('error')
  })
})

// ── Step 2 validation ─────────────────────────────────────────────────────

describe('validateStep — step 2 (Product)', () => {
  it('done for single-platform state', () => {
    expect(validateStep(2, base())).toBe('done')
  })

  it('error when appName is empty', () => {
    const s = { ...base(), appName: '' }
    expect(validateStep(2, s)).toBe('error')
  })

  it('error when platforms is empty', () => {
    const s = { ...base(), platforms: [] }
    expect(validateStep(2, s)).toBe('error')
  })

  it('error when ≥2 platforms and platformShares is empty (required when ≥2 platforms)', () => {
    const s = { ...base(), platforms: ['iOS', 'Android'] as WizardState['platforms'], platformShares: {} }
    expect(validateStep(2, s)).toBe('error')
  })

  it('done when ≥2 platforms and platformShares has entries for each', () => {
    const s = { ...base(), platforms: ['iOS', 'Android'] as WizardState['platforms'], platformShares: { iOS: 60, Android: 40 } }
    expect(validateStep(2, s)).toBe('done')
  })

  it('error when region is empty', () => {
    const s = { ...base(), region: '' }
    expect(validateStep(2, s)).toBe('error')
  })

  it('dimmed (not required) when modelling is empty and platforms is single', () => {
    // modelling is only required when Web + ≥1 mobile — single platform doesn't require it
    const s = { ...base(), modelling: '' }
    expect(validateStep(2, s)).toBe('done')
  })

  it('error when Web + iOS both selected and modelling is empty', () => {
    const s = {
      ...base(),
      platforms: ['iOS', 'Web'] as WizardState['platforms'],
      platformShares: { iOS: 60, Web: 40 },
      modelling: '',
    }
    expect(validateStep(2, s)).toBe('error')
  })
})

// ── Step 3 validation ─────────────────────────────────────────────────────

describe('validateStep — step 3 (Marketing)', () => {
  it('done for fully-filled state', () => {
    expect(validateStep(3, base())).toBe('done')
  })

  it('error when budgetMonthly is 0 and budgetPeriod is monthly', () => {
    const s = { ...base(), budgetMonthly: 0, budgetPeriod: 'monthly' as const }
    expect(validateStep(3, s)).toBe('error')
  })

  it('error when budgetAnnual is 0 and budgetPeriod is annual', () => {
    const s = { ...base(), budgetAnnual: 0, budgetPeriod: 'annual' as const }
    expect(validateStep(3, s)).toBe('error')
  })

  it('error when usesOffline is null (unanswered)', () => {
    const s = { ...base(), usesOffline: null }
    expect(validateStep(3, s)).toBe('error')
  })

  it('error when digitalMediaTypes is empty', () => {
    const s = { ...base(), digitalMediaTypes: [] }
    expect(validateStep(3, s)).toBe('error')
  })

  it('error when hasAttrGaps is empty string (unanswered)', () => {
    const s = { ...base(), hasAttrGaps: '' as WizardState['hasAttrGaps'] }
    expect(validateStep(3, s)).toBe('error')
  })

  it('error when coverageConfidence is empty string', () => {
    const s = { ...base(), coverageConfidence: '' as WizardState['coverageConfidence'] }
    expect(validateStep(3, s)).toBe('error')
  })

  it('error when no campaign type selected (ua, ue, brand all false)', () => {
    const s = { ...base(), ua: false, ue: false, brand: false }
    expect(validateStep(3, s)).toBe('error')
  })

  it('done when campaignGrouping is false (campaignGroupingChoice not required)', () => {
    const s = { ...base(), campaignGrouping: false, campaignGroupingChoice: '' }
    expect(validateStep(3, s)).toBe('done')
  })

  it('error when campaignGrouping is true and campaignGroupingChoice is empty', () => {
    const s = { ...base(), campaignGrouping: true, campaignGroupingChoice: '' }
    expect(validateStep(3, s)).toBe('error')
  })

  it('done when campaignGrouping is true and campaignGroupingChoice is filled', () => {
    const s = { ...base(), campaignGrouping: true, campaignGroupingChoice: 'Brand campaigns' }
    expect(validateStep(3, s)).toBe('done')
  })

  it('error when campaignGrouping is null (unanswered)', () => {
    const s = { ...base(), campaignGrouping: null }
    expect(validateStep(3, s)).toBe('error')
  })
})

// ── Step 4 validation ─────────────────────────────────────────────────────

describe('validateStep — step 4 (Business & Funnel)', () => {
  it('done for subscription state with funnel', () => {
    expect(validateStep(4, base())).toBe('done')
  })

  it('error when business is empty', () => {
    const s = { ...base(), business: '' as WizardState['business'] }
    expect(validateStep(4, s)).toBe('error')
  })

  it('error when funnel.items has fewer than 3 events', () => {
    const s = { ...base(), funnel: { ...base().funnel, items: ['install', 'trial_start'] } }
    expect(validateStep(4, s)).toBe('error')
  })

  it('error when funnel.kpi is empty', () => {
    const s = { ...base(), funnel: { ...base().funnel, kpi: '' } }
    expect(validateStep(4, s)).toBe('error')
  })

  it('webFunnel is dimmed (not validated) when Web not in platforms', () => {
    // base() has platforms: ['iOS'] — webFunnel items empty should not error
    const s = { ...base(), webFunnel: { items: [], kpi: '' } }
    expect(validateStep(4, s)).toBe('done')
  })

  it('error when Web is selected and webFunnel.items has fewer than 2', () => {
    const s = {
      ...base(),
      platforms: ['iOS', 'Web'] as WizardState['platforms'],
      platformShares: { iOS: 60, Web: 40 },
      modelling: 'separate',
      webFunnel: { items: ['landing_page'], kpi: '' },
    }
    expect(validateStep(4, s)).toBe('error')
  })
})

// ── Step 5 validation ─────────────────────────────────────────────────────

describe('validateStep — step 5 (Data Sources)', () => {
  it('done for mobile-only iOS with AppsFlyer', () => {
    expect(validateStep(5, base())).toBe('done')
  })

  it('error when mmp is empty and platforms contains mobile', () => {
    const s = { ...base(), mmp: '' }
    expect(validateStep(5, s)).toBe('error')
  })

  it('appsflyerCohortAccess is dimmed (not required) when mmp is not appsflyer', () => {
    const s = { ...base(), mmp: 'adjust', appsflyerCohortAccess: null }
    expect(validateStep(5, s)).toBe('done')
  })

  it('error when mmp is appsflyer and appsflyerCohortAccess is null', () => {
    const s = { ...base(), mmp: 'appsflyer', appsflyerCohortAccess: null }
    expect(validateStep(5, s)).toBe('error')
  })

  it('webAttrSources is dimmed when Web not in platforms', () => {
    const s = { ...base(), webAttrSources: [] }
    expect(validateStep(5, s)).toBe('done')
  })

  it('error when Web in platforms and webAttrSources is empty', () => {
    const s = {
      ...base(),
      platforms: ['iOS', 'Web'] as WizardState['platforms'],
      platformShares: { iOS: 60, Web: 40 },
      modelling: 'separate',
      webAttrSources: [],
    }
    expect(validateStep(5, s)).toBe('error')
  })

  it('error when adSpendSources is empty', () => {
    const s = { ...base(), adSpendSources: [] }
    expect(validateStep(5, s)).toBe('error')
  })

  it('error when history is missing for a selected platform', () => {
    const s = { ...base(), history: {} }
    expect(validateStep(5, s)).toBe('error')
  })
})

// ── Steps 6 and 7 are optional — never block ─────────────────────────────

describe('validateStep — steps 6 and 7 (optional)', () => {
  it('step 6 is always done regardless of content', () => {
    expect(validateStep(6, base())).toBe('done')
    expect(validateStep(6, { ...base(), externalFactors: [] })).toBe('done')
  })

  it('step 7 is always done regardless of content', () => {
    expect(validateStep(7, base())).toBe('done')
    expect(validateStep(7, { ...base(), objectivesGoal: '' })).toBe('done')
  })
})

// ── Step 8 (Review) — no new input, always done ───────────────────────────

describe('validateStep — step 8 (Review)', () => {
  it('step 8 is always done (no new required fields)', () => {
    expect(validateStep(8, base())).toBe('done')
  })
})

// ── buildValidationSummary ─────────────────────────────────────────────────

describe('buildValidationSummary', () => {
  it('empty array when all required steps are valid', () => {
    expect(buildValidationSummary(base())).toHaveLength(0)
  })

  it('lists step 1 when companyName is missing', () => {
    const s = { ...base(), companyName: '' }
    const summary = buildValidationSummary(s)
    expect(summary.some((e) => e.step === 1)).toBe(true)
  })

  it('includes firstInvalidField pointing to "companyName" for step 1 failure', () => {
    const s = { ...base(), companyName: '' }
    const summary = buildValidationSummary(s)
    const entry = summary.find((e) => e.step === 1)
    expect(entry?.firstInvalidField).toBe('companyName')
  })

  it('lists step 3 when campaign types all false', () => {
    const s = { ...base(), ua: false, ue: false, brand: false }
    const summary = buildValidationSummary(s)
    expect(summary.some((e) => e.step === 3)).toBe(true)
  })

  it('does NOT list step 6 even when external factors are empty', () => {
    const s = { ...base(), externalFactors: [] }
    expect(buildValidationSummary(s).some((e) => e.step === 6)).toBe(false)
  })

  it('isApproveBlocked is true when any required step is invalid', () => {
    const s = { ...base(), companyName: '' }
    const summary = buildValidationSummary(s)
    expect(summary.length).toBeGreaterThan(0)
  })

  it('multiple invalid steps all appear in the summary', () => {
    const s = { ...base(), companyName: '', platforms: [] }
    const summary = buildValidationSummary(s)
    const steps = summary.map((e) => e.step)
    expect(steps).toContain(1)
    expect(steps).toContain(2)
  })
})
```

- [ ] **Step 3.2 — Run to confirm FAIL (module not found)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.test.ts 2>&1 | tail -20
```

Expected output contains: `Cannot find module './validation'`

---

### Task 4 — Implement `validation.ts`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.ts`

- [ ] **Step 4.1 — Create the file**

```ts
// validation.ts
// Pure functions — no Vue, no Vuetify, no side effects.
// design-consolidated §8a:
//   - A step is "done" when every *active* required field is valid.
//   - "Active" = required-ness is not gated by an unanswered parent (the "dimmed" concept).
//   - Steps 6 (External Factors) and 7 (Objectives) are optional — never error.
//   - Output Approve is blocked until all required steps are valid.

import type { WizardState } from '../interfaces/aimOnboarding'

export type StepStatus = 'done' | 'error' | 'dimmed'

export interface ValidationSummaryEntry {
  step: number
  stepLabel: string
  firstInvalidField: string
  message: string
}

// ── Step labels ─────────────────────────────────────────────────────────────

const STEP_LABELS: Record<number, string> = {
  1: 'Your Team',
  2: 'About Your Product',
  3: 'Marketing Setup',
  4: 'Business Model & Funnel',
  5: 'Data Sources',
  6: 'External Factors',
  7: 'Objectives',
  8: 'Review',
}

// ── Step validators ─────────────────────────────────────────────────────────

function validateStep1(s: WizardState): string | null {
  if (!s.companyName?.trim()) return 'companyName'
  if (!s.projectLeads?.length || !s.projectLeads[0]?.name?.trim()) return 'projectLeads'
  if (!s.dataLeads?.length || !s.dataLeads[0]?.name?.trim()) return 'dataLeads'
  return null
}

function validateStep2(s: WizardState): string | null {
  if (!s.appName?.trim()) return 'appName'
  if (!s.platforms?.length) return 'platforms'
  if (s.platforms.length >= 2) {
    const covered = s.platforms.every((p) => (s.platformShares[p] ?? 0) > 0)
    if (!covered) return 'platformShares'
  }
  // modelling required when Web + ≥1 mobile selected
  const hasMobile = s.platforms.includes('iOS') || s.platforms.includes('Android')
  const hasWeb    = s.platforms.includes('Web')
  if (hasMobile && hasWeb && !s.modelling) return 'modelling'
  if (!s.region?.trim()) return 'region'
  return null
}

function validateStep3(s: WizardState): string | null {
  const budget = s.budgetPeriod === 'annual' ? s.budgetAnnual : s.budgetMonthly
  if (!budget) return 'budgetMonthly'
  if (s.usesOffline === null || s.usesOffline === undefined) return 'usesOffline'
  if (!s.digitalMediaTypes?.length) return 'digitalMediaTypes'
  if (!s.hasAttrGaps) return 'hasAttrGaps'
  if (!s.coverageConfidence) return 'coverageConfidence'
  if (!s.ua && !s.ue && !s.brand) return 'ua'
  // campaignGrouping is required (§3.9); must be explicitly answered
  if (s.campaignGrouping === null || s.campaignGrouping === undefined) return 'campaignGrouping'
  // campaignGroupingChoice required only when campaignGrouping = true
  if (s.campaignGrouping === true && !s.campaignGroupingChoice?.trim()) return 'campaignGroupingChoice'
  return null
}

function validateStep4(s: WizardState): string | null {
  if (!s.business) return 'business'
  if (!s.funnel?.items?.length || s.funnel.items.length < 3) return 'funnel.items'
  if (!s.funnel?.kpi) return 'funnel.kpi'
  // webFunnel required only when Web selected (active required field)
  if (s.platforms?.includes('Web')) {
    if (!s.webFunnel?.items?.length || s.webFunnel.items.length < 2) return 'webFunnel.items'
  }
  return null
}

function validateStep5(s: WizardState): string | null {
  const hasMobile = s.platforms?.includes('iOS') || s.platforms?.includes('Android')
  const hasWeb    = s.platforms?.includes('Web')
  // MMP required when mobile platforms present
  if (hasMobile && !s.mmp) return 'mmp'
  // AppsFlyer cohort access required only when mmp = 'appsflyer' (active required)
  if (s.mmp === 'appsflyer' && s.appsflyerCohortAccess === null) return 'appsflyerCohortAccess'
  // Web attribution required only when Web selected (active required)
  if (hasWeb && !s.webAttrSources?.length) return 'webAttrSources'
  // Ad spend sources always required
  if (!s.adSpendSources?.length) return 'adSpendSources'
  // History required per selected platform
  const missingHistory = s.platforms?.some((p) => !s.history?.[p])
  if (missingHistory) return 'history'
  return null
}

// Steps 6 and 7 are optional — always return null (never block)
function validateStep6(_s: WizardState): string | null { return null }
function validateStep7(_s: WizardState): string | null { return null }
// Step 8 (Review) collects no new required fields
function validateStep8(_s: WizardState): string | null { return null }

const STEP_VALIDATORS: Record<number, (s: WizardState) => string | null> = {
  1: validateStep1,
  2: validateStep2,
  3: validateStep3,
  4: validateStep4,
  5: validateStep5,
  6: validateStep6,
  7: validateStep7,
  8: validateStep8,
}

// ── Public API ──────────────────────────────────────────────────────────────

/**
 * Returns the validation status of a single wizard step.
 * 'done'  — all active required fields valid.
 * 'error' — at least one active required field is missing/invalid.
 * 'dimmed'— not applicable (reserved; currently not returned — parent-question
 *            gating is handled within each validator by checking platform/business).
 */
export function validateStep(step: number, state: WizardState): StepStatus {
  const validator = STEP_VALIDATORS[step]
  if (!validator) return 'done'
  return validator(state) === null ? 'done' : 'error'
}

/**
 * Returns a list of validation summary entries for every required step that has
 * at least one invalid active required field.  An empty array means the wizard
 * is complete and the Approve button may be enabled.
 *
 * Steps 6 (External Factors) and 7 (Objectives) are optional and never appear.
 */
export function buildValidationSummary(state: WizardState): ValidationSummaryEntry[] {
  const entries: ValidationSummaryEntry[] = []
  // Required steps only: 1–5, 8 (6 and 7 are optional)
  const requiredSteps = [1, 2, 3, 4, 5, 8]
  for (const step of requiredSteps) {
    const validator = STEP_VALIDATORS[step]
    const firstInvalidField = validator(state)
    if (firstInvalidField !== null) {
      entries.push({
        step,
        stepLabel: STEP_LABELS[step] ?? `Step ${step}`,
        firstInvalidField,
        message: `Step ${step} (${STEP_LABELS[step]}) has incomplete required fields.`,
      })
    }
  }
  return entries
}
```

- [ ] **Step 4.2 — Run validation tests (expect PASS)**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.test.ts 2>&1 | tail -30
```

Expected: all tests pass, no failures.

- [ ] **Step 4.3 — Commit validation**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.ts \
        packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.test.ts
git commit -m "feat(onboarding): add logic/validation — per-step gating + output summary (F7)"
```

---

### Task 5 — Full test run + lint

- [ ] **Step 5.1 — Run full advertiser test suite**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx vitest run 2>&1 | tail -30
```

Expected: all tests pass including the new F7 tests. Zero regressions.

- [ ] **Step 5.2 — Markdown lint**

```bash
cd /Users/mukey/Documents/kochava-projects/aim-onboarding-tool/docs
npx markdownlint-cli2 "docs/16-PRDs/061-aim-onboarding-decision-tool/plans/F7-logic-validation-milestones.md"
```

Expected: no errors.

- [ ] **Step 5.3 — TypeScript type check**

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/ko-diagnostic-toolkit/prefetched_repos/frontend-mos/packages/advertiser
npx tsc --noEmit 2>&1 | grep -i "AimOnboarding\|validation\|milestones" | head -20
```

Expected: no type errors in the new files.

---

## Self-Review Checklist

**Spec coverage:**

| Requirement | Task |
|---|---|
| `nextMonday` rounds on-or-after Monday (§4.2, §8a) | Task 1 test, Task 2 impl |
| `est.start = anchorDate + weekOffset` (§8a) | Task 1 test, Task 2 impl |
| `est.end = est.start + durationWeeks` (§8a) | Task 1 test, Task 2 impl |
| Cascade: delay → shift all subsequent milestones (§8a) | Task 1 cascade tests |
| Cumulative cascade (two delays) | Task 1 cumulative test |
| 6-milestone template, both tiers, correct weekOffsets (§4.2 Tab 2) | Task 1 template tests |
| `durationWeeks = 0` for sub-day milestones | Task 1 test |
| `anchorDate` injected — no `new Date()` inside | Task 2 impl |
| Per-step validation: conditional required-ness (dimmed fields excluded) | Task 3 & 4 |
| `platformShares` only required when ≥2 platforms | Task 3 & 4 |
| `modelling` only required when Web + mobile | Task 3 & 4 |
| `campaignGroupingChoice` only required when `campaignGrouping = true` (§3.9) | Task 3 & 4 |
| `appsflyerCohortAccess` only required when `mmp = 'appsflyer'` | Task 3 & 4 |
| `webAttrSources`, `webFunnel` only required when Web selected | Task 3 & 4 |
| Steps 6 and 7 never block (optional) | Task 3 & 4 |
| `buildValidationSummary` lists offending steps + `firstInvalidField` (§8a) | Task 3 & 4 |
| Approve gate: blocked when summary is non-empty (§8a) | Task 3 test |
