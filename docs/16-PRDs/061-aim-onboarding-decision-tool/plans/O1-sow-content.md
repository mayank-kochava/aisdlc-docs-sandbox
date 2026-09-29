---
id: plan-o1
title: "O1 — SoW Content"
---

## O1 — SoW Content: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `output/SowTab.vue` — the read-only generated Scope-of-Work document body rendered as a monospace typed document, including the DRAFT stamp, tier-recommendation band, all nine document sections (Engagement, Platform & region, Campaign types, Client ad channels, Business model & KPI, Funnel events, Data sources, External factors, Objectives), and associated Vitest component tests.

**Architecture:** `SowTab.vue` is a **pure display component** — it takes wizard state and computed values (`tier`, `modelCount`) as typed props and emits nothing. All business logic lives upstream in `useAimOnboarding` (F3) and `recommendTier` (F5); this component contains only display formatting: label maps, funnel ordering, empty-state guards, and the brand ad-stock flag. The `.paper` card targets visual parity with the `ui-preview.html` `#out` screen's `.paper` block. The approval block, inline notes, and PDF button are **not in this plan** (O2/O3). The validation summary and tab chrome are **not in this plan** (O7).

**Tech Stack:** Vue 3.5, Vuetify 3.12, TypeScript 5, Vitest 4.x + `@vue/test-utils`. Token source: MOS theme (`rgb(var(--v-theme-*))`), JetBrains Mono on the `.paper` card only.

---

## Scope boundary (read before implementing)

O1 renders only the document body inside `.paper`. This table locks what is in vs out:

| Element | Plan |
|---------|------|
| DRAFT stamp | **O1** |
| Generated date line | **O1** |
| Tier-recommendation band | **O1** |
| Nine `.ds` document sections | **O1** |
| Funnel chip → chevron sequence | **O1** |
| Objectives placeholder (all-empty guard) | **O1** |
| Brand ~30-day ad-stock flag (§9) | **O1** |
| Approval block (name/title/timestamp) | O2 |
| Inline section notes textareas | O2 |
| Download-PDF button / `.doc-meta` | O3 |
| `.ohead` header, `.gen` line, `.otabs` strip | O7 |
| Validation summary banner | O7 |

---

## Files

| Action | Path |
|--------|------|
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue` |
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts` |
| **Read — do not modify** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts` (F2 — import `AimWizardState`) |

---

## Dependencies

- **F2** (`interfaces/aimOnboarding.ts`) — `AimWizardState` type. If F2 is not yet merged, declare a `SowTabStub` locally in the spec file (shown in Task 1) and replace the import once F2 lands.
- **F3** (`useAimOnboarding.ts`) — upstream composer of wizard state; provides the computed `tier` and `modelCount` values. O1 does **not** call `recommendTier` directly — those values arrive as props.

---

## State shape consumed (design-consolidated §5)

O1 reads the following wizard fields. All sourced from `AimWizardState`:

```ts
// Identity
companyName: string
projectLeads: Array<{ name: string; email: string }>
dataLeads: Array<{ name: string; email: string }>

// Platform & region
platforms: string[]               // ['iOS','Android','Web']
platformShares: Record<string, number>
modelling: string                 // 'single' | 'separate' | 'unified'
region: string
regionOther: string               // used when region === 'Other'

// Marketing
digitalMediaTypes: string[]       // client ad channels (bug-fix #7)
ua: boolean; uaShare: string
ue: boolean; ueShare: string
brand: boolean; brandShare: string
campaignGrouping: boolean         // note: campaignGrouping, not campaignMerge
campaignGroupingChoice: string

// Business & funnel
business: string                  // 'subscription' | 'ecommerce' | 'gaming'
funnel: {
  items: string[]                 // event keys in order
  kpi: string                     // KPI event key
  names: Record<string, string>   // display label per event key
}

// Data sources
mmp: string
mmpCollection: string
webAttrSources: string[]
adSpendSources: string[]
spendCollection: string
history: Record<string, '12_24' | '24_36' | '36_plus'>

// External factors
externalFactors: string[]
externalFactorsNotes: string

// Objectives (all optional)
objectivesGoal: string
objectivesSuccess: string
objectivesMarketing: string

// Approval status (for DRAFT stamp gate)
approvedAt: string | null
```

Props passed by the output shell (not from raw state):

```ts
tier: 'aim_x' | 'aim_pro'
modelCount: number
generatedAt: string   // ISO timestamp formatted for display
```

---

## Display label maps (copy into the component)

These maps live inside `SowTab.vue` (not in a separate file — used only here):

```ts
const REGION_LABELS: Record<string, string> = {
  US: 'United States',
  UK: 'United Kingdom',
  DE: 'Germany',
  FR: 'France',
  CA: 'Canada',
  Other: 'Other',
}

const BUSINESS_LABELS: Record<string, string> = {
  subscription: 'Subscription',
  ecommerce: 'E-commerce',
  gaming: 'Gaming',
}

