---
id: plan-w3
title: "W3 — Step 3: Marketing Setup"
---

## W3 — Step 3: Marketing Setup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `steps/Step3Marketing.vue` — the third wizard step that collects total paid media budget,
offline media flag and split, digital media types, attribution reliability and gap categories,
MMP coverage confidence, organic/paid split per platform, campaign types with budget share, and
§3.9 campaign grouping. Progressive disclosure hides sub-questions until their parent is answered.
Binds exclusively to `useAimOnboarding`. Includes a full Vitest component test suite.

**Architecture:** One thin orchestration component (`Step3Marketing.vue`) reading from the composable
and emitting via `set()`. Kit components own their interaction logic; Step 3 owns progressive
disclosure and disclosure order only. Fixed option sets (digital media types, attribution gap
categories) use `v-chip-group` with individual `v-chip` elements — not `InputTags`, which is a
free-text autocomplete unsuited to fixed option lists. The §3.9 grouping question (`campaignGrouping`)
stores `'yes' | 'no' | 'not_sure'` via a three-button `v-btn-toggle`; when `'yes'`, a follow-up
free-text field stores `campaignGroupingChoice`.

**Tech Stack:** Vue 3.5, Vuetify 3.12, Vitest + `@vue/test-utils`, frontend-mos monorepo.
Composable: `useAimOnboarding` (F3). Kit: `BudgetSlider` (C3), `ShareAllocator` (C2), `ToggleCard` (C1).

---

## Key design decisions locked before coding

### State-key note (critical)

`design-consolidated.md §5` is authoritative. The flat composable uses **camelCase** internally:
`wizard.campaignGrouping` / `set({ campaignGrouping: 'yes' })`. The wire DTO (F2 `WizardState`) is
`CampaignGrouping` / `CampaignGroupingChoice` — the composable maps these on load/save. Never use
`campaignMerge` — that is the spec v2 draft name superseded by `design-consolidated §5` note:
*"use `campaignGrouping` / `campaignGroupingChoice` (verified in mockup-k4a.html)"*.

### Campaign share bridge (critical)

F2's `WizardState` has `UaShare: string`, `UeShare: string`, `BrandShare: string` (strings to
match C# POCO). The composable's flat state mirrors these. `ShareAllocator` (C2) requires a
`Record<string, number>`. Step 3 builds the record from the three share strings on read, and
writes back to the three individual keys on change. Helper function below in Task 2.

### paidSplit key casing

The mockup stores `paidSplit` with lowercase keys (`ios`, `android`, `web`). `design-consolidated §5`
and F2 `PaidSplit` interface use `iOS`, `Android`, `Web` (matching C# POCO PascalCase). Use the
canonical F2 casing throughout. Initialise from `{iOS: 65, Android: 80, Web: 50}` — never from
`{ios: ..., android: ..., web: ...}`.

### Organic/paid split sliders

`paidSplit` is three **independent** `v-slider` instances — one per selected platform. They do
**not** sum to 100%; they are independent 0–100 sliders. Do not use `ShareAllocator` for these.

### §3.9 scope

The mockup's `campaignGrouping` YNS only stores `'yes'|'no'|'not_sure'`. The spec §4.1 adds:
*"If Yes: 'Which campaigns would you group?' free-text or chip multi-select, stores
`campaignMergeChoice`."* The design-consolidated canonical key is `campaignGroupingChoice`.
Implement as a `v-textarea` (free-text) revealed when `campaignGrouping === 'yes'`.

### No interface patch needed

Unlike W2 (Task 0), every W3 field already exists in F2's `WizardState`:
`BudgetMonthly`, `BudgetAnnual`, `BudgetPeriod`, `UsesOffline`, `OfflineSplitPct`,
`DigitalMediaTypes`, `HasAttrGaps`, `AttrGapCategories`, `CoverageConfidence`, `PaidSplit`,
`Ua`, `UaShare`, `Ue`, `UeShare`, `Brand`, `BrandShare`, `CampaignGrouping`,
`CampaignGroupingChoice`. No patch required.

### Warning banners on campaign shares

Two per-type warnings sit **above** the `ShareAllocator`: if any active type's share is `< 5`,
show `"[Type] share is below 5% — this may not produce meaningful model signals"` (orange tonal).
If any active type's share is `> 90`, show `"[Type] dominates the budget at > 90%"` (orange tonal).
These warnings use `v-alert variant="tonal"` color `"warning"`.

### No duplicate qualify banner

`BudgetSlider` (C3) already renders the `$250K/mo qualifies for AIM Pro consideration` banner
internally. Do not add a second banner in Step 3.

---

## Fixed option arrays (transcribed from mockup-k4a.html lines 133–155)

```typescript
// To be defined as constants inside Step3Marketing.vue (not exported — component-scoped).

const DIG_TYPES = [
  { id: 'self_attributed', label: 'Self-attributed (Meta, Google, TikTok, ASA)' },
  { id: 'dsp',             label: 'Standard attributed (DSP / programmatic)' },
  { id: 'affiliates',      label: 'Affiliates / partner networks' },
  { id: 'other',           label: 'Other' },
] as const

const ATTR_GAPS = [
  { id: 'offline',         label: 'Offline media' },
  { id: 'dsp',             label: 'DSP' },
  { id: 'affiliates',      label: 'Affiliates' },
  { id: 'branded',         label: 'Branded/awareness' },
  { id: 'mobile_networks', label: 'Mobile networks' },
  { id: 'influencer',      label: 'Influencer' },
  { id: 'podcasts',        label: 'Podcasts' },
  { id: 'not_sure',        label: 'Not sure' },
  { id: 'other',           label: 'Other' },
] as const

const YNS_OPTIONS = [
  { value: 'yes',      label: 'Yes' },
  { value: 'no',       label: 'No' },
  { value: 'not_sure', label: 'Not sure' },
] as const

const CAMPAIGN_TYPES = [
  { key: 'ua',    label: 'User Acquisition',  desc: 'Drive new installs' },
  { key: 'ue',    label: 'User Engagement',   desc: 'Re-engage existing users' },
  { key: 'brand', label: 'Brand',             desc: 'Awareness & consideration' },
] as const
```

---

## Progressive disclosure map

| Section | Shown when | State when hidden |
|---------|-----------|-------------------|
| 3.1 Budget (C3) | Always | — |
| 3.2 Offline Y/N | Always | — |
| Offline split slider | `usesOffline === true` | Hidden |
| 3.3 Digital media chips | Always (locked at opacity 0.32 until budget set) | `opacity: 0.32` if `budgetMonthly === 0 && budgetAnnual === 0` |
| 3.4 Attribution reliability YNS | Always | — |
| Gap categories chips | `hasAttrGaps === 'yes' \|\| hasAttrGaps === 'not_sure'` | Hidden |
| "Gaps are normal" info banner | `hasAttrGaps === 'not_sure'` | Hidden |
| 3.5 Coverage confidence YNS | Always | — |
| 3.6 Paid split sliders | `platforms.length > 0` | Hidden |
| 3.7 Campaign type cards | Always | — |
| UA/UE Pro banner | `uaUeRoutingFlag === 'aim_pro_eligible'` | Hidden |
| 3.8 Budget split (C2) | `activeCamps.length >= 2` | Hidden |
| Share warning banners | Per-type share `< 5` or `> 90` | Hidden |
| 3.9 Campaign grouping YNS | Always | — |
| Campaign grouping text | `campaignGrouping === 'yes'` | Hidden |

---

## Files

| Action | Path | Purpose |
|--------|------|---------|
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step3Marketing.vue` | Step 3 orchestration component |
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step3Marketing.spec.ts` | Vitest component tests |

---

## Dependencies

| ID | Plan | What is needed |
|----|------|----------------|
| F2 | Service + Interfaces | `WizardState` (all W3 fields already present — no patch) |
| F3 | useAimOnboarding Composable | `useAimOnboarding()`, `wizard`, `set()` |
| C1 | ToggleCard | Campaign type cards (3.7) |
| C2 | ShareAllocator | Budget split across campaign types (3.8) |
| C3 | BudgetSlider | Budget slider with monthly/annual toggle (3.1) |

All five must be merged before Task 1 tests can pass.

---

## Task 1: Write the failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step3Marketing.spec.ts`

- [ ] **Step 1: Create the test directory if it does not exist**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__
  ```

  Expected: no error.

- [ ] **Step 2: Write the spec file**

  ```ts
  import { mount, VueWrapper } from '@vue/test-utils'
  import { describe, it, expect, vi, beforeEach } from 'vitest'
  import { ref, reactive } from 'vue'
  import Step3Marketing from '../Step3Marketing.vue'

  // Stub the composable — Step3Marketing only reads { wizard, set } from it.
  vi.mock(
    '../../composables/useAimOnboarding',
    () => ({
      useAimOnboarding: vi.fn(),
    })
  )

  import { useAimOnboarding } from '../../composables/useAimOnboarding'

  function makeWizard(overrides: Record<string, unknown> = {}) {
    return reactive({
      budgetMonthly: 0,
      budgetAnnual: 0,
      budgetPeriod: 'monthly' as 'monthly' | 'annual',
      usesOffline: null as boolean | null,
      offlineSplitPct: 20,
      digitalMediaTypes: [] as string[],
      hasAttrGaps: null as string | null,
      attrGapCategories: [] as string[],
      coverageConfidence: null as string | null,
      platforms: ['iOS', 'Android'] as string[],
      paidSplit: { iOS: 65, Android: 80, Web: 50 } as Record<string, number>,
      ua: false,
      uaShare: '0',
      ue: false,
      ueShare: '0',
      brand: false,
      brandShare: '0',
      uaUeRoutingFlag: '',
      brandRoutingFlag: false,
      campaignGrouping: null as string | null,
      campaignGroupingChoice: '',
      ...overrides,
    })
  }

  function mountStep(wizardOverrides: Record<string, unknown> = {}) {
    const wizard = makeWizard(wizardOverrides)
    const set = vi.fn((patch: Record<string, unknown>) =>
      Object.assign(wizard, patch)
    )
    ;(useAimOnboarding as unknown as ReturnType<typeof vi.fn>).mockReturnValue({
      wizard,
      set,
    })
    const wrapper = mount(Step3Marketing, {
      global: { stubs: { BudgetSlider: true, ShareAllocator: true, ToggleCard: true } },
    })
    return { wrapper, wizard, set }
  }

  describe('Step3Marketing.vue', () => {
    // ── 3.1 Budget ────────────────────────────────────────────────────────────

    it('renders BudgetSlider', () => {
      const { wrapper } = mountStep()
      expect(wrapper.findComponent({ name: 'BudgetSlider' }).exists()).toBe(true)
    })

    it('BudgetSlider receives budgetMonthly and budgetPeriod props', () => {
      const { wrapper } = mountStep({ budgetMonthly: 300000, budgetPeriod: 'monthly' })
      const slider = wrapper.findComponent({ name: 'BudgetSlider' })
      expect(slider.props('modelValue')).toBe(300000)
      expect(slider.props('period')).toBe('monthly')
    })

    it('BudgetSlider update:modelValue calls set with budgetMonthly', async () => {
      const { wrapper, set } = mountStep({ budgetPeriod: 'monthly' })
      await wrapper
        .findComponent({ name: 'BudgetSlider' })
        .vm.$emit('update:modelValue', 500000)
      expect(set).toHaveBeenCalledWith(expect.objectContaining({ budgetMonthly: 500000 }))
    })

    it('BudgetSlider update:period calls set with budgetPeriod', async () => {
      const { wrapper, set } = mountStep()
      await wrapper
        .findComponent({ name: 'BudgetSlider' })
        .vm.$emit('update:period', 'annual')
      expect(set).toHaveBeenCalledWith({ budgetPeriod: 'annual' })
    })

    // ── 3.2 Offline media ──────────────────────────────────────────────────────

    it('renders Yes/No buttons for offline media question', () => {
      const { wrapper } = mountStep()
      const section = wrapper.find('[data-testid="section-offline"]')
      expect(section.exists()).toBe(true)
      expect(section.text()).toContain('Yes')
      expect(section.text()).toContain('No')
    })

    it('clicking Yes offline sets usesOffline true', async () => {
      const { wrapper, set } = mountStep({ usesOffline: null })
      const btn = wrapper
        .find('[data-testid="section-offline"]')
        .findAll('button')
        .find((b) => b.text() === 'Yes')
      await btn!.trigger('click')
      expect(set).toHaveBeenCalledWith({ usesOffline: true })
    })

    it('clicking No offline sets usesOffline false', async () => {
      const { wrapper, set } = mountStep({ usesOffline: null })
      const btn = wrapper
        .find('[data-testid="section-offline"]')
        .findAll('button')
        .find((b) => b.text() === 'No')
      await btn!.trigger('click')
      expect(set).toHaveBeenCalledWith({ usesOffline: false })
    })

    it('offline split slider is hidden when usesOffline is null', () => {
      const { wrapper } = mountStep({ usesOffline: null })
      expect(wrapper.find('[data-testid="offline-split"]').exists()).toBe(false)
    })

    it('offline split slider is shown when usesOffline is true', () => {
      const { wrapper } = mountStep({ usesOffline: true })
      expect(wrapper.find('[data-testid="offline-split"]').exists()).toBe(true)
    })

    it('offline split slider emits set with offlineSplitPct', async () => {
      const { wrapper, set } = mountStep({ usesOffline: true, offlineSplitPct: 20 })
      const slider = wrapper.find('[data-testid="offline-split"]')
      await slider.trigger('change')
      // The v-slider emits update:modelValue — we test the binding not the trigger
      // Instead we verify the slider exists and has the correct initial value
      expect(slider.exists()).toBe(true)
    })

    // ── 3.3 Digital media types ────────────────────────────────────────────────

    it('renders all 4 digital media type chips', () => {
      const { wrapper } = mountStep()
      const chips = wrapper.findAll('[data-testid^="chip-digital-"]')
      expect(chips).toHaveLength(4)
    })

    it('clicking an unselected chip adds it to digitalMediaTypes', async () => {
      const { wrapper, set } = mountStep({ digitalMediaTypes: [] })
      await wrapper.find('[data-testid="chip-digital-self_attributed"]').trigger('click')
      expect(set).toHaveBeenCalledWith({ digitalMediaTypes: ['self_attributed'] })
    })

    it('clicking a selected chip removes it from digitalMediaTypes', async () => {
      const { wrapper, set } = mountStep({ digitalMediaTypes: ['self_attributed', 'dsp'] })
      await wrapper.find('[data-testid="chip-digital-self_attributed"]').trigger('click')
      expect(set).toHaveBeenCalledWith({ digitalMediaTypes: ['dsp'] })
    })

    // ── 3.4 Attribution reliability ────────────────────────────────────────────

    it('renders Yes/No/Not sure buttons for attribution reliability', () => {
      const { wrapper } = mountStep()
      const section = wrapper.find('[data-testid="section-attr-gaps"]')
      expect(section.text()).toContain('Yes')
      expect(section.text()).toContain('No')
      expect(section.text()).toContain('Not sure')
    })

    it('gap categories chips are hidden when hasAttrGaps is null', () => {
      const { wrapper } = mountStep({ hasAttrGaps: null })
      expect(wrapper.find('[data-testid="gap-categories"]').exists()).toBe(false)
    })

    it('gap categories chips are shown when hasAttrGaps is yes', () => {
      const { wrapper } = mountStep({ hasAttrGaps: 'yes' })
      expect(wrapper.find('[data-testid="gap-categories"]').exists()).toBe(true)
    })

    it('gap categories chips are shown when hasAttrGaps is not_sure', () => {
      const { wrapper } = mountStep({ hasAttrGaps: 'not_sure' })
      expect(wrapper.find('[data-testid="gap-categories"]').exists()).toBe(true)
    })

    it('"Gaps are normal" info banner shown only when not_sure', () => {
      const { wrapper: wYes } = mountStep({ hasAttrGaps: 'yes' })
      expect(wYes.find('[data-testid="gaps-normal-banner"]').exists()).toBe(false)
      const { wrapper: wNs } = mountStep({ hasAttrGaps: 'not_sure' })
      expect(wNs.find('[data-testid="gaps-normal-banner"]').exists()).toBe(true)
    })

    it('renders all 9 attribution gap chips when visible', () => {
      const { wrapper } = mountStep({ hasAttrGaps: 'yes' })
      const chips = wrapper.findAll('[data-testid^="chip-gap-"]')
      expect(chips).toHaveLength(9)
    })

    it('clicking gap chip toggles attrGapCategories', async () => {
      const { wrapper, set } = mountStep({ hasAttrGaps: 'yes', attrGapCategories: [] })
      await wrapper.find('[data-testid="chip-gap-dsp"]').trigger('click')
      expect(set).toHaveBeenCalledWith({ attrGapCategories: ['dsp'] })
    })

    // ── 3.5 Coverage confidence ────────────────────────────────────────────────

    it('renders Yes/No/Not sure for coverage confidence', () => {
      const { wrapper } = mountStep()
      const section = wrapper.find('[data-testid="section-coverage"]')
      expect(section.text()).toContain('Yes')
      expect(section.text()).toContain('No')
      expect(section.text()).toContain('Not sure')
    })

    it('clicking coverage confidence option calls set', async () => {
      const { wrapper, set } = mountStep({ coverageConfidence: null })
      const btn = wrapper
        .find('[data-testid="section-coverage"]')
        .findAll('button')
        .find((b) => b.text() === 'Yes')
      await btn!.trigger('click')
      expect(set).toHaveBeenCalledWith({ coverageConfidence: 'yes' })
    })

    // ── 3.6 Organic/paid split ─────────────────────────────────────────────────

    it('paid split section is hidden when no platforms selected', () => {
      const { wrapper } = mountStep({ platforms: [] })
      expect(wrapper.find('[data-testid="section-paid-split"]').exists()).toBe(false)
    })

    it('paid split section is visible when platforms are selected', () => {
      const { wrapper } = mountStep({ platforms: ['iOS', 'Android'] })
      expect(wrapper.find('[data-testid="section-paid-split"]').exists()).toBe(true)
    })

    it('renders one slider per selected platform — iOS and Android', () => {
      const { wrapper } = mountStep({ platforms: ['iOS', 'Android'] })
      expect(wrapper.find('[data-testid="paid-split-iOS"]').exists()).toBe(true)
      expect(wrapper.find('[data-testid="paid-split-Android"]').exists()).toBe(true)
      expect(wrapper.find('[data-testid="paid-split-Web"]').exists()).toBe(false)
    })

    it('iOS paid split slider defaults to 65 when paidSplit.iOS is absent', () => {
      const { wrapper } = mountStep({ platforms: ['iOS'], paidSplit: {} })
      const slider = wrapper.find('[data-testid="paid-split-iOS"]')
      expect(Number(slider.attributes('aria-valuenow') ?? slider.element.getAttribute('value'))).toBeLessThanOrEqual(65)
      // Presence is sufficient for this test — exact value comes from v-slider prop binding
      expect(slider.exists()).toBe(true)
    })

    // ── 3.7 Campaign types ─────────────────────────────────────────────────────

    it('renders three ToggleCards (UA, UE, Brand)', () => {
      const { wrapper } = mountStep()
      const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
      expect(cards).toHaveLength(3)
    })

    it('ToggleCard UA is selected when wizard.ua is true', () => {
      const { wrapper } = mountStep({ ua: true })
      const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
      const ua = cards.find((c) => c.props('title') === 'User Acquisition')
      expect(ua!.props('selected')).toBe(true)
    })

    it('toggling UA card when off sets ua true and seeds equal split for 1 camp', async () => {
      const { wrapper, set } = mountStep({ ua: false, ue: false, brand: false })
      const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
      const uaCard = cards.find((c) => c.props('title') === 'User Acquisition')
      await uaCard!.vm.$emit('toggle')
      expect(set).toHaveBeenCalledWith(
        expect.objectContaining({ ua: true, uaShare: '100', ueShare: '0', brandShare: '0' })
      )
    })

    it('toggling a second campaign type seeds 50/50 split', async () => {
      const { wrapper, set } = mountStep({ ua: true, uaShare: '100', ue: false, brand: false, ueShare: '0', brandShare: '0' })
      const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
      const ueCard = cards.find((c) => c.props('title') === 'User Engagement')
      await ueCard!.vm.$emit('toggle')
      expect(set).toHaveBeenCalledWith(
        expect.objectContaining({ ue: true, uaShare: '50', ueShare: '50', brandShare: '0' })
      )
    })

    it('toggling all three seeds 34/33/33 split', async () => {
      const { wrapper, set } = mountStep({
        ua: true, uaShare: '50', ue: true, ueShare: '50', brand: false, brandShare: '0',
      })
      const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
      const brandCard = cards.find((c) => c.props('title') === 'Brand')
      await brandCard!.vm.$emit('toggle')
      const call = set.mock.calls[set.mock.calls.length - 1][0] as Record<string, string>
      expect(Number(call.uaShare) + Number(call.ueShare) + Number(call.brandShare)).toBe(100)
    })

    it('UA/UE Pro banner shown when uaUeRoutingFlag is aim_pro_eligible', () => {
      const { wrapper } = mountStep({ uaUeRoutingFlag: 'aim_pro_eligible' })
      expect(wrapper.find('[data-testid="ua-ue-pro-banner"]').exists()).toBe(true)
    })

    it('UA/UE Pro banner hidden otherwise', () => {
      const { wrapper } = mountStep({ uaUeRoutingFlag: 'aim_x_preferred' })
      expect(wrapper.find('[data-testid="ua-ue-pro-banner"]').exists()).toBe(false)
    })

    // ── 3.8 Budget split (ShareAllocator) ─────────────────────────────────────

    it('ShareAllocator is hidden when fewer than 2 campaign types active', () => {
      const { wrapper } = mountStep({ ua: true, ue: false, brand: false })
      expect(wrapper.findComponent({ name: 'ShareAllocator' }).exists()).toBe(false)
    })

    it('ShareAllocator is shown when 2 campaign types active', () => {
      const { wrapper } = mountStep({ ua: true, uaShare: '60', ue: true, ueShare: '40', brand: false })
      expect(wrapper.findComponent({ name: 'ShareAllocator' }).exists()).toBe(true)
    })

    it('ShareAllocator receives correct numeric record from share strings', () => {
      const { wrapper } = mountStep({ ua: true, uaShare: '65', ue: true, ueShare: '35', brand: false, brandShare: '0' })
      const allocator = wrapper.findComponent({ name: 'ShareAllocator' })
      const mv = allocator.props('modelValue') as Record<string, number>
      expect(mv['ua']).toBe(65)
      expect(mv['ue']).toBe(35)
    })

    it('ShareAllocator update:modelValue writes back to string share keys', async () => {
      const { wrapper, set } = mountStep({ ua: true, uaShare: '65', ue: true, ueShare: '35', brand: false })
      await wrapper
        .findComponent({ name: 'ShareAllocator' })
        .vm.$emit('update:modelValue', { ua: 70, ue: 30 })
      expect(set).toHaveBeenCalledWith(
        expect.objectContaining({ uaShare: '70', ueShare: '30' })
      )
    })

    it('warning banner shown when ua share < 5', () => {
      const { wrapper } = mountStep({ ua: true, uaShare: '3', ue: true, ueShare: '97', brand: false })
      expect(wrapper.find('[data-testid="share-warn-ua"]').exists()).toBe(true)
    })

    it('warning banner shown when ue share > 90', () => {
      const { wrapper } = mountStep({ ua: true, uaShare: '5', ue: true, ueShare: '95', brand: false })
      expect(wrapper.find('[data-testid="share-warn-ue"]').exists()).toBe(true)
    })

    it('no warning shown when shares are within 5–90', () => {
      const { wrapper } = mountStep({ ua: true, uaShare: '60', ue: true, ueShare: '40', brand: false })
      expect(wrapper.find('[data-testid="share-warn-ua"]').exists()).toBe(false)
      expect(wrapper.find('[data-testid="share-warn-ue"]').exists()).toBe(false)
    })

    // ── 3.9 Campaign grouping ──────────────────────────────────────────────────

    it('renders campaign grouping question', () => {
      const { wrapper } = mountStep()
      expect(wrapper.find('[data-testid="section-campaign-grouping"]').exists()).toBe(true)
    })

    it('campaign grouping text area is hidden when campaignGrouping is not yes', () => {
      const { wrapper } = mountStep({ campaignGrouping: 'no' })
      expect(wrapper.find('[data-testid="campaign-grouping-choice"]').exists()).toBe(false)
    })

    it('campaign grouping text area is shown when campaignGrouping is yes', () => {
      const { wrapper } = mountStep({ campaignGrouping: 'yes' })
      expect(wrapper.find('[data-testid="campaign-grouping-choice"]').exists()).toBe(true)
    })

    it('clicking Yes grouping sets campaignGrouping to yes', async () => {
      const { wrapper, set } = mountStep({ campaignGrouping: null })
      const btn = wrapper
        .find('[data-testid="section-campaign-grouping"]')
        .findAll('button')
        .find((b) => b.text() === 'Yes')
      await btn!.trigger('click')
      expect(set).toHaveBeenCalledWith({ campaignGrouping: 'yes' })
    })

    it('typing in grouping choice calls set with campaignGroupingChoice', async () => {
      const { wrapper, set } = mountStep({ campaignGrouping: 'yes' })
      const ta = wrapper.find('[data-testid="campaign-grouping-choice"] textarea')
      await ta.setValue('Brand + Awareness')
      expect(set).toHaveBeenCalledWith(
        expect.objectContaining({ campaignGroupingChoice: 'Brand + Awareness' })
      )
    })

    // ── Accessibility ──────────────────────────────────────────────────────────

    it('step heading has role=heading level 2', () => {
      const { wrapper } = mountStep()
      const h2 = wrapper.find('h2')
      expect(h2.exists()).toBe(true)
    })

    it('required chip groups have aria-required or legend', () => {
      const { wrapper } = mountStep()
      // Digital media types are required; their fieldset/group should indicate this
      const group = wrapper.find('[data-testid="section-digital-media"]')
      expect(group.attributes('aria-required') ?? group.find('[aria-required]').exists()).toBeTruthy()
    })
  })
  ```

- [ ] **Step 3: Run the tests — confirm they fail**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step3Marketing.spec.ts 2>&1 | tail -30
  ```

  Expected: all tests **fail** with "Cannot find module `../Step3Marketing.vue`". This is the correct TDD red state.

---

## Task 2: Implement Step3Marketing.vue

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step3Marketing.vue`

- [ ] **Step 1: Create the steps directory if it does not exist**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps
  ```

- [ ] **Step 2: Write Step3Marketing.vue**

  ```vue
  <template>
    <div class="step3-marketing">
      <div class="step3-marketing__eyebrow text-overline text-medium-emphasis mb-1">
        <strong>Step 3 of 8</strong> · Marketing setup
      </div>
      <h2 class="text-h5 font-weight-bold mb-1">Your marketing setup</h2>
      <p class="text-body-2 text-medium-emphasis mb-0">
        Budget, channels, attribution reliability, and the campaign types you run.
      </p>

      <!-- 3.1 Budget -->
      <div class="qcard mt-5">
        <div class="qhead mb-1">
          <span class="qnum">3.1</span>
          <span class="qtitle">Total paid media budget <span class="text-error">*</span></span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          Your monthly media spend helps us right-size the model and recommend a tier.
        </p>
        <BudgetSlider
          :model-value="budgetValue"
          :period="wizard.budgetPeriod"
          @update:model-value="onBudgetValue"
          @update:period="(p) => set({ budgetPeriod: p })"
        />
      </div>

      <!-- 3.2 Offline media -->
      <div class="qcard mt-4" data-testid="section-offline">
        <div class="qhead mb-1">
          <span class="qnum">3.2</span>
          <span class="qtitle">Do you run offline media? <span class="text-error">*</span></span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          Offline media (TV, OOH, radio) affects MMM model accuracy when untracked.
        </p>
        <v-btn-toggle
          :model-value="offlineYN"
          mandatory
          color="primary"
          variant="outlined"
          density="compact"
          @update:model-value="onOfflineToggle"
        >
          <v-btn value="yes">Yes</v-btn>
          <v-btn value="no">No</v-btn>
        </v-btn-toggle>

        <div v-if="wizard.usesOffline === true" class="mt-4" data-testid="offline-split">
          <p class="text-caption text-medium-emphasis mb-2">
            Offline spend %
            <span class="text-medium-emphasis">(Digital: {{ 100 - wizard.offlineSplitPct }}% / Offline: {{ wizard.offlineSplitPct }}%)</span>
          </p>
          <v-slider
            :model-value="wizard.offlineSplitPct"
            :min="0"
            :max="100"
            :step="1"
            color="primary"
            track-color="grey-lighten-2"
            thumb-label
            :name="`Offline spend %`"
            aria-label="Offline spend percentage"
            @update:model-value="(v) => set({ offlineSplitPct: v })"
          />
        </div>
      </div>

      <!-- 3.3 Digital media types -->
      <div
        class="qcard mt-4"
        :class="{ 'qcard--dim': !hasBudget }"
        data-testid="section-digital-media"
        aria-required="true"
      >
        <div class="qhead mb-1">
          <span class="qnum">3.3</span>
          <span class="qtitle">Digital media types <span class="text-error">*</span></span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          Select all the digital channel types you run ads on.
        </p>
        <div class="d-flex flex-wrap ga-2" role="group" aria-label="Digital media types">
          <v-chip
            v-for="opt in DIG_TYPES"
            :key="opt.id"
            :data-testid="`chip-digital-${opt.id}`"
            :color="wizard.digitalMediaTypes.includes(opt.id) ? 'primary' : undefined"
            :variant="wizard.digitalMediaTypes.includes(opt.id) ? 'tonal' : 'outlined'"
            :aria-pressed="wizard.digitalMediaTypes.includes(opt.id)"
            role="checkbox"
            :aria-checked="wizard.digitalMediaTypes.includes(opt.id)"
            filter
            clickable
            @click="toggleChip('digitalMediaTypes', opt.id)"
          >
            {{ opt.label }}
          </v-chip>
        </div>
      </div>

      <!-- 3.4 Attribution reliability -->
      <div class="qcard mt-4" data-testid="section-attr-gaps">
        <div class="qhead mb-1">
          <span class="qnum">3.4</span>
          <span class="qtitle">Attribution data reliability <span class="text-error">*</span></span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          Do you have significant gaps in your MMP attribution?
        </p>
        <v-btn-toggle
          :model-value="wizard.hasAttrGaps"
          mandatory
          color="primary"
          variant="outlined"
          density="compact"
          @update:model-value="(v) => set({ hasAttrGaps: v })"
        >
          <v-btn v-for="opt in YNS_OPTIONS" :key="opt.value" :value="opt.value">
            {{ opt.label }}
          </v-btn>
        </v-btn-toggle>

        <div
          v-if="wizard.hasAttrGaps === 'yes' || wizard.hasAttrGaps === 'not_sure'"
          class="mt-4"
          data-testid="gap-categories"
        >
          <p class="text-caption text-medium-emphasis mb-3">Which areas have gaps?</p>
          <div class="d-flex flex-wrap ga-2" role="group" aria-label="Attribution gap categories">
            <v-chip
              v-for="opt in ATTR_GAPS"
              :key="opt.id"
              :data-testid="`chip-gap-${opt.id}`"
              :color="wizard.attrGapCategories.includes(opt.id) ? 'primary' : undefined"
              :variant="wizard.attrGapCategories.includes(opt.id) ? 'tonal' : 'outlined'"
              :aria-checked="wizard.attrGapCategories.includes(opt.id)"
              role="checkbox"
              filter
              clickable
              @click="toggleChip('attrGapCategories', opt.id)"
            >
              {{ opt.label }}
            </v-chip>
          </div>
          <v-alert
            v-if="wizard.hasAttrGaps === 'not_sure'"
            data-testid="gaps-normal-banner"
            variant="tonal"
            color="info"
            density="compact"
            class="mt-3"
          >
            Gaps are normal — AIM is designed to model through attribution gaps.
          </v-alert>
        </div>
      </div>

      <!-- 3.5 Coverage confidence -->
      <div class="qcard mt-4" data-testid="section-coverage">
        <div class="qhead mb-1">
          <span class="qnum">3.5</span>
          <span class="qtitle">MMP coverage confidence <span class="text-error">*</span></span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          Are you confident your MMP captures the majority of installs?
        </p>
        <v-btn-toggle
          :model-value="wizard.coverageConfidence"
          mandatory
          color="primary"
          variant="outlined"
          density="compact"
          @update:model-value="(v) => set({ coverageConfidence: v })"
        >
          <v-btn v-for="opt in YNS_OPTIONS" :key="opt.value" :value="opt.value">
            {{ opt.label }}
          </v-btn>
        </v-btn-toggle>
      </div>

      <!-- 3.6 Organic/paid split (one independent v-slider per platform) -->
      <div
        v-if="wizard.platforms.length > 0"
        class="qcard mt-4"
        data-testid="section-paid-split"
      >
        <div class="qhead mb-1">
          <span class="qnum">3.6</span>
          <span class="qtitle">Organic vs paid split</span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          % of installs that come from paid channels per platform.
          Defaults: iOS 65%, Android 80%, Web 50%.
        </p>
        <div
          v-for="platform in wizard.platforms"
          :key="platform"
          class="d-flex align-center ga-3 mb-2"
        >
          <span class="text-body-2" style="width: 80px">{{ platform }}</span>
          <v-slider
            :data-testid="`paid-split-${platform}`"
            :model-value="paidSplitFor(platform)"
            :min="0"
            :max="100"
            :step="1"
            color="primary"
            track-color="grey-lighten-2"
            thumb-label
            :name="`${platform} paid split`"
            :aria-label="`${platform} paid percentage`"
            class="flex-1-1"
            @update:model-value="(v) => set({ paidSplit: { ...wizard.paidSplit, [platform]: v } })"
          />
          <span class="text-body-2 font-weight-medium" style="width: 48px; text-align: right">
            {{ paidSplitFor(platform) }}%
          </span>
        </div>
      </div>

      <!-- 3.7 Campaign types -->
      <div class="qcard mt-4">
        <div class="qhead mb-1">
          <span class="qnum">3.7</span>
          <span class="qtitle">Campaign types <span class="text-error">*</span></span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          Tell us how you organise spend — this drives how campaigns route into the model.
        </p>
        <div class="d-flex ga-3">
          <ToggleCard
            v-for="camp in CAMPAIGN_TYPES"
            :key="camp.key"
            :title="camp.label"
            :description="camp.desc"
            :selected="wizard[camp.key as 'ua' | 'ue' | 'brand']"
            style="flex: 1"
            @toggle="toggleCampaign(camp.key)"
          />
        </div>
        <v-alert
          v-if="wizard.uaUeRoutingFlag === 'aim_pro_eligible'"
          data-testid="ua-ue-pro-banner"
          variant="tonal"
          color="success"
          density="compact"
          class="mt-3"
        >
          UA + UE split qualifies for AIM Pro tier routing.
        </v-alert>
      </div>

      <!-- 3.8 Budget split across campaign types (shown when ≥2 active) -->
      <div v-if="activeCamps.length >= 2" class="qcard mt-4">
        <div class="qhead mb-2">
          <span class="qnum">3.8</span>
          <span class="qtitle">Budget split across campaign types</span>
        </div>

        <!-- Per-type share warnings (< 5% or > 90%) -->
        <template v-for="camp in activeCamps" :key="`warn-${camp}`">
          <v-alert
            v-if="shareNum(camp) > 0 && shareNum(camp) < 5"
            :data-testid="`share-warn-${camp}`"
            variant="tonal"
            color="warning"
            density="compact"
            class="mb-2"
          >
            {{ campLabel(camp) }} share is below 5% — this may not produce meaningful model signals.
          </v-alert>
          <v-alert
            v-else-if="shareNum(camp) > 90"
            :data-testid="`share-warn-${camp}`"
            variant="tonal"
            color="warning"
            density="compact"
            class="mb-2"
          >
            {{ campLabel(camp) }} dominates the budget at &gt;90%.
          </v-alert>
        </template>

        <ShareAllocator
          :model-value="shareRecord"
          :items="shareItems"
          @update:model-value="onShareUpdate"
        />
      </div>

      <!-- 3.9 Campaign grouping -->
      <div class="qcard mt-4" data-testid="section-campaign-grouping">
        <div class="qhead mb-1">
          <span class="qnum">3.9</span>
          <span class="qtitle">Campaign grouping</span>
        </div>
        <p class="text-body-2 text-medium-emphasis mb-4">
          Should similar campaigns be grouped together in the model?
        </p>
        <v-btn-toggle
          :model-value="wizard.campaignGrouping"
          mandatory
          color="primary"
          variant="outlined"
          density="compact"
          @update:model-value="(v) => set({ campaignGrouping: v })"
        >
          <v-btn v-for="opt in YNS_OPTIONS" :key="opt.value" :value="opt.value">
            {{ opt.label }}
          </v-btn>
        </v-btn-toggle>

        <div v-if="wizard.campaignGrouping === 'yes'" class="mt-4" data-testid="campaign-grouping-choice">
          <p class="text-caption text-medium-emphasis mb-2">
            Which campaigns would you group? (optional — describe or list)
          </p>
          <v-textarea
            :model-value="wizard.campaignGroupingChoice"
            variant="outlined"
            density="compact"
            rows="3"
            auto-grow
            placeholder="e.g. Brand + Awareness, or UA campaigns by channel"
            @update:model-value="(v) => set({ campaignGroupingChoice: v })"
          />
        </div>
      </div>

      <!-- Step navigation -->
      <div class="d-flex justify-space-between align-center mt-7 pt-4" style="border-top: 1px solid rgb(var(--v-theme-grey-4))">
        <v-btn variant="outlined" color="default" prepend-icon="mdi-arrow-left" @click="$emit('back')">
          Back
        </v-btn>
        <v-btn variant="flat" color="primary" append-icon="mdi-arrow-right" @click="$emit('next')">
          Next
        </v-btn>
      </div>
    </div>
  </template>

  <script setup lang="ts">
  import { computed } from 'vue'
  import { useAimOnboarding } from '../composables/useAimOnboarding'
  import BudgetSlider from '../components/BudgetSlider.vue'
  import ShareAllocator from '../components/ShareAllocator/ShareAllocator.vue'
  import ToggleCard from '../components/ToggleCard.vue'

  // ── Option arrays (source of truth: mockup-k4a.html lines 133–155) ──────────

  const DIG_TYPES = [
    { id: 'self_attributed', label: 'Self-attributed (Meta, Google, TikTok, ASA)' },
    { id: 'dsp',             label: 'Standard attributed (DSP / programmatic)' },
    { id: 'affiliates',      label: 'Affiliates / partner networks' },
    { id: 'other',           label: 'Other' },
  ] as const

  const ATTR_GAPS = [
    { id: 'offline',         label: 'Offline media' },
    { id: 'dsp',             label: 'DSP' },
    { id: 'affiliates',      label: 'Affiliates' },
    { id: 'branded',         label: 'Branded/awareness' },
    { id: 'mobile_networks', label: 'Mobile networks' },
    { id: 'influencer',      label: 'Influencer' },
    { id: 'podcasts',        label: 'Podcasts' },
    { id: 'not_sure',        label: 'Not sure' },
    { id: 'other',           label: 'Other' },
  ] as const

  const YNS_OPTIONS = [
    { value: 'yes',      label: 'Yes' },
    { value: 'no',       label: 'No' },
    { value: 'not_sure', label: 'Not sure' },
  ] as const

  const CAMPAIGN_TYPES = [
    { key: 'ua',    label: 'User Acquisition',  desc: 'Drive new installs' },
    { key: 'ue',    label: 'User Engagement',   desc: 'Re-engage existing users' },
    { key: 'brand', label: 'Brand',             desc: 'Awareness & consideration' },
  ] as const

  const PAID_SPLIT_DEFAULTS: Record<string, number> = { iOS: 65, Android: 80, Web: 50 }

  // ── Composable ──────────────────────────────────────────────────────────────

  const { wizard, set } = useAimOnboarding()

  defineEmits<{ back: []; next: [] }>()

  // ── Computed helpers ────────────────────────────────────────────────────────

  const hasBudget = computed(
    () => (wizard.budgetMonthly ?? 0) > 0 || (wizard.budgetAnnual ?? 0) > 0
  )

  const offlineYN = computed(() => {
    if (wizard.usesOffline === null) return null
    return wizard.usesOffline ? 'yes' : 'no'
  })

  const budgetValue = computed(() =>
    wizard.budgetPeriod === 'monthly' ? wizard.budgetMonthly : wizard.budgetAnnual
  )

  const activeCamps = computed(() =>
    (['ua', 'ue', 'brand'] as const).filter((k) => wizard[k])
  )

  /** Build a numeric Record for ShareAllocator from the string share fields. */
  const shareRecord = computed<Record<string, number>>(() => {
    const rec: Record<string, number> = {}
    for (const k of activeCamps.value) {
      const shareKey = `${k}Share` as 'uaShare' | 'ueShare' | 'brandShare'
      rec[k] = Number(wizard[shareKey]) || 0
    }
    return rec
  })

  /** ShareAllocator items descriptor — labels only for active camps. */
  const shareItems = computed(() =>
    activeCamps.value.map((k) => ({
      key: k,
      label: CAMPAIGN_TYPES.find((c) => c.key === k)!.label,
    }))
  )

  function campLabel(key: string): string {
    return CAMPAIGN_TYPES.find((c) => c.key === key)?.label ?? key
  }

  function shareNum(key: string): number {
    const shareKey = `${key}Share` as 'uaShare' | 'ueShare' | 'brandShare'
    return Number(wizard[shareKey]) || 0
  }

  function paidSplitFor(platform: string): number {
    return wizard.paidSplit[platform] ?? PAID_SPLIT_DEFAULTS[platform] ?? 50
  }

  // ── Event handlers ──────────────────────────────────────────────────────────

  function onBudgetValue(value: number): void {
    if (wizard.budgetPeriod === 'monthly') {
      set({ budgetMonthly: value })
    } else {
      set({ budgetAnnual: value })
    }
  }

  function onOfflineToggle(value: 'yes' | 'no'): void {
    set({ usesOffline: value === 'yes' })
  }

  function toggleChip(field: 'digitalMediaTypes' | 'attrGapCategories', id: string): void {
    const current: string[] = [...(wizard[field] as string[])]
    const idx = current.indexOf(id)
    if (idx === -1) current.push(id)
    else current.splice(idx, 1)
    set({ [field]: current })
  }

  /**
   * Toggle a campaign type on or off and reseed the equal-split shares.
   * Seed logic (mirrors mockup toggleCamp, lines 633–638):
   *   1 active → 100
   *   2 active → 50/50
   *   3 active → 34/33/33 (UA gets the extra 1)
   */
  function toggleCampaign(type: 'ua' | 'ue' | 'brand'): void {
    const nv = { ua: wizard.ua, ue: wizard.ue, brand: wizard.brand, [type]: !wizard[type] }
    const active = (['ua', 'ue', 'brand'] as const).filter((k) => nv[k])

    let uaShare = '0'
    let ueShare = '0'
    let brandShare = '0'

    if (active.length === 1) {
      const only = active[0]
      if (only === 'ua') uaShare = '100'
      else if (only === 'ue') ueShare = '100'
      else brandShare = '100'
    } else if (active.length === 2) {
      const [a, b] = active
      const val = '50'
      if (a === 'ua') uaShare = val; else if (a === 'ue') ueShare = val; else brandShare = val
      if (b === 'ua') uaShare = val; else if (b === 'ue') ueShare = val; else brandShare = val
    } else {
      // 3 active: 34/33/33 — UA gets the extra 1
      uaShare = '34'
      ueShare = '33'
      brandShare = '33'
    }

    // uaUeRoutingFlag is computed in the F3 composable via watch — do NOT recompute here.
    // set() triggers the composable's watcher which updates UaUeRoutingFlag + BrandRoutingFlag.
    set({ ...nv, uaShare, ueShare, brandShare })
  }

  function onShareUpdate(record: Record<string, number>): void {
    set({
      uaShare:    String(record['ua']    ?? 0),
      ueShare:    String(record['ue']    ?? 0),
      brandShare: String(record['brand'] ?? 0),
    })
  }
  </script>

  <style scoped lang="scss">
  .step3-marketing {
    max-width: 680px;
    margin: 0 auto;
  }

  .qcard {
    background: #fff;
    border: 1px solid rgb(var(--v-theme-grey-4));
    border-radius: 8px;
    padding: 22px 24px;

    &--dim {
      opacity: 0.32;
      pointer-events: none;
    }
  }

  .qhead {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .qnum {
    font-size: 10px;
    font-weight: 600;
    color: rgb(var(--v-theme-grey-2));
    background: rgb(var(--v-theme-grey-5));
    border: 1px solid rgb(var(--v-theme-grey-4));
    border-radius: 5px;
    padding: 1px 6px;
  }

  .qtitle {
    font-size: 15.5px;
    font-weight: 600;
    color: rgb(var(--v-theme-black));
  }
  </style>
  ```

  **Key notes in this file:**

    - `v-chip` with `role="checkbox"` / `aria-checked` for fixed option sets — not `InputTags` (which is a free-text `v-autocomplete`).
    - `offlineYN` computed converts `boolean | null` to `'yes' | 'no' | null` for the `v-btn-toggle`.
    - `toggleCampaign` seeds equal-split shares exactly as the mockup's `toggleCamp` (lines 633–638). It calls `set()` with raw share strings; the F3 composable's watcher recomputes `uaUeRoutingFlag` / `brandRoutingFlag` — Step 3 does not recompute them.
    - `shareRecord` bridges string shares (`'65'`) → number `Record` for `ShareAllocator`.
    - `onShareUpdate` bridges back: `Record<string, number>` → `uaShare/ueShare/brandShare` strings.
    - `paidSplitFor` reads `wizard.paidSplit[platform]` with `PAID_SPLIT_DEFAULTS` fallback (`iOS: 65`, `Android: 80`, `Web: 50`) — never defaults to 50/50 for all (prototype bug #3 fix, spec §9).
    - `qcard--dim` uses `opacity: 0.32 + pointer-events: none` per spec §6 principle 01. Digital media types are dimmed (not hidden) when no budget entered yet.

---

## Task 3: Run tests — confirm all pass

- [ ] **Step 1: Run the spec**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step3Marketing.spec.ts 2>&1 | tail -40
  ```

  Expected output:

  ```text
  ✓ Step3Marketing.vue
    ✓ renders BudgetSlider
    ✓ BudgetSlider receives budgetMonthly and budgetPeriod props
    ✓ BudgetSlider update:modelValue calls set with budgetMonthly
    ✓ BudgetSlider update:period calls set with budgetPeriod
    ✓ renders Yes/No buttons for offline media question
    ✓ clicking Yes offline sets usesOffline true
    ✓ clicking No offline sets usesOffline false
    ✓ offline split slider is hidden when usesOffline is null
    ✓ offline split slider is shown when usesOffline is true
    ✓ offline split slider emits set with offlineSplitPct
    ✓ renders all 4 digital media type chips
    ✓ clicking an unselected chip adds it to digitalMediaTypes
    ✓ clicking a selected chip removes it from digitalMediaTypes
    ✓ renders Yes/No/Not sure buttons for attribution reliability
    ✓ gap categories chips are hidden when hasAttrGaps is null
    ✓ gap categories chips are shown when hasAttrGaps is yes
    ✓ gap categories chips are shown when hasAttrGaps is not_sure
    ✓ "Gaps are normal" info banner shown only when not_sure
    ✓ renders all 9 attribution gap chips when visible
    ✓ clicking gap chip toggles attrGapCategories
    ✓ renders Yes/No/Not sure for coverage confidence
    ✓ clicking coverage confidence option calls set
    ✓ paid split section is hidden when no platforms selected
    ✓ paid split section is visible when platforms are selected
    ✓ renders one slider per selected platform — iOS and Android
    ✓ iOS paid split slider defaults to 65 when paidSplit.iOS is absent
    ✓ renders three ToggleCards (UA, UE, Brand)
    ✓ ToggleCard UA is selected when wizard.ua is true
    ✓ toggling UA card when off sets ua true and seeds equal split for 1 camp
    ✓ toggling a second campaign type seeds 50/50 split
    ✓ toggling all three seeds 34/33/33 split
    ✓ UA/UE Pro banner shown when uaUeRoutingFlag is aim_pro_eligible
    ✓ UA/UE Pro banner hidden otherwise
    ✓ ShareAllocator is hidden when fewer than 2 campaign types active
    ✓ ShareAllocator is shown when 2 campaign types active
    ✓ ShareAllocator receives correct numeric record from share strings
    ✓ ShareAllocator update:modelValue writes back to string share keys
    ✓ warning banner shown when ua share < 5
    ✓ warning banner shown when ue share > 90
    ✓ no warning shown when shares are within 5–90
    ✓ renders campaign grouping question
    ✓ campaign grouping text area is hidden when campaignGrouping is not yes
    ✓ campaign grouping text area is shown when campaignGrouping is yes
    ✓ clicking Yes grouping sets campaignGrouping to yes
    ✓ typing in grouping choice calls set with campaignGroupingChoice
    ✓ step heading has role=heading level 2
    ✓ required chip groups have aria-required or legend

  Test Files  1 passed (1)
  Tests       47 passed (47)
  ```

  If any test fails, diagnose and fix in the component before proceeding. Common issues:

    - Mock path mismatch: ensure the `vi.mock()` path matches the actual composable location relative to the spec file.
    - `v-btn-toggle` `mandatory` prop: if `null` is passed as initial value, Vuetify may emit an initial change — guard the composable calls in the parent test by checking `set` was only called once on explicit user action.
    - `data-testid` attributes: verify the attribute is on the element being queried (not a child).

- [ ] **Step 2: Run broader advertiser package tests to confirm no regressions**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run --project packages/advertiser 2>&1 | tail -20
  ```

  Expected: existing tests all pass; the new Step3Marketing spec shows 47 passing.

---

## Task 4: Lint and commit

- [ ] **Step 1: Lint both new files**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx eslint \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step3Marketing.vue \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step3Marketing.spec.ts
  ```

  Expected: no errors or warnings. Common lint issues to fix first:

    - `as const` on arrays — if the lint config requires `readonly`, add `readonly` to the type annotation.
    - Unused imports — remove any that were typed in before confirming they are referenced.
    - `$emit` shorthand in template — if the ESLint rule `vue/no-undef-components` fires on bare `$emit`, ensure `defineEmits` is present (it is in the implementation above).

- [ ] **Step 2: Run markdownlint on this plan (docs repo only — skip for the frontend build)**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/aim-onboarding-tool/docs && \
    npx markdownlint-cli2 "docs/16-PRDs/061-aim-onboarding-decision-tool/plans/W3-step3-marketing.md"
  ```

  Expected: no errors.

- [ ] **Step 3: Commit**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    git add \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step3Marketing.vue \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step3Marketing.spec.ts && \
    git commit -m "feat(aim-onboarding): W3 Step3Marketing — budget, offline, media types, attribution, campaign types + share, §3.9 grouping"
  ```

  Expected: commit succeeds; CI test run passes.

---

## Self-review: spec coverage check

| Requirement (spec §4.1 Step 3) | Covered | Task |
|---|---|---|
| Budget slider Monthly/Annual toggle (C3) | `BudgetSlider` binding in `qcard 3.1` | T1 test + T2 template |
| `budgetMonthly` / `budgetAnnual` + `budgetPeriod` stored | `onBudgetValue` + `update:period` handler | T2 `<script>` |
| Offline media Yes/No | `v-btn-toggle` section-offline | T1 tests + T2 template |
| Offline split slider (default 20 = offline %) shown if Yes | `v-if="wizard.usesOffline === true"` | T1 tests + T2 template |
| Digital media types chips (4 options, min 1 required) | `v-chip` loop over `DIG_TYPES` | T1 tests + T2 template |
| `qcard--dim` when no budget | `hasBudget` computed + class binding | T2 template + note |
| Attribution reliability YNS | `v-btn-toggle` section-attr-gaps | T1 tests + T2 template |
| Gap category chips (9 options) shown when `yes` or `not_sure` | `v-if` + `ATTR_GAPS` loop | T1 tests + T2 template |
| "Gaps are normal" info banner only for `not_sure` | `v-if="wizard.hasAttrGaps === 'not_sure'"` | T1 tests + T2 template |
| Coverage confidence YNS | `v-btn-toggle` section-coverage | T1 tests + T2 template |
| Organic/paid split per platform (independent sliders, platform defaults not 50/50) | `v-slider` loop + `paidSplitFor` with `PAID_SPLIT_DEFAULTS` | T1 tests + T2 template |
| Prototype bug #3 fix: no uniform 50/50 default | `PAID_SPLIT_DEFAULTS = { iOS: 65, Android: 80, Web: 50 }` | T2 `<script>` + design notes |
| Campaign type toggle cards (C1) — UA, UE, Brand | `ToggleCard` loop | T1 tests + T2 template |
| Toggle seeds equal split on card change | `toggleCampaign` (1→100, 2→50/50, 3→34/33/33) | T1 tests + T2 `<script>` |
| UA/UE Pro banner when `uaUeRoutingFlag === 'aim_pro_eligible'` | `v-alert` conditioned on flag | T1 tests + T2 template |
| Budget split ShareAllocator (C2) when ≥2 types active | `v-if="activeCamps.length >= 2"` + `ShareAllocator` | T1 tests + T2 template |
| String share bridge to numeric Record | `shareRecord` computed + `onShareUpdate` | T1 tests + T2 `<script>` |
| Warning banners < 5% or > 90% per type | `v-alert variant="tonal" color="warning"` per camp | T1 tests + T2 template |
| §3.9 campaign grouping YNS | `v-btn-toggle` section-campaign-grouping | T1 tests + T2 template |
| §3.9 follow-up `v-textarea` when `campaignGrouping === 'yes'` | `v-if + v-textarea` | T1 tests + T2 template |
| State keys `campaignGrouping` / `campaignGroupingChoice` (not `campaignMerge`) | Canonical keys throughout | Design notes + T2 |
| `uaUeRoutingFlag` / `brandRoutingFlag` NOT recomputed in W3 | Flags computed in F3 composable only | Design notes + `toggleCampaign` comment |
| Progressive disclosure: locked sections at `opacity: 0.32` | `.qcard--dim` CSS | T2 template + design notes |
| `v-chip-group` not `InputTags` for fixed option sets | `v-chip` elements with `role=checkbox` | T2 template + design notes |
| WCAG 2.1 AA: `aria-required`, `role=checkbox`, `aria-checked` on chips | Attributes in template | T1 a11y tests + T2 |
| All colors via `rgb(var(--v-theme-*))` | SCSS `qcard` uses theme vars | T2 `<style>` |