const MEDIA_TYPE_LABELS: Record<string, string> = {
  self_attributed: 'Self-attributed (Meta, Google, TikTok, ASA)',
  standard_attributed: 'Standard attributed (DSP/programmatic)',
  affiliates: 'Affiliates/partner networks',
  other: 'Other',
}

const MMP_LABELS: Record<string, string> = {
  appsflyer: 'AppsFlyer',
  adjust: 'Adjust',
  singular: 'Singular',
  branch: 'Branch',
  kochava: 'Kochava',
  other_none: 'Other/none',
}

const HISTORY_LABELS: Record<string, string> = {
  '12_24': '12–24 months',
  '24_36': '24–36 months',
  '36_plus': '36+ months',
}

const TIER_LABELS: Record<string, string> = {
  aim_x: 'AIM X',
  aim_pro: 'AIM Pro',
}

const EXTERNAL_FACTOR_LABELS: Record<string, string> = {
  promotional: 'Promotional activity',
  competitor: 'Competitor activity',
  product_launch: 'Major product launch/rebrand',
  pr_media: 'PR/earned media spike',
  macro_economic: 'Macro economic event',
  other: 'Other',
}
```

---

## Task 1: Write failing component tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts`

- [ ] **Step 1: Create the spec file**

```ts
// packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts

import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import SowTab from '../SowTab.vue'

// Minimal state stub — replace with:
// import type { AimWizardState } from '../../interfaces/aimOnboarding'
// once plan F2 is merged.
interface SowStub {
  companyName: string
  projectLeads: Array<{ name: string; email: string }>
  dataLeads: Array<{ name: string; email: string }>
  platforms: string[]
  platformShares: Record<string, number>
  modelling: string
  region: string
  regionOther: string
  digitalMediaTypes: string[]
  ua: boolean; uaShare: string
  ue: boolean; ueShare: string
  brand: boolean; brandShare: string
  campaignGrouping: boolean
  campaignGroupingChoice: string
  business: string
  funnel: { items: string[]; kpi: string; names: Record<string, string> }
  mmp: string
  mmpCollection: string
  webAttrSources: string[]
  adSpendSources: string[]
  spendCollection: string
  history: Record<string, string>
  externalFactors: string[]
  externalFactorsNotes: string
  objectivesGoal: string
  objectivesSuccess: string
  objectivesMarketing: string
  approvedAt: string | null
}

function makeState(overrides: Partial<SowStub> = {}): SowStub {
  return {
    companyName: 'Acme Corp',
    projectLeads: [{ name: 'Jane Doe', email: 'jane@acme.com' }],
    dataLeads: [{ name: 'John Smith', email: 'john@acme.com' }],
    platforms: ['iOS', 'Android'],
    platformShares: { iOS: 60, Android: 40 },
    modelling: 'unified',
    region: 'US',
    regionOther: '',
    digitalMediaTypes: ['self_attributed', 'standard_attributed'],
    ua: true, uaShare: '60',
    ue: true, ueShare: '40',
    brand: false, brandShare: '',
    campaignGrouping: false,
    campaignGroupingChoice: '',
    business: 'subscription',
    funnel: {
      items: ['install', 'registration', 'trial_start', 'subscription_start', 'revenue'],
      kpi: 'trial_start',
      names: {
        install: 'Install',
        registration: 'Registration',
        trial_start: 'Trial Start',
        subscription_start: 'Subscription',
        revenue: 'Revenue',
      },
    },
    mmp: 'appsflyer',
    mmpCollection: 'direct_api',
    webAttrSources: [],
    adSpendSources: ['mmp_direct'],
    spendCollection: 'direct_api',
    history: { iOS: '24_36', Android: '24_36' },
    externalFactors: [],
    externalFactorsNotes: '',
    objectivesGoal: '',
    objectivesSuccess: '',
    objectivesMarketing: '',
    approvedAt: null,
    ...overrides,
  }
}

const defaultProps = {
  state: makeState(),
  tier: 'aim_x' as const,
  modelCount: 1,
  generatedAt: '11 Jun 2026',
}

function mountSow(props: Partial<typeof defaultProps> = {}) {
  return mount(SowTab, { props: { ...defaultProps, ...props } })
}

// ─── DRAFT stamp ─────────────────────────────────────────────────────────────

describe('DRAFT stamp', () => {
  it('shows DRAFT when approvedAt is null', () => {
    const w = mountSow({ state: makeState({ approvedAt: null }) })
    expect(w.find('[data-testid="sow-draft"]').exists()).toBe(true)
  })

  it('hides DRAFT when approvedAt is set', () => {
    const w = mountSow({ state: makeState({ approvedAt: '2026-06-16T10:00:00Z' }) })
    expect(w.find('[data-testid="sow-draft"]').exists()).toBe(false)
  })
})

// ─── Generated date ───────────────────────────────────────────────────────────

describe('generated date', () => {
  it('renders the generatedAt prop', () => {
    const w = mountSow({ generatedAt: '11 Jun 2026' })
    expect(w.find('[data-testid="sow-generated-date"]').text()).toBe('Generated 11 Jun 2026')
  })
})

// ─── Tier band ────────────────────────────────────────────────────────────────

describe('tier band', () => {
  it('shows AIM X for tier aim_x', () => {
    const w = mountSow({ tier: 'aim_x' })
    expect(w.find('[data-testid="sow-tier-band"]').text()).toContain('AIM X')
  })

  it('shows AIM Pro for tier aim_pro', () => {
    const w = mountSow({ tier: 'aim_pro' })
    expect(w.find('[data-testid="sow-tier-band"]').text()).toContain('AIM Pro')
  })
})

// ─── Engagement section ───────────────────────────────────────────────────────

describe('Engagement section', () => {
  it('renders company name', () => {
    const w = mountSow({ state: makeState({ companyName: 'BrightFit Inc.' }) })
    const section = w.find('[data-testid="sow-section-engagement"]')
    expect(section.text()).toContain('BrightFit Inc.')
  })

  it('renders project lead name and email', () => {
    const w = mountSow()
    const section = w.find('[data-testid="sow-section-engagement"]')
    expect(section.text()).toContain('Jane Doe')
    expect(section.text()).toContain('jane@acme.com')
  })

  it('renders data lead name when different from project lead', () => {
    const w = mountSow()
    const section = w.find('[data-testid="sow-section-engagement"]')
    expect(section.text()).toContain('John Smith')
  })
})

// ─── Platform & region section ────────────────────────────────────────────────

describe('Platform & region section', () => {
  it('renders comma-joined platform list', () => {
    const w = mountSow({ state: makeState({ platforms: ['iOS', 'Android'] }) })
    const section = w.find('[data-testid="sow-section-platform"]')
    expect(section.text()).toContain('iOS, Android')
  })

  it('renders region label for known region code', () => {
    const w = mountSow({ state: makeState({ region: 'US' }) })
    expect(w.find('[data-testid="sow-section-platform"]').text()).toContain('United States')
  })

  it('renders regionOther text when region is Other', () => {
    const w = mountSow({ state: makeState({ region: 'Other', regionOther: 'Brazil' }) })
    expect(w.find('[data-testid="sow-section-platform"]').text()).toContain('Brazil')
  })

  it('renders modelling approach label', () => {
    const w = mountSow({ state: makeState({ modelling: 'unified' }) })
    expect(w.find('[data-testid="sow-section-platform"]').text()).toContain('Unified model')
  })

  it('renders model count', () => {
    const w = mountSow({ modelCount: 2 })
    expect(w.find('[data-testid="sow-section-platform"]').text()).toContain('2 estimated model')
  })
})

// ─── Campaign types section ───────────────────────────────────────────────────

describe('Campaign types section', () => {
  it('lists selected campaign types with shares', () => {
    const w = mountSow({ state: makeState({ ua: true, uaShare: '60', ue: true, ueShare: '40', brand: false }) })
    const section = w.find('[data-testid="sow-section-campaigns"]')
    expect(section.text()).toContain('UA')
    expect(section.text()).toContain('60')
    expect(section.text()).not.toContain('Brand')
  })
})

// ─── Client ad channels section (bug-fix #7) ─────────────────────────────────

describe('Client ad channels section', () => {
  it('renders selected digitalMediaTypes as human labels', () => {
    const w = mountSow({
      state: makeState({ digitalMediaTypes: ['self_attributed', 'affiliates'] }),
    })
    const section = w.find('[data-testid="sow-section-ad-channels"]')
    expect(section.text()).toContain('Self-attributed')
    expect(section.text()).toContain('Affiliates')
  })

  it('section is present even with one digital media type', () => {
    const w = mountSow({ state: makeState({ digitalMediaTypes: ['standard_attributed'] }) })
    expect(w.find('[data-testid="sow-section-ad-channels"]').exists()).toBe(true)
  })
})

// ─── Business model & KPI section ────────────────────────────────────────────

describe('Business model & KPI section', () => {
  it('renders business model label', () => {
    const w = mountSow({ state: makeState({ business: 'subscription' }) })
    expect(w.find('[data-testid="sow-section-business"]').text()).toContain('Subscription')
  })

  it('renders KPI event display name', () => {
    const w = mountSow()
    expect(w.find('[data-testid="sow-section-business"]').text()).toContain('Trial Start')
  })

  it('renders ecommerce business model label', () => {
    const w = mountSow({ state: makeState({ business: 'ecommerce' }) })
    expect(w.find('[data-testid="sow-section-business"]').text()).toContain('E-commerce')
  })
})

// ─── Funnel section ───────────────────────────────────────────────────────────

describe('Funnel section', () => {
  it('renders each funnel event chip in order', () => {
    const w = mountSow()
    const chips = w.findAll('[data-testid="sow-funnel-chip"]')
    expect(chips).toHaveLength(5)
    expect(chips[0].text()).toBe('Install')
    expect(chips[4].text()).toBe('Revenue')
  })

  it('marks KPI chip with the kpi modifier', () => {
    const w = mountSow()
    const kpiChip = w.find('[data-testid="sow-funnel-chip"][data-kpi="true"]')
    expect(kpiChip.text()).toBe('Trial Start')
  })

  it('renders chevrons between chips', () => {
    const w = mountSow()
    const chevrons = w.findAll('[data-testid="sow-funnel-chevron"]')
    // N chips → N-1 chevrons
    expect(chevrons).toHaveLength(4)
  })
})

// ─── Data sources section ─────────────────────────────────────────────────────

describe('Data sources section', () => {
  it('renders MMP name', () => {
    const w = mountSow({ state: makeState({ mmp: 'appsflyer' }) })
    expect(w.find('[data-testid="sow-section-data"]').text()).toContain('AppsFlyer')
  })

  it('renders history period for each platform', () => {
    const w = mountSow({ state: makeState({ history: { iOS: '12_24', Android: '36_plus' } }) })
    const section = w.find('[data-testid="sow-section-data"]')
    expect(section.text()).toContain('12–24 months')
    expect(section.text()).toContain('36+ months')
  })
})

// ─── External factors section ─────────────────────────────────────────────────

describe('External factors section', () => {
  it('renders selected factor labels', () => {
    const w = mountSow({
      state: makeState({ externalFactors: ['promotional', 'competitor'] }),
    })
    const section = w.find('[data-testid="sow-section-external"]')
    expect(section.text()).toContain('Promotional activity')
    expect(section.text()).toContain('Competitor activity')
  })

  it('renders free-text notes when provided', () => {
    const w = mountSow({
      state: makeState({
        externalFactors: ['macro_economic'],
        externalFactorsNotes: 'Recession concerns Q3',
      }),
    })
    expect(w.find('[data-testid="sow-section-external"]').text()).toContain('Recession concerns Q3')
  })

  it('renders "None selected" when no factors are chosen', () => {
    const w = mountSow({ state: makeState({ externalFactors: [] }) })
    expect(w.find('[data-testid="sow-section-external"]').text()).toContain('None selected')
  })
})

// ─── Objectives section ───────────────────────────────────────────────────────

describe('Objectives section', () => {
  it('renders objectivesGoal when provided', () => {
    const w = mountSow({
      state: makeState({ objectivesGoal: 'Reduce CAC by 20%' }),
    })
    expect(w.find('[data-testid="sow-section-objectives"]').text()).toContain('Reduce CAC by 20%')
  })

  it('renders placeholder when all three objectives fields are empty', () => {
    const w = mountSow({
      state: makeState({ objectivesGoal: '', objectivesSuccess: '', objectivesMarketing: '' }),
    })
    const placeholder = w.find('[data-testid="sow-objectives-placeholder"]')
    expect(placeholder.exists()).toBe(true)
    expect(placeholder.text()).toContain('No objectives captured')
  })

  it('does NOT render placeholder when any objectives field is non-empty', () => {
    const w = mountSow({
      state: makeState({ objectivesGoal: 'Grow revenue', objectivesSuccess: '', objectivesMarketing: '' }),
    })
    expect(w.find('[data-testid="sow-objectives-placeholder"]').exists()).toBe(false)
  })

  it('placeholder carries sow-no-print class (for O3 print stylesheet)', () => {
    const w = mountSow({
      state: makeState({ objectivesGoal: '', objectivesSuccess: '', objectivesMarketing: '' }),
    })
    expect(w.find('[data-testid="sow-objectives-placeholder"]').classes()).toContain('sow-no-print')
  })
})

// ─── Brand ad-stock flag (§9) ─────────────────────────────────────────────────

describe('Brand campaign ad-stock flag', () => {
  it('shows ~30-day ad-stock note when brand campaigns are selected', () => {
    const w = mountSow({
      state: makeState({ brand: true, brandShare: '20' }),
    })
    expect(w.find('[data-testid="sow-brand-adstock"]').exists()).toBe(true)
    expect(w.find('[data-testid="sow-brand-adstock"]').text()).toContain('30-day')
  })

  it('does NOT show ad-stock note when brand is false', () => {
    const w = mountSow({ state: makeState({ brand: false }) })
    expect(w.find('[data-testid="sow-brand-adstock"]').exists()).toBe(false)
  })
})

// ─── Scope-lock negative tests (O2 / O7 elements must NOT be rendered) ────────

describe('scope lock — O2 and O7 elements are not rendered by O1', () => {
  it('approval block (name/title inputs) is NOT rendered', () => {
    const w = mountSow()
    expect(w.find('[data-testid="sow-approval-block"]').exists()).toBe(false)
  })

  it('inline section notes textareas are NOT rendered', () => {
    const w = mountSow()
    expect(w.findAll('[data-testid="sow-section-notes"]')).toHaveLength(0)
  })
})
```

- [ ] **Step 2: Run the tests — expect FAIL (component not found)**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts
```

Expected output contains: `Cannot find module '../SowTab.vue'`

---

## Task 2: Build `SowTab.vue` — shell + DRAFT + generated date + tier band

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue`

- [ ] **Step 1: Create the component with the paper shell, DRAFT stamp, date, and tier band**

```vue
<!-- packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue -->
<template>
  <div class="sow-paper" :class="{ 'sow-paper--approved': !!state.approvedAt }">
    <!-- DRAFT stamp — hidden once approved -->
    <div
      v-if="!state.approvedAt"
      data-testid="sow-draft"
      class="sow-draft"
      aria-label="Draft document"
    >
      DRAFT
    </div>

    <!-- Generated date -->
    <p data-testid="sow-generated-date" class="sow-meta">Generated {{ generatedAt }}</p>

    <!-- Tier recommendation band -->
    <div data-testid="sow-tier-band" class="sow-tierband">
      <span class="sow-tierband__label">Tier recommendation</span>
      <span class="sow-tierband__value">{{ TIER_LABELS[tier] ?? tier }}</span>
      <span class="sow-tierband__desc">— best suited based on your budget &amp; campaign mix</span>
    </div>

    <!-- Document sections rendered in Task 3 -->
  </div>
</template>

<script setup lang="ts">
// Replace SowStateProps with AimWizardState from F2 once merged:
// import type { AimWizardState } from '../interfaces/aimOnboarding'

interface SowStateProps {
  companyName: string
  projectLeads: Array<{ name: string; email: string }>
  dataLeads: Array<{ name: string; email: string }>
  platforms: string[]
  platformShares: Record<string, number>
  modelling: string
  region: string
  regionOther: string
  digitalMediaTypes: string[]
  ua: boolean
  uaShare: string
  ue: boolean
  ueShare: string
  brand: boolean
  brandShare: string
  campaignGrouping: boolean
  campaignGroupingChoice: string
  business: string
  funnel: { items: string[]; kpi: string; names: Record<string, string> }
  mmp: string
  mmpCollection: string
  webAttrSources: string[]
  adSpendSources: string[]
  spendCollection: string
  history: Record<string, string>
  externalFactors: string[]
  externalFactorsNotes: string
  objectivesGoal: string
  objectivesSuccess: string
  objectivesMarketing: string
  approvedAt: string | null
}

const props = defineProps<{
  state: SowStateProps
  tier: 'aim_x' | 'aim_pro'
  modelCount: number
  generatedAt: string
}>()

// ── Display label maps ────────────────────────────────────────────────────────

const REGION_LABELS: Record<string, string> = {
  US: 'United States',
  UK: 'United Kingdom',
  DE: 'Germany',
  FR: 'France',
  CA: 'Canada',
  Other: 'Other',
}

const BUSINESS_LABELS: Record<string, string> = {
  subscription: 'Subscription',
  ecommerce: 'E-commerce',
  gaming: 'Gaming',
}

const MEDIA_TYPE_LABELS: Record<string, string> = {
  self_attributed: 'Self-attributed (Meta, Google, TikTok, ASA)',
  standard_attributed: 'Standard attributed (DSP/programmatic)',
  affiliates: 'Affiliates/partner networks',
  other: 'Other',
}

const MMP_LABELS: Record<string, string> = {
  appsflyer: 'AppsFlyer',
  adjust: 'Adjust',
  singular: 'Singular',
  branch: 'Branch',
  kochava: 'Kochava',
  other_none: 'Other/none',
}

const HISTORY_LABELS: Record<string, string> = {
  '12_24': '12–24 months',
  '24_36': '24–36 months',
  '36_plus': '36+ months',
}

const TIER_LABELS: Record<string, string> = {
  aim_x: 'AIM X',
  aim_pro: 'AIM Pro',
}

const MODELLING_LABELS: Record<string, string> = {
  unified: 'Unified model',
  separate: 'Separate models',
  single: 'Single model',
}

const EXTERNAL_FACTOR_LABELS: Record<string, string> = {
  promotional: 'Promotional activity',
  competitor: 'Competitor activity',
  product_launch: 'Major product launch/rebrand',
  pr_media: 'PR/earned media spike',
  macro_economic: 'Macro economic event',
  other: 'Other',
}

// ── Computed display values ───────────────────────────────────────────────────

const regionLabel = computed(() => {
  if (props.state.region === 'Other') return props.state.regionOther || 'Other'
  return REGION_LABELS[props.state.region] ?? props.state.region
})

const platformList = computed(() => props.state.platforms.join(', '))

const modellingLabel = computed(() => MODELLING_LABELS[props.state.modelling] ?? props.state.modelling)

const activeCampaigns = computed(() => {
  const list: Array<{ label: string; share: string }> = []
  if (props.state.ua) list.push({ label: 'UA', share: props.state.uaShare })
  if (props.state.ue) list.push({ label: 'UE', share: props.state.ueShare })
  if (props.state.brand) list.push({ label: 'Brand', share: props.state.brandShare })
  return list
})

const adChannelLabels = computed(() =>
  props.state.digitalMediaTypes.map((k) => MEDIA_TYPE_LABELS[k] ?? k)
)

const businessLabel = computed(() => BUSINESS_LABELS[props.state.business] ?? props.state.business)

const kpiLabel = computed(() => props.state.funnel.names[props.state.funnel.kpi] ?? props.state.funnel.kpi)

const funnelItems = computed(() =>
  props.state.funnel.items.map((key) => ({
    key,
    label: props.state.funnel.names[key] ?? key,
    isKpi: key === props.state.funnel.kpi,
  }))
)

const mmpLabel = computed(() => MMP_LABELS[props.state.mmp] ?? props.state.mmp)

const historyLines = computed(() =>
  Object.entries(props.state.history).map(([platform, val]) => ({
    platform,
    label: HISTORY_LABELS[val] ?? val,
  }))
)

const externalFactorLabels = computed(() =>
  props.state.externalFactors.map((k) => EXTERNAL_FACTOR_LABELS[k] ?? k)
)

const objectivesEmpty = computed(() =>
  !props.state.objectivesGoal && !props.state.objectivesSuccess && !props.state.objectivesMarketing
)
</script>

<script lang="ts">
import { computed } from 'vue'
</script>

<style scoped>
.sow-paper {
  background: #fff;
  border: 1px solid rgb(var(--v-theme-outline));
  border-radius: 8px;
  padding: 36px 40px;
  position: relative;
  font-family: 'JetBrains Mono', ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: 13px;
  line-height: 1.75;
  color: rgb(var(--v-theme-on-surface));
  box-shadow: 0 1px 2px rgba(16, 24, 40, 0.04);
}

/* Form controls inside the paper keep Inter (for O2 inputs) */
.sow-paper :deep(.v-field),
.sow-paper :deep(.v-btn) {
  font-family: Inter, system-ui, sans-serif;
}

.sow-draft {
  position: absolute;
  top: 26px;
  right: 34px;
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.22em;
  color: rgb(var(--v-theme-outline));
  border: 2px solid rgb(var(--v-theme-outline));
  border-radius: 6px;
  padding: 5px 14px;
}

.sow-meta {
  font-size: 11px;
  color: rgb(var(--v-theme-on-surface-variant));
  letter-spacing: 0.02em;
  margin: 0 0 4px;
}

.sow-tierband {
  display: flex;
  align-items: center;
  gap: 10px;
  background: rgba(var(--v-theme-success), 0.1);
  border: 1px solid rgba(var(--v-theme-success), 0.2);
  border-radius: 8px;
  padding: 11px 14px;
  margin: 18px 0;
}

.sow-tierband__label {
  font-size: 10px;
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: rgb(var(--v-theme-success));
}

.sow-tierband__value {
  font-size: 13px;
  font-weight: 700;
}

.sow-tierband__desc {
  font-size: 12px;
  color: rgb(var(--v-theme-on-surface-variant));
}
</style>
```

- [ ] **Step 2: Run the DRAFT + tier band tests — expect those cases to PASS**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts --reporter=verbose 2>&1 | grep -E 'PASS|FAIL|✓|✗|×'
```

Expected: `DRAFT stamp` and `tier band` and `generated date` describe blocks pass; section tests still fail (elements not yet rendered).

---

## Task 3: Add all nine document sections

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue`

- [ ] **Step 1: Replace the `<!-- Document sections rendered in Task 3 -->` comment with the nine sections**

In `SowTab.vue`, replace the placeholder comment inside `<template>` with:

```vue
    <!-- 1. Engagement -->
    <div data-testid="sow-section-engagement" class="sow-section">
      <div class="sow-section__label">Engagement</div>
      <div class="sow-section__value">
        {{ state.companyName }}
        <template v-if="state.projectLeads.length">
          &nbsp;·&nbsp;Project lead:
          <template v-for="(lead, i) in state.projectLeads" :key="i">
            {{ lead.name }} · {{ lead.email }}<template v-if="i < state.projectLeads.length - 1">, </template>
          </template>
        </template>
        <template v-if="state.dataLeads.length">
          &nbsp;·&nbsp;Data lead:
          <template v-for="(lead, i) in state.dataLeads" :key="i">
            {{ lead.name }}<template v-if="i < state.dataLeads.length - 1">, </template>
          </template>
        </template>
      </div>
    </div>

    <!-- 2. Platform & region -->
    <div data-testid="sow-section-platform" class="sow-section">
      <div class="sow-section__label">Platform &amp; region</div>
      <div class="sow-section__value">
        {{ platformList }} · {{ regionLabel }} · {{ modellingLabel }} · {{ modelCount }} estimated model{{ modelCount !== 1 ? 's' : '' }}
      </div>
    </div>

    <!-- 3. Campaign types -->
    <div data-testid="sow-section-campaigns" class="sow-section">
      <div class="sow-section__label">Campaign types</div>
      <div class="sow-section__value">
        <span v-for="(c, i) in activeCampaigns" :key="c.label">
          {{ c.label }} {{ c.share }}%<template v-if="i < activeCampaigns.length - 1">, </template>
        </span>
        <!-- Brand ad-stock flag (§9) -->
        <span
          v-if="state.brand"
          data-testid="sow-brand-adstock"
          class="sow-brand-adstock"
        >
          &nbsp;— Brand campaigns carry ~30-day ad stock (vs 7-day for UA/UE)
        </span>
      </div>
    </div>

    <!-- 4. Client ad channels (bug-fix #7) -->
    <div data-testid="sow-section-ad-channels" class="sow-section">
      <div class="sow-section__label">Client ad channels</div>
      <div class="sow-section__value">
        {{ adChannelLabels.join(', ') || '—' }}
      </div>
    </div>

    <!-- 5. Business model & KPI -->
    <div data-testid="sow-section-business" class="sow-section">
      <div class="sow-section__label">Business &amp; KPI</div>
      <div class="sow-section__value">
        {{ businessLabel }} · Primary KPI: {{ kpiLabel }}
      </div>
    </div>

    <!-- 6. Funnel events (chip → chevron sequence) -->
    <div data-testid="sow-section-funnel" class="sow-section">
      <div class="sow-section__label">Funnel events</div>
      <div class="sow-funnel" role="list" aria-label="Conversion funnel">
        <template v-for="(item, i) in funnelItems" :key="item.key">
          <span
            data-testid="sow-funnel-chip"
            :data-kpi="item.isKpi ? 'true' : undefined"
            :class="['sow-funnel__chip', { 'sow-funnel__chip--kpi': item.isKpi }]"
            role="listitem"
          >{{ item.label }}</span>
          <span
            v-if="i < funnelItems.length - 1"
            data-testid="sow-funnel-chevron"
            class="sow-funnel__chevron"
            aria-hidden="true"
          >›</span>
        </template>
      </div>
    </div>

    <!-- 7. Data sources -->
    <div data-testid="sow-section-data" class="sow-section">
      <div class="sow-section__label">Data sources</div>
      <div class="sow-section__value">
        MMP: {{ mmpLabel }}
        <template v-if="historyLines.length">
          &nbsp;·&nbsp;History:
          <span v-for="(h, i) in historyLines" :key="h.platform">
            {{ h.platform }} {{ h.label }}<template v-if="i < historyLines.length - 1">, </template>
          </span>
        </template>
        <template v-if="state.webAttrSources.length">
          &nbsp;·&nbsp;Web attribution: {{ state.webAttrSources.join(', ') }}
        </template>
      </div>
    </div>

    <!-- 8. External factors -->
    <div data-testid="sow-section-external" class="sow-section">
      <div class="sow-section__label">External factors</div>
      <div class="sow-section__value">
        <template v-if="externalFactorLabels.length">
          {{ externalFactorLabels.join(', ') }}
          <template v-if="state.externalFactorsNotes">
            <br />{{ state.externalFactorsNotes }}
          </template>
        </template>
        <template v-else>None selected</template>
      </div>
    </div>

    <!-- 9. Objectives (bug-fix #6) -->
    <div data-testid="sow-section-objectives" class="sow-section">
      <div class="sow-section__label">Objectives</div>
      <div class="sow-section__value">
        <template v-if="!objectivesEmpty">
          <p v-if="state.objectivesGoal">{{ state.objectivesGoal }}</p>
          <p v-if="state.objectivesSuccess">{{ state.objectivesSuccess }}</p>
          <p v-if="state.objectivesMarketing">{{ state.objectivesMarketing }}</p>
        </template>
        <!-- Placeholder: shown when all three fields empty; excluded from PDF (O3 uses .sow-no-print) -->
        <p
          v-else
          data-testid="sow-objectives-placeholder"
          class="sow-objectives-placeholder sow-no-print"
        >
          No objectives captured — add them in Step 7 or in the notes field below.
        </p>
      </div>
    </div>
```

- [ ] **Step 2: Add the remaining scoped styles for sections and funnel chips**

Append to `<style scoped>` in `SowTab.vue`:

```css
.sow-section {
  padding: 16px 0;
  border-bottom: 1px solid rgb(var(--v-theme-outline));
}

.sow-section:last-child {
  border-bottom: none;
}

.sow-section__label {
  font-size: 10.5px;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: rgb(var(--v-theme-primary));
  margin-bottom: 7px;
}

.sow-section__value {
  font-size: 13.5px;
  line-height: 1.7;
}

.sow-funnel {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  align-items: center;
  margin-top: 6px;
}

.sow-funnel__chip {
  background: rgb(var(--v-theme-surface-variant));
  border: 1px solid rgb(var(--v-theme-outline));
  border-radius: 9999px;
  padding: 3px 12px;
  font-size: 12px;
}

.sow-funnel__chip--kpi {
  background: rgba(var(--v-theme-primary), 0.07);
  border-color: rgba(var(--v-theme-primary), 0.18);
  color: rgb(var(--v-theme-primary));
  font-weight: 600;
}

.sow-funnel__chevron {
  color: rgb(var(--v-theme-on-surface-variant));
  font-size: 14px;
}

.sow-brand-adstock {
  font-size: 12px;
  color: rgb(var(--v-theme-on-surface-variant));
}

.sow-objectives-placeholder {
  color: rgb(var(--v-theme-on-surface-variant));
  font-style: italic;
  margin: 0;
}

/* .sow-no-print is a hook for O3's @media print stylesheet — no print CSS here */
```

- [ ] **Step 3: Run all tests — expect all to PASS**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts
```

Expected:

```text
Test Files  1 passed (1)
Tests       XX passed (XX)
```

---

## Task 4: Fix `<script>` setup, run full suite, commit

- [ ] **Step 1: Consolidate the `<script>` blocks**

The component has two `<script>` blocks from the draft above — merge them into a single `<script setup lang="ts">`. The `computed` import moves to the top of the `<script setup>` block. The final `<script setup lang="ts">` opens with:

```vue
<script setup lang="ts">
import { computed } from 'vue'

// ... rest of the script content
```

Remove the trailing `<script lang="ts">` block entirely.

- [ ] **Step 2: Lint-check the component**

```bash
cd packages/advertiser && npx eslint src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue --max-warnings=0
```

Expected: no errors or warnings.

- [ ] **Step 3: Run the full test suite to confirm no regressions**

```bash
cd packages/advertiser && npm run test:ci
```

Expected: all previously-passing tests still pass.

- [ ] **Step 4: Commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts
git commit -m "feat(aim-onboarding): add SowTab read-only SoW document component (O1)

- JetBrains Mono paper card with DRAFT stamp (gated on approvedAt)
- Tier recommendation band using MOS theme tokens
- Nine document sections: Engagement, Platform & region, Campaign types,
  Client ad channels, Business model & KPI, Funnel events, Data sources,
  External factors, Objectives
- Funnel chip→chevron sequence with KPI chip highlighted
- Objectives placeholder (sow-no-print hook for O3 PDF)
- Brand ~30-day ad-stock flag when brand campaigns selected
- Scope-lock tests assert O2/O7 elements absent
- Bug-fixes #6 (objectives in SoW) and #7 (ad channels in output)"
```

---

## Task 5: Wire F2 import (do after F2 merges)

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue`

- [ ] **Step 1: Replace the local `SowStateProps` interface with the F2 import**

At the top of `<script setup lang="ts">`, replace:

```ts
// Remove: interface SowStateProps { ... }
```

Add:

```ts
import type { AimWizardState } from '../interfaces/aimOnboarding'
```

Change the `state` prop type from `SowStateProps` to `AimWizardState`:

```ts
const props = defineProps<{
  state: AimWizardState
  tier: 'aim_x' | 'aim_pro'
  modelCount: number
  generatedAt: string
}>()
```

- [ ] **Step 2: Update the spec import if field names differ**

If `AimWizardState` uses different optional-ness or names than `SowStub`, update `makeState()` defaults in the spec. Test logic does not change.

- [ ] **Step 3: Run tests and full suite**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts && npm run test:ci
```

Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue \
        packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowTab.spec.ts
git commit -m "refactor(aim-onboarding): wire SowTab state prop to AimWizardState from F2 interfaces"
```

---

## Self-review checklist

### Spec coverage

| §4.2 Tab 1 requirement | Covered by |
|------------------------|-----------|
| Engagement details | Task 3 section 1 + tests |
| Platform & region | Task 3 section 2 + tests |
| Campaign types | Task 3 section 3 + tests |
| Client ad channels (bug-fix #7) | Task 3 section 4 + tests |
| Business model & KPI | Task 3 section 5 + tests |
| Funnel events (chip→chevron, KPI highlighted) | Task 3 section 6 + funnel tests |
| Data sources | Task 3 section 7 + tests |
| External factors | Task 3 section 8 + tests |
| Objectives (from Step 7) | Task 3 section 9 + tests |
| Objectives placeholder when empty (§4.1 Step 7) | `objectivesEmpty` computed + placeholder test |
| Placeholder carries `sow-no-print` class (O3 hook) | placeholder class test |
| Placeholder text excluded from PDF | documented; O3 consumes `.sow-no-print` |
| DRAFT stamp | Task 2 + tests |
| Tier band | Task 2 + tests |
| Brand ~30-day ad-stock flag (§9) | `sow-brand-adstock` element + 2 tests |
| Approval block **NOT here** | scope-lock negative test |
| Inline notes **NOT here** | scope-lock negative test |
| JetBrains Mono on document | `.sow-paper` font-family |
| Form controls stay Inter | `:deep(.v-field)` override |
| Navy (primary token) section labels | `.sow-section__label` uses `rgb(var(--v-theme-primary))` |
| No hardcoded hex — tokens only | all colors use `rgb(var(--v-theme-*))` |
| `data-testid` attributes for all tested elements | throughout template |

### Placeholder scan

No TBD, TODO, "implement later", "similar to", or vague steps found.

### Type consistency

- `SowStateProps` / `AimWizardState` used as the `state` prop type in Tasks 1–4; replaced with `AimWizardState` in Task 5.
- `funnelItems` computed returns `Array<{ key, label, isKpi }>` — template reads `.isKpi`, tests check `data-kpi="true"` attribute. Consistent.
- `activeCampaigns` returns `Array<{ label, share }>` — template reads `.label` and `.share`. Consistent.
- `TIER_LABELS`, `REGION_LABELS` etc. declared once in the component, referenced in template via the computed wrappers. No duplication.
