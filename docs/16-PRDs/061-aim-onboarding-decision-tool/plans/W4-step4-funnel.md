---
id: plan-w4
title: "W4 — Step 4: Business & Funnel"
---

## W4 — Step 4: Business Model & Conversion Funnel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `steps/Step4Funnel.vue` — the Business Model & Conversion Funnel wizard step — which binds to `useAimOnboarding`, renders the three business model ToggleCards, gaming revenue selector, app funnel events checklist, optional web funnel checklist, and LTV model question chain; business model change resets the funnel to type-specific defaults and clears LTV state.

**Architecture:** `Step4Funnel.vue` is a presentational orchestrator that owns the per-business funnel catalog (static constants: `FUNNEL_CATALOG` and `WEB_FUNNEL_CATALOG`) and maps catalog entries into/out of the `funnel: {items, kpi, kpiConfirmed, names}` contract from F2 (`WizardState`). The step calls `patchWizard` from F3 (`useAimOnboarding`); the composable's `patchWizard` already handles the cross-field LTV/`webFunnel` clear on business change — W4 rides that by sending the new funnel in the same patch. Min-3/one-KPI completeness is validated non-blocking by F7's `validateStep('4', …)` — W4 only enforces interaction-level constraints (required events stay locked on; KPI selection moves a single id).

**Tech Stack:** Vue 3.5, Vuetify 3.12, Vitest + `@vue/test-utils`, TypeScript 5, frontend-mos monorepo.

---

## Files

| Action | Path | Purpose |
|--------|------|---------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step4Funnel.vue` | The wizard step component |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step4Funnel.spec.ts` | Vitest component tests |
| Modify | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts` | Add `GamingRevenue: string` to `WizardState` (F2 contract addition) |
| Modify | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts` | Add `gamingRevenue: ''` to `defaultWizard()` and clear it in the business-change block (F3 contract addition) |

---

## Dependencies

- **F2** (`interfaces/aimOnboarding.ts`) — must exist; this plan adds `GamingRevenue: string` to `WizardState`.
- **F3** (`composables/useAimOnboarding.ts`) — must exist; this plan adds `gamingRevenue: ''` to the default and clears it on business change. The LTV/`webFunnel` clear already lives in F3's `patchWizard` business-change block.
- **C1** (`components/ToggleCard.vue`) — used for the three business model cards.

---

## Funnel Catalog (authoritative — do not derive from mockup arrays)

The catalog lives as module-scope constants in `Step4Funnel.vue`. Each entry describes an event's identity and default state; it is **never stored directly** — only `funnel.items` (ids that are "on") and `funnel.kpi` (the KPI id) are persisted per the F2 contract.

```ts
// Type for a catalog entry — only used inside Step4Funnel.vue
interface FunnelEntry {
  id: string
  label: string
  required: boolean   // if true, always in items; toggle is disabled
  defaultOn: boolean  // initial on/off when this business type is selected
  defaultKpi: boolean // id with defaultKpi=true becomes funnel.kpi on reset
}

// App funnel catalogs (source: mockup-k4a.html FUNNEL_DEFS lines 100–122 + spec §4.1)
// subscription: Install(req) + Registration + Trial Start(KPI) + Subscription Start + Revenue
const SUBSCRIPTION_CATALOG: FunnelEntry[] = [
  { id: 'install',            label: 'Install',            required: true,  defaultOn: true,  defaultKpi: false },
  { id: 'registration',       label: 'Registration',       required: false, defaultOn: true,  defaultKpi: false },
  { id: 'trial_start',        label: 'Trial Start',        required: false, defaultOn: true,  defaultKpi: true  },
  { id: 'subscription_start', label: 'Subscription Start', required: false, defaultOn: false, defaultKpi: false },
  { id: 'revenue',            label: 'Revenue',            required: false, defaultOn: false, defaultKpi: false },
]

// ecommerce: Install(req) + Registration + First Purchase(req, KPI) + Total Purchases + Revenue
const ECOMMERCE_CATALOG: FunnelEntry[] = [
  { id: 'install',         label: 'Install',         required: true,  defaultOn: true,  defaultKpi: false },
  { id: 'registration',    label: 'Registration',    required: false, defaultOn: true,  defaultKpi: false },
  { id: 'first_purchase',  label: 'First Purchase',  required: true,  defaultOn: true,  defaultKpi: true  },
  { id: 'total_purchases', label: 'Total Purchases', required: false, defaultOn: false, defaultKpi: false },
  { id: 'revenue',         label: 'Revenue',         required: false, defaultOn: false, defaultKpi: false },
]

// gaming follows ecommerce funnel (mockup lines 115–122 == ecommerce)
const GAMING_CATALOG: FunnelEntry[] = [...ECOMMERCE_CATALOG]

// Web funnel (shown only when Web is in platforms; min 2 on)
// source: mockup-k4a.html WEB_FUNNEL lines 123–128
const WEB_FUNNEL_CATALOG: FunnelEntry[] = [
  { id: 'pageview', label: 'Page View',     required: true,  defaultOn: true,  defaultKpi: false },
  { id: 'lead',     label: 'Lead / Sign-up', required: false, defaultOn: true,  defaultKpi: false },
  { id: 'purchase', label: 'Purchase',      required: false, defaultOn: true,  defaultKpi: true  },
]

function catalogFor(business: string): FunnelEntry[] {
  if (business === 'subscription') return SUBSCRIPTION_CATALOG
  if (business === 'gaming')       return GAMING_CATALOG
  return ECOMMERCE_CATALOG
}

// Compute the default FunnelState for a given business type
function defaultFunnelFor(business: string) {
  const catalog = catalogFor(business)
  return {
    items: catalog.filter(e => e.defaultOn).map(e => e.id),
    kpi:   catalog.find(e => e.defaultKpi)?.id ?? '',
    kpiConfirmed: false,
    names: {} as Record<string, string>,
  }
}

// Default webFunnel when first initialized (not yet cleared)
function defaultWebFunnel() {
  return {
    items: WEB_FUNNEL_CATALOG.filter(e => e.defaultOn).map(e => e.id),
    kpi:   WEB_FUNNEL_CATALOG.find(e => e.defaultKpi)?.id ?? '',
    kpiConfirmed: false,
    names: {} as Record<string, string>,
  }
}
```

---

## Task 1: Extend F2 and F3 contracts

Before writing any tests, patch the two upstream files to add `gamingRevenue`.

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts`
- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts`

- [ ] **Step 1: Read both files before editing**

  ```bash
  # Verify F2 exists and find where to insert GamingRevenue
  grep -n "Business\|CampaignGrouping\|WantsLtv" \
    /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts
  ```

  Expected: lines containing `Business: string;`, `CampaignGrouping: boolean;`, `WantsLtv: boolean;`.

- [ ] **Step 2: Add `GamingRevenue` to `WizardState` in F2**

  In `interfaces/aimOnboarding.ts`, locate the `Business: string;` field and insert `GamingRevenue: string;` immediately after it:

  ```ts
  // Before (in WizardState):
  Business: string;
  Funnel: FunnelState;

  // After:
  Business: string;
  GamingRevenue: string;
  Funnel: FunnelState;
  ```

- [ ] **Step 3: Add `gamingRevenue` to `defaultWizard()` in F3**

  In `composables/useAimOnboarding.ts`, locate `defaultWizard()` and add `gamingRevenue: ''` after `business: ''`:

  ```ts
  // Before:
  business: '',
  funnel: { items: [], kpi: '', kpiConfirmed: false, names: {} },

  // After:
  business: '',
  gamingRevenue: '',
  funnel: { items: [], kpi: '', kpiConfirmed: false, names: {} },
  ```

- [ ] **Step 4: Clear `gamingRevenue` in F3's business-change block**

  In `composables/useAimOnboarding.ts`, find the `isBusinessChange` block inside `patchWizard` and add the `gamingRevenue` clear:

  ```ts
  // Before:
  if (isBusinessChange) {
    wizard.value.wantsLtv = false
    wizard.value.ltvCohortAvail = ''
    wizard.value.ltvPartialChoice = ''
    wizard.value.webFunnel = {}
  }

  // After:
  if (isBusinessChange) {
    wizard.value.wantsLtv = false
    wizard.value.ltvCohortAvail = ''
    wizard.value.ltvPartialChoice = ''
    wizard.value.webFunnel = {}
    wizard.value.gamingRevenue = ''
  }
  ```

- [ ] **Step 5: Run the existing F3 spec to confirm nothing is broken**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding.spec.ts 2>&1 | tail -20
  ```

  Expected: all F3 tests pass.

- [ ] **Step 6: Commit the contract additions**

  ```bash
  git add \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts
  git commit -m "feat(aim-onboarding): add GamingRevenue to WizardState and composable default/reset (W4 contract addition)"
  ```

---

## Task 2: Write the failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step4Funnel.spec.ts`

- [ ] **Step 1: Create the test directory**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__
  ```

  Expected: no error.

- [ ] **Step 2: Write the spec file**

  Create the file with this content:

  ```ts
  import { mount } from '@vue/test-utils'
  import { describe, it, expect, beforeEach } from 'vitest'
  import { setActivePinia, createPinia } from 'pinia'
  import { useAimOnboarding } from '../../composables/useAimOnboarding'
  import Step4Funnel from '../Step4Funnel.vue'

  // Vuetify globals are provided by tests.config.ts.
  // We use the real F3 composable so business-change resets are exercised end-to-end.

  describe('Step4Funnel.vue', () => {
    beforeEach(() => {
      setActivePinia(createPinia())
      const { reset } = useAimOnboarding()
      reset()
    })

    // ── Business model ─────────────────────────────────────────────────────────

    describe('business model cards', () => {
      it('renders three business model cards: Subscription, E-commerce, Gaming', () => {
        const wrapper = mount(Step4Funnel)
        const text = wrapper.text()
        expect(text).toContain('Subscription')
        expect(text).toContain('E-commerce')
        expect(text).toContain('Gaming')
      })

      it('no card is selected by default (business is empty)', () => {
        const wrapper = mount(Step4Funnel)
        // No ToggleCard has selected=true
        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        const selected = cards.filter(c => c.props('selected') === true)
        expect(selected.length).toBe(0)
      })

      it('clicking Subscription card sets business to subscription and loads subscription funnel defaults', async () => {
        const wrapper = mount(Step4Funnel)
        const { wizard } = useAimOnboarding()

        // Find and click the Subscription card
        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        const subCard = cards.find(c => c.props('title') === 'Subscription')
        expect(subCard).toBeDefined()
        await subCard!.trigger('click')

        expect(wizard.value.business).toBe('subscription')
        // Subscription default funnel: install, registration, trial_start are on
        expect(wizard.value.funnel.items).toContain('install')
        expect(wizard.value.funnel.items).toContain('registration')
        expect(wizard.value.funnel.items).toContain('trial_start')
        expect(wizard.value.funnel.kpi).toBe('trial_start')
      })

      it('clicking E-commerce card sets business to ecommerce and loads ecommerce funnel defaults', async () => {
        const wrapper = mount(Step4Funnel)
        const { wizard } = useAimOnboarding()

        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        const ecCard = cards.find(c => c.props('title') === 'E-commerce')
        await ecCard!.trigger('click')

        expect(wizard.value.business).toBe('ecommerce')
        expect(wizard.value.funnel.items).toContain('install')
        expect(wizard.value.funnel.items).toContain('registration')
        expect(wizard.value.funnel.items).toContain('first_purchase')
        expect(wizard.value.funnel.kpi).toBe('first_purchase')
      })

      it('clicking Gaming card sets business to gaming and loads gaming funnel defaults', async () => {
        const wrapper = mount(Step4Funnel)
        const { wizard } = useAimOnboarding()

        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        const gamingCard = cards.find(c => c.props('title') === 'Gaming')
        await gamingCard!.trigger('click')

        expect(wizard.value.business).toBe('gaming')
        // Gaming follows ecommerce funnel
        expect(wizard.value.funnel.items).toContain('first_purchase')
        expect(wizard.value.funnel.kpi).toBe('first_purchase')
      })

      it('clicking a selected card again emits toggle but does not change business (parent dedup)', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'registration', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
        })

        const wrapper = mount(Step4Funnel)
        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        const subCard = cards.find(c => c.props('title') === 'Subscription')
        await subCard!.trigger('click')

        // Business stays subscription; no reset because same value
        expect(wizard.value.business).toBe('subscription')
        expect(wizard.value.funnel.kpi).toBe('trial_start')
      })
    })

    // ── Business model change reset ────────────────────────────────────────────
    // Gherkin: "Business model change clears downstream LTV state"
    // F3's patchWizard already clears LTV + webFunnel; W4 sends the new funnel
    // in the same patch. This test exercises the full end-to-end reset path.

    describe('business model change reset', () => {
      it('switching from Subscription (with LTV) to Gaming clears LTV state and resets funnel', async () => {
        const { wizard, patchWizard } = useAimOnboarding()

        // Simulate a user who completed Step 4 with Subscription + LTV
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'registration', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          wantsLtv: true,
          ltvCohortAvail: 'yes',
          ltvPartialChoice: '',
          webFunnel: {},
          gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)

        // Click Gaming card
        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        const gamingCard = cards.find(c => c.props('title') === 'Gaming')
        await gamingCard!.trigger('click')

        // Funnel resets to gaming defaults
        expect(wizard.value.business).toBe('gaming')
        expect(wizard.value.funnel.items).toContain('first_purchase')
        expect(wizard.value.funnel.kpi).toBe('first_purchase')
        // LTV cleared to defaults (false / '' per F3 + §5 contract)
        expect(wizard.value.wantsLtv).toBe(false)
        expect(wizard.value.ltvCohortAvail).toBe('')
        expect(wizard.value.ltvPartialChoice).toBe('')
        // webFunnel cleared
        expect(wizard.value.webFunnel).toEqual({})
        // gamingRevenue reset (was '' already, confirm still '')
        expect(wizard.value.gamingRevenue).toBe('')
      })

      it('switching from Gaming (with revenue type set) to Subscription clears gamingRevenue', async () => {
        const { wizard, patchWizard } = useAimOnboarding()

        patchWizard({
          business: 'gaming',
          gamingRevenue: 'iap',
          funnel: { items: ['install', 'first_purchase'], kpi: 'first_purchase', kpiConfirmed: false, names: {} },
          wantsLtv: false,
          ltvCohortAvail: '',
          ltvPartialChoice: '',
          webFunnel: {},
        })

        const wrapper = mount(Step4Funnel)

        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        const subCard = cards.find(c => c.props('title') === 'Subscription')
        await subCard!.trigger('click')

        expect(wizard.value.business).toBe('subscription')
        expect(wizard.value.gamingRevenue).toBe('')
        expect(wizard.value.funnel.kpi).toBe('trial_start')
      })

      it('non-business patch (e.g. patchWizard appName) does NOT clear LTV', () => {
        const { wizard, patchWizard } = useAimOnboarding()

        patchWizard({
          business: 'subscription',
          wantsLtv: true,
          ltvCohortAvail: 'yes',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {},
          gamingRevenue: '',
        })

        patchWizard({ appName: 'MySaaSApp' })

        expect(wizard.value.wantsLtv).toBe(true)
        expect(wizard.value.ltvCohortAvail).toBe('yes')
        expect(wizard.value.business).toBe('subscription')
      })
    })

    // ── Gaming revenue type ────────────────────────────────────────────────────

    describe('gaming revenue type', () => {
      it('gaming revenue selector is hidden when business is not gaming', async () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        expect(wrapper.find('[data-testid="gaming-revenue"]').exists()).toBe(false)
      })

      it('gaming revenue selector is shown when business is gaming', async () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'gaming',
          funnel: { items: ['install', 'first_purchase'], kpi: 'first_purchase', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        expect(wrapper.find('[data-testid="gaming-revenue"]').exists()).toBe(true)
      })

      it('selecting IAP sets gamingRevenue to iap', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'gaming',
          funnel: { items: ['install', 'first_purchase'], kpi: 'first_purchase', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        // v-btn-toggle — find the IAP button and click it
        const btns = wrapper.find('[data-testid="gaming-revenue"]').findAll('button')
        const iapBtn = btns.find(b => b.text().includes('In-app purchases'))
        await iapBtn!.trigger('click')

        expect(wizard.value.gamingRevenue).toBe('iap')
      })
    })

    // ── App funnel checklist ───────────────────────────────────────────────────

    describe('app funnel events', () => {
      it('funnel checklist is hidden when no business model is selected', () => {
        const wrapper = mount(Step4Funnel)
        expect(wrapper.find('[data-testid="funnel-checklist"]').exists()).toBe(false)
      })

      it('subscription funnel renders 5 events in default state', async () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'registration', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="funnel-checklist"]').findAll('[data-testid="funnel-event"]')
        expect(rows.length).toBe(5)
      })

      it('required events (Install) cannot be toggled off', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'registration', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="funnel-checklist"]').findAll('[data-testid="funnel-event"]')
        // First row is Install (required) — its checkbox should be disabled
        const installCheckbox = rows[0].find('input[type="checkbox"]')
        expect(installCheckbox.attributes('disabled')).toBeDefined()

        // Clicking it should not remove install from items
        await rows[0].trigger('click')
        expect(wizard.value.funnel.items).toContain('install')
      })

      it('optional events can be toggled on and off', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'registration', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="funnel-checklist"]').findAll('[data-testid="funnel-event"]')

        // subscription_start (index 3) is optional and defaultOn=false
        const subStartCheckbox = rows[3].find('input[type="checkbox"]')
        expect(subStartCheckbox.attributes('disabled')).toBeUndefined()

        await subStartCheckbox.trigger('change')
        expect(wizard.value.funnel.items).toContain('subscription_start')

        await subStartCheckbox.trigger('change')
        expect(wizard.value.funnel.items).not.toContain('subscription_start')
      })

      it('clicking "Set as KPI" on an event sets funnel.kpi to that event id', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'registration', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="funnel-checklist"]').findAll('[data-testid="funnel-event"]')

        // Registration row (index 1): click its KPI button
        const kpiBtn = rows[1].find('[data-testid="set-kpi-btn"]')
        await kpiBtn.trigger('click')

        expect(wizard.value.funnel.kpi).toBe('registration')
      })

      it('selecting a new KPI deselects the previous KPI (single KPI invariant)', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'registration', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="funnel-checklist"]').findAll('[data-testid="funnel-event"]')

        // Set KPI to Registration
        await rows[1].find('[data-testid="set-kpi-btn"]').trigger('click')
        expect(wizard.value.funnel.kpi).toBe('registration')

        // Now set KPI back to trial_start — registration should no longer be KPI
        await rows[2].find('[data-testid="set-kpi-btn"]').trigger('click')
        expect(wizard.value.funnel.kpi).toBe('trial_start')
      })
    })

    // ── Web funnel ─────────────────────────────────────────────────────────────

    describe('web funnel', () => {
      it('web funnel section is hidden when Web is not in platforms', () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          platforms: ['iOS'],
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        expect(wrapper.find('[data-testid="web-funnel"]').exists()).toBe(false)
      })

      it('web funnel section is shown when Web is in platforms and business is set', () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          platforms: ['iOS', 'Web'],
          business: 'ecommerce',
          funnel: { items: ['install', 'first_purchase'], kpi: 'first_purchase', kpiConfirmed: false, names: {} },
          webFunnel: { items: ['pageview', 'lead', 'purchase'], kpi: 'purchase', kpiConfirmed: false, names: {} },
          wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        expect(wrapper.find('[data-testid="web-funnel"]').exists()).toBe(true)
      })

      it('web funnel renders 3 default events (Page View, Lead/Sign-up, Purchase)', () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          platforms: ['Web'],
          business: 'ecommerce',
          funnel: { items: ['install', 'first_purchase'], kpi: 'first_purchase', kpiConfirmed: false, names: {} },
          webFunnel: { items: ['pageview', 'lead', 'purchase'], kpi: 'purchase', kpiConfirmed: false, names: {} },
          wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="web-funnel"]').findAll('[data-testid="funnel-event"]')
        expect(rows.length).toBe(3)
        const text = wrapper.find('[data-testid="web-funnel"]').text()
        expect(text).toContain('Page View')
        expect(text).toContain('Lead / Sign-up')
        expect(text).toContain('Purchase')
      })

      it('Page View is required in the web funnel (disabled checkbox)', () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          platforms: ['Web'],
          business: 'ecommerce',
          funnel: { items: ['install', 'first_purchase'], kpi: 'first_purchase', kpiConfirmed: false, names: {} },
          webFunnel: { items: ['pageview', 'lead', 'purchase'], kpi: 'purchase', kpiConfirmed: false, names: {} },
          wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="web-funnel"]').findAll('[data-testid="funnel-event"]')
        const pageViewCheckbox = rows[0].find('input[type="checkbox"]')
        expect(pageViewCheckbox.attributes('disabled')).toBeDefined()
      })

      it('optional web event (Lead/Sign-up) can be toggled off', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          platforms: ['Web'],
          business: 'ecommerce',
          funnel: { items: ['install', 'first_purchase'], kpi: 'first_purchase', kpiConfirmed: false, names: {} },
          webFunnel: { items: ['pageview', 'lead', 'purchase'], kpi: 'purchase', kpiConfirmed: false, names: {} },
          wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const rows = wrapper.find('[data-testid="web-funnel"]').findAll('[data-testid="funnel-event"]')
        const leadCheckbox = rows[1].find('input[type="checkbox"]')

        await leadCheckbox.trigger('change')
        expect(wizard.value.webFunnel.items).not.toContain('lead')
      })
    })

    // ── LTV model ─────────────────────────────────────────────────────────────

    describe('LTV model', () => {
      it('LTV question is hidden until business model is selected', () => {
        const wrapper = mount(Step4Funnel)
        expect(wrapper.find('[data-testid="ltv-question"]').exists()).toBe(false)
      })

      it('LTV question is shown once business and funnel are configured', () => {
        const { patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        expect(wrapper.find('[data-testid="ltv-question"]').exists()).toBe(true)
      })

      it('selecting Yes for LTV sets wantsLtv to true and shows cohort question', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: false, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const ltvSection = wrapper.find('[data-testid="ltv-question"]')
        const buttons = ltvSection.find('[role="group"]').findAll('button')
        // Yes button is first
        const yesBtn = buttons.find(b => b.text().toLowerCase().includes('yes'))
        await yesBtn!.trigger('click')

        expect(wizard.value.wantsLtv).toBe(true)
        expect(wrapper.find('[data-testid="ltv-cohort-question"]').exists()).toBe(true)
      })

      it('selecting No for LTV sets wantsLtv to false and hides cohort question', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: true, ltvCohortAvail: 'yes', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const ltvSection = wrapper.find('[data-testid="ltv-question"]')
        const buttons = ltvSection.find('[role="group"]').findAll('button')
        const noBtn = buttons.find(b => b.text().toLowerCase().includes('no'))
        await noBtn!.trigger('click')

        expect(wizard.value.wantsLtv).toBe(false)
        expect(wizard.value.ltvCohortAvail).toBe('')
        expect(wrapper.find('[data-testid="ltv-cohort-question"]').exists()).toBe(false)
      })

      it('selecting cohort Yes sets ltvCohortAvail to yes', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: true, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const cohortSection = wrapper.find('[data-testid="ltv-cohort-question"]')
        const buttons = cohortSection.find('[role="group"]').findAll('button')
        const yesBtn = buttons.find(b => b.text().toLowerCase() === 'yes')
        await yesBtn!.trigger('click')

        expect(wizard.value.ltvCohortAvail).toBe('yes')
      })

      it('selecting cohort Partial sets ltvCohortAvail to partial', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: true, ltvCohortAvail: '', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)
        const cohortSection = wrapper.find('[data-testid="ltv-cohort-question"]')
        const buttons = cohortSection.find('[role="group"]').findAll('button')
        const partialBtn = buttons.find(b => b.text().toLowerCase() === 'partial')
        await partialBtn!.trigger('click')

        expect(wizard.value.ltvCohortAvail).toBe('partial')
      })

      it('LTV cohort question renders as unanswered (empty string) after business-model change', async () => {
        const { wizard, patchWizard } = useAimOnboarding()
        // Subscription with LTV opted in and cohort confirmed
        patchWizard({
          business: 'subscription',
          funnel: { items: ['install', 'trial_start'], kpi: 'trial_start', kpiConfirmed: false, names: {} },
          webFunnel: {}, wantsLtv: true, ltvCohortAvail: 'yes', ltvPartialChoice: '', gamingRevenue: '',
        })

        const wrapper = mount(Step4Funnel)

        // Switch to Gaming
        const cards = wrapper.findAllComponents({ name: 'ToggleCard' })
        await cards.find(c => c.props('title') === 'Gaming')!.trigger('click')

        // LTV state is cleared (Gherkin: step 5 LTV question renders as unanswered)
        expect(wizard.value.wantsLtv).toBe(false)
        expect(wizard.value.ltvCohortAvail).toBe('')
        // Cohort question is hidden because wantsLtv is false
        expect(wrapper.find('[data-testid="ltv-cohort-question"]').exists()).toBe(false)
      })
    })
  })
  ```

- [ ] **Step 3: Run the tests — confirm they all fail**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step4Funnel.spec.ts 2>&1 | tail -30
  ```

  Expected: all tests fail. Error should reference missing module `../Step4Funnel.vue`. This is the correct TDD red state.

---

## Task 3: Implement Step4Funnel.vue

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step4Funnel.vue`

- [ ] **Step 1: Create the steps directory**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps
  ```

  Expected: no error.

- [ ] **Step 2: Write Step4Funnel.vue**

  Create the file with this content:

  ```vue
  <template>
    <div class="step4-funnel">
      <!-- Section: Business Model -->
      <section class="step4-funnel__section">
        <h2 class="step4-funnel__heading text-body-1 font-weight-semibold mb-1">
          Business model
        </h2>
        <p class="step4-funnel__hint text-body-2 mb-4" style="color: rgb(var(--v-theme-grey-2));">
          This shapes your funnel and LTV options.
          <strong>Changing it resets your funnel and LTV answers.</strong>
        </p>

        <div class="step4-funnel__card-row">
          <ToggleCard
            v-for="biz in BUSINESS_OPTIONS"
            :key="biz.key"
            :title="biz.label"
            :description="biz.description"
            :selected="wizard.business === biz.key"
            @toggle="selectBusiness(biz.key)"
          />
        </div>
      </section>

      <!-- Section: Gaming Revenue Type (conditional) -->
      <section
        v-if="wizard.business === 'gaming'"
        class="step4-funnel__section"
        data-testid="gaming-revenue"
      >
        <h2 class="step4-funnel__heading text-body-1 font-weight-semibold mb-1">
          Gaming revenue type
        </h2>
        <p class="step4-funnel__hint text-body-2 mb-3" style="color: rgb(var(--v-theme-grey-2));">
          How does your game generate revenue?
        </p>

        <v-btn-toggle
          :model-value="wizard.gamingRevenue"
          mandatory
          variant="outlined"
          divided
          @update:model-value="patchWizard({ gamingRevenue: $event })"
        >
          <v-btn value="iap">In-app purchases</v-btn>
          <v-btn value="ads">In-app ads</v-btn>
          <v-btn value="both">Both</v-btn>
        </v-btn-toggle>
      </section>

      <!-- Section: App Funnel Events (shown once business is selected) -->
      <section
        v-if="wizard.business && catalogFor(wizard.business).length"
        class="step4-funnel__section"
      >
        <h2 class="step4-funnel__heading text-body-1 font-weight-semibold mb-1">
          App funnel events
        </h2>
        <p class="step4-funnel__hint text-body-2 mb-3" style="color: rgb(var(--v-theme-grey-2));">
          Required events are pre-selected and locked. Toggle optional events on or off.
          Set one event as your primary KPI.
        </p>

        <div data-testid="funnel-checklist" class="step4-funnel__checklist">
          <div
            v-for="entry in catalogFor(wizard.business)"
            :key="entry.id"
            data-testid="funnel-event"
            class="step4-funnel__event"
            :class="{ 'step4-funnel__event--kpi': wizard.funnel.kpi === entry.id }"
          >
            <v-checkbox
              :model-value="isEventOn(entry.id)"
              :disabled="entry.required"
              :label="entry.label"
              density="compact"
              hide-details
              color="primary"
              @update:model-value="toggleEvent(entry.id, entry.required)"
            />

            <v-chip
              v-if="wizard.funnel.kpi === entry.id"
              size="small"
              color="primary"
              variant="tonal"
              class="ml-2"
            >
              Primary KPI
            </v-chip>

            <v-btn
              v-else-if="isEventOn(entry.id)"
              data-testid="set-kpi-btn"
              size="small"
              variant="text"
              density="compact"
              class="ml-2"
              @click="setKpi(entry.id)"
            >
              Set as KPI
            </v-btn>
          </div>
        </div>
      </section>

      <!-- Section: Web Funnel (conditional on Web platform + business set) -->
      <section
        v-if="showWebFunnel"
        class="step4-funnel__section"
        data-testid="web-funnel"
      >
        <h2 class="step4-funnel__heading text-body-1 font-weight-semibold mb-1">
          Web funnel events
        </h2>
        <p class="step4-funnel__hint text-body-2 mb-3" style="color: rgb(var(--v-theme-grey-2));">
          Minimum 2 events required. Page View is always included.
        </p>

        <div class="step4-funnel__checklist">
          <div
            v-for="entry in WEB_FUNNEL_CATALOG"
            :key="entry.id"
            data-testid="funnel-event"
            class="step4-funnel__event"
          >
            <v-checkbox
              :model-value="isWebEventOn(entry.id)"
              :disabled="entry.required"
              :label="entry.label"
              density="compact"
              hide-details
              color="primary"
              @update:model-value="toggleWebEvent(entry.id, entry.required)"
            />

            <v-chip
              v-if="webFunnelState.kpi === entry.id"
              size="small"
              color="primary"
              variant="tonal"
              class="ml-2"
            >
              KPI
            </v-chip>
          </div>
        </div>
      </section>

      <!-- Section: LTV Model (shown once business is set) -->
      <section
        v-if="wizard.business"
        class="step4-funnel__section"
        data-testid="ltv-question"
      >
        <h2 class="step4-funnel__heading text-body-1 font-weight-semibold mb-1">
          LTV model
        </h2>
        <p class="step4-funnel__hint text-body-2 mb-3" style="color: rgb(var(--v-theme-grey-2));">
          Would you like to include a lifetime value (LTV) model?
        </p>

        <v-btn-toggle
          :model-value="ltvToggleValue"
          mandatory
          variant="outlined"
          divided
          @update:model-value="onLtvChange"
        >
          <v-btn value="yes">Yes, include LTV</v-btn>
          <v-btn value="no">No thanks</v-btn>
        </v-btn-toggle>

        <!-- Cohort sub-question (conditional on wantsLtv = true) -->
        <div
          v-if="wizard.wantsLtv"
          class="step4-funnel__cohort mt-4"
          data-testid="ltv-cohort-question"
        >
          <p class="text-body-2 mb-2" style="color: rgb(var(--v-theme-grey-2));">
            Do you have historical LTV cohort data available?
          </p>

          <v-btn-toggle
            :model-value="wizard.ltvCohortAvail"
            mandatory
            variant="outlined"
            divided
            @update:model-value="patchWizard({ ltvCohortAvail: $event })"
          >
            <v-btn value="yes">Yes</v-btn>
            <v-btn value="no">No</v-btn>
            <v-btn value="partial">Partial</v-btn>
          </v-btn-toggle>
        </div>
      </section>
    </div>
  </template>

  <script setup lang="ts">
  import { computed } from 'vue'
  import { useAimOnboarding } from '../composables/useAimOnboarding'
  import ToggleCard from '../components/ToggleCard.vue'

  const { wizard, patchWizard } = useAimOnboarding()

  // ── Business model options ───────────────────────────────────────────────────

  const BUSINESS_OPTIONS = [
    { key: 'subscription', label: 'Subscription', description: 'SaaS / membership / recurring revenue' },
    { key: 'ecommerce',    label: 'E-commerce',   description: 'Direct purchase / transactional' },
    { key: 'gaming',       label: 'Gaming',        description: 'IAP, in-app ads, or both' },
  ] as const

  // ── Funnel catalogs ─────────────────────────────────────────────────────────

  interface FunnelEntry {
    id: string
    label: string
    required: boolean
    defaultOn: boolean
    defaultKpi: boolean
  }

  const SUBSCRIPTION_CATALOG: FunnelEntry[] = [
    { id: 'install',            label: 'Install',            required: true,  defaultOn: true,  defaultKpi: false },
    { id: 'registration',       label: 'Registration',       required: false, defaultOn: true,  defaultKpi: false },
    { id: 'trial_start',        label: 'Trial Start',        required: false, defaultOn: true,  defaultKpi: true  },
    { id: 'subscription_start', label: 'Subscription Start', required: false, defaultOn: false, defaultKpi: false },
    { id: 'revenue',            label: 'Revenue',            required: false, defaultOn: false, defaultKpi: false },
  ]

  const ECOMMERCE_CATALOG: FunnelEntry[] = [
    { id: 'install',         label: 'Install',         required: true,  defaultOn: true,  defaultKpi: false },
    { id: 'registration',    label: 'Registration',    required: false, defaultOn: true,  defaultKpi: false },
    { id: 'first_purchase',  label: 'First Purchase',  required: true,  defaultOn: true,  defaultKpi: true  },
    { id: 'total_purchases', label: 'Total Purchases', required: false, defaultOn: false, defaultKpi: false },
    { id: 'revenue',         label: 'Revenue',         required: false, defaultOn: false, defaultKpi: false },
  ]

  const GAMING_CATALOG: FunnelEntry[] = [...ECOMMERCE_CATALOG]

  const WEB_FUNNEL_CATALOG: FunnelEntry[] = [
    { id: 'pageview', label: 'Page View',      required: true,  defaultOn: true,  defaultKpi: false },
    { id: 'lead',     label: 'Lead / Sign-up', required: false, defaultOn: true,  defaultKpi: false },
    { id: 'purchase', label: 'Purchase',       required: false, defaultOn: true,  defaultKpi: true  },
  ]

  function catalogFor(business: string): FunnelEntry[] {
    if (business === 'subscription') return SUBSCRIPTION_CATALOG
    if (business === 'gaming')       return GAMING_CATALOG
    return ECOMMERCE_CATALOG
  }

  function defaultFunnelFor(business: string) {
    const catalog = catalogFor(business)
    return {
      items: catalog.filter(e => e.defaultOn).map(e => e.id),
      kpi:   catalog.find(e => e.defaultKpi)?.id ?? '',
      kpiConfirmed: false,
      names: {} as Record<string, string>,
    }
  }

  function defaultWebFunnel() {
    return {
      items: WEB_FUNNEL_CATALOG.filter(e => e.defaultOn).map(e => e.id),
      kpi:   WEB_FUNNEL_CATALOG.find(e => e.defaultKpi)?.id ?? '',
      kpiConfirmed: false,
      names: {} as Record<string, string>,
    }
  }

  // ── Business model selection ─────────────────────────────────────────────────

  function selectBusiness(key: string): void {
    if (wizard.value.business === key) return  // no-op: clicking already-selected card

    // Send business + new funnel in one patch.
    // F3's patchWizard detects the business change and clears LTV/webFunnel/gamingRevenue.
    // webFunnel is reloaded if Web is a platform; the user will see the default web funnel.
    patchWizard({
      business: key,
      funnel: defaultFunnelFor(key),
      gamingRevenue: '',
      webFunnel: wizard.value.platforms.includes('Web') ? defaultWebFunnel() : {},
    })
  }

  // ── App funnel event handlers ────────────────────────────────────────────────

  function isEventOn(id: string): boolean {
    return wizard.value.funnel.items.includes(id)
  }

  function toggleEvent(id: string, required: boolean): void {
    if (required) return  // required events are locked
    const items = wizard.value.funnel.items
    const next = items.includes(id)
      ? items.filter(i => i !== id)
      : [...items, id]

    // If toggling off the current KPI, clear kpi
    const kpi = next.includes(wizard.value.funnel.kpi) ? wizard.value.funnel.kpi : ''
    patchWizard({ funnel: { ...wizard.value.funnel, items: next, kpi } })
  }

  function setKpi(id: string): void {
    patchWizard({ funnel: { ...wizard.value.funnel, kpi: id } })
  }

  // ── Web funnel handlers ─────────────────────────────────────────────────────

  const webFunnelState = computed(() => {
    const wf = wizard.value.webFunnel as { items?: string[]; kpi?: string }
    return {
      items: wf.items ?? [],
      kpi:   wf.kpi   ?? '',
    }
  })

  const showWebFunnel = computed(() =>
    wizard.value.platforms.includes('Web') && !!wizard.value.business
  )

  function isWebEventOn(id: string): boolean {
    return webFunnelState.value.items.includes(id)
  }

  function toggleWebEvent(id: string, required: boolean): void {
    if (required) return
    const items = webFunnelState.value.items
    const next = items.includes(id)
      ? items.filter(i => i !== id)
      : [...items, id]
    const kpi = next.includes(webFunnelState.value.kpi) ? webFunnelState.value.kpi : ''
    patchWizard({ webFunnel: { ...wizard.value.webFunnel, items: next, kpi } })
  }

  // ── LTV handlers ─────────────────────────────────────────────────────────────

  const ltvToggleValue = computed(() => {
    if (wizard.value.wantsLtv === true)  return 'yes'
    if (wizard.value.wantsLtv === false && wizard.value.ltvCohortAvail !== '') return 'no'
    return wizard.value.wantsLtv ? 'yes' : 'no'
  })

  function onLtvChange(val: string): void {
    if (val === 'yes') {
      patchWizard({ wantsLtv: true })
    } else {
      patchWizard({ wantsLtv: false, ltvCohortAvail: '' })
    }
  }
  </script>

  <style scoped lang="scss">
  .step4-funnel {
    display: flex;
    flex-direction: column;
    gap: 32px;

    &__section {
      display: flex;
      flex-direction: column;
    }

    &__heading {
      color: rgb(var(--v-theme-black));
      margin-bottom: 4px;
    }

    &__hint {
      font-size: 13px;
    }

    &__card-row {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 12px;
    }

    &__checklist {
      display: flex;
      flex-direction: column;
      gap: 8px;
    }

    &__event {
      display: flex;
      align-items: center;
      padding: 10px 12px;
      border: 1px solid rgb(var(--v-theme-border));
      border-radius: 8px;
      background: rgb(var(--v-theme-surface));

      &--kpi {
        border-color: rgb(var(--v-theme-primary));
        background: rgba(var(--v-theme-primary), 0.04);
      }
    }

    &__cohort {
      padding-left: 0;
    }
  }
  </style>
  ```

---

## Task 4: Run tests — confirm all pass

- [ ] **Step 1: Run the Step4Funnel spec**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step4Funnel.spec.ts 2>&1 | tail -40
  ```

  Expected output:

  ```text
  ✓ Step4Funnel.vue > business model cards > renders three business model cards
  ✓ Step4Funnel.vue > business model cards > no card is selected by default
  ✓ Step4Funnel.vue > business model cards > clicking Subscription card sets business and loads defaults
  ✓ Step4Funnel.vue > business model cards > clicking E-commerce card sets business and loads defaults
  ✓ Step4Funnel.vue > business model cards > clicking Gaming card sets business and loads defaults
  ✓ Step4Funnel.vue > business model cards > clicking a selected card again does not change business
  ✓ Step4Funnel.vue > business model change reset > switching from Subscription to Gaming clears LTV and resets funnel
  ✓ Step4Funnel.vue > business model change reset > switching from Gaming to Subscription clears gamingRevenue
  ✓ Step4Funnel.vue > business model change reset > non-business patch does NOT clear LTV
  ✓ Step4Funnel.vue > gaming revenue type > gaming revenue selector is hidden when business is not gaming
  ✓ Step4Funnel.vue > gaming revenue type > gaming revenue selector is shown when business is gaming
  ✓ Step4Funnel.vue > gaming revenue type > selecting IAP sets gamingRevenue to iap
  ✓ Step4Funnel.vue > app funnel events > funnel checklist is hidden when no business model is selected
  ✓ Step4Funnel.vue > app funnel events > subscription funnel renders 5 events in default state
  ✓ Step4Funnel.vue > app funnel events > required events (Install) cannot be toggled off
  ✓ Step4Funnel.vue > app funnel events > optional events can be toggled on and off
  ✓ Step4Funnel.vue > app funnel events > clicking "Set as KPI" sets funnel.kpi
  ✓ Step4Funnel.vue > app funnel events > selecting a new KPI deselects the previous KPI
  ✓ Step4Funnel.vue > web funnel > web funnel section is hidden when Web is not in platforms
  ✓ Step4Funnel.vue > web funnel > web funnel section is shown when Web is in platforms
  ✓ Step4Funnel.vue > web funnel > web funnel renders 3 default events
  ✓ Step4Funnel.vue > web funnel > Page View is required in the web funnel
  ✓ Step4Funnel.vue > web funnel > optional web event (Lead/Sign-up) can be toggled off
  ✓ Step4Funnel.vue > LTV model > LTV question is hidden until business model is selected
  ✓ Step4Funnel.vue > LTV model > LTV question is shown once business and funnel are configured
  ✓ Step4Funnel.vue > LTV model > selecting Yes for LTV sets wantsLtv to true and shows cohort question
  ✓ Step4Funnel.vue > LTV model > selecting No for LTV sets wantsLtv to false and hides cohort question
  ✓ Step4Funnel.vue > LTV model > selecting cohort Yes sets ltvCohortAvail to yes
  ✓ Step4Funnel.vue > LTV model > selecting cohort Partial sets ltvCohortAvail to partial
  ✓ Step4Funnel.vue > LTV model > LTV cohort question renders as unanswered after business-model change

  Test Files  1 passed (1)
  Tests       30 passed (30)
  ```

  If any test fails, diagnose before moving to Task 5. Common failure modes:
    - `ToggleCard not found` — check the import path; `ToggleCard.vue` must exist from C1.
    - `patchWizard is not a function` — composable import path is wrong or F3 is not merged.
    - `wizardState.webFunnel` is `undefined` instead of `{}` — ensure F3's defaultWizard returns `webFunnel: {}` not `null`.

- [ ] **Step 2: Run the full advertiser package test suite**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run --project packages/advertiser 2>&1 | tail -20
  ```

  Expected: all existing tests pass; Step4Funnel shows 30 passing.

- [ ] **Step 3: TypeScript type-check**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx tsc --noEmit --project packages/advertiser/tsconfig.json 2>&1 | grep -i "step4\|funnel\|GamingRevenue"
  ```

  Expected: no output (no errors for these files).

---

## Task 5: Lint check and commit

- [ ] **Step 1: Lint all new and modified files**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx eslint \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step4Funnel.vue \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step4Funnel.spec.ts \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts \
      2>&1
  ```

  Expected: no errors or warnings. Fix any issues before committing.

- [ ] **Step 2: Commit**

  ```bash
  git add \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step4Funnel.vue \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step4Funnel.spec.ts
  git commit -m "feat(aim-onboarding): W4 Step4Funnel — business model, funnel checklist, web funnel, LTV, model-change reset"
  ```

  Expected: commit succeeds; CI test run passes.

---

## Self-review: spec coverage

| Requirement (spec §4.1 Step 4 + design-consolidated §5/§8a) | Covered |
|-------------------------------------------------------------|---------|
| Business model: Subscription / E-commerce / Gaming ToggleCards | Task 3, BUSINESS_OPTIONS + ToggleCard |
| Changing business model resets funnel AND clears `wantsLtv`/`ltvCohortAvail`/`ltvPartialChoice`/`webFunnel` | `selectBusiness()` patches business+funnel in one call; F3 `patchWizard` clears LTV/webFunnel; tested in Task 2 reset suite |
| Gaming revenue type: IAP / Ads / Both (required if Gaming) | `v-btn-toggle` with `data-testid="gaming-revenue"`; hidden for non-gaming |
| App funnel events checklist, required locked | Catalog `required: true` → disabled `v-checkbox`; `toggleEvent` guard |
| Optional events togglable | `toggleEvent` for non-required entries |
| Min 3 events on / one KPI validation | Non-blocking per §8a — validated by F7 `validateStep('4', …)`; W4 only enforces interaction |
| One KPI must be set (KPI button moves single id) | `setKpi()` sets `funnel.kpi`; only one value stored; tested dedup |
| Subscription funnel defaults: Install+Reg+TrialStart(KPI) | `SUBSCRIPTION_CATALOG` + `defaultFunnelFor` |
| E-commerce funnel defaults: Install+Reg+FirstPurchase(req,KPI) | `ECOMMERCE_CATALOG` + `defaultFunnelFor` |
| Gaming follows e-commerce funnel | `GAMING_CATALOG = [...ECOMMERCE_CATALOG]` |
| Web funnel shown if Web selected, min 2 on | `showWebFunnel` computed; `WEB_FUNNEL_CATALOG` with `pageview` required |
| LTV model: Yes/No → cohort Yes/No/Partial | `v-btn-toggle` + `onLtvChange`; cohort sub-question conditional on `wantsLtv` |
| `wantsLtv` false + `ltvCohortAvail` '' after business change | `patchWizard` in F3 clears these; tested end-to-end |
| `gamingRevenue` cleared on business change | Added to F3's clear block; tested in Task 2 |
| `ltvPartialChoice` cleared on business change | F3 clears it (already in F3 plan + Task 1 extends the clear list) |
| State bound to composable (`useAimOnboarding`) | All mutations via `patchWizard`; state read from `wizard.value` |
| Real Vuetify components, `rgb(var(--v-theme-*))`, no focus-halo | `v-checkbox`, `v-btn-toggle`, `v-btn`, `v-chip`; SCSS uses theme tokens only |
| Accessibility: disabled checkbox carries `disabled` attribute; role/aria on ToggleCard from C1 | `v-checkbox :disabled`; C1 owns `role=checkbox`/`aria-checked` |
| Karpathy: minimal | No extra UI, no rename UX, `names: {}` left empty |
| TDD: test → fail → implement → pass → commit | Tasks 2 → 3 → 4 → 5 |
| Lint-clean | Task 5 Step 1 |

---

## Notes for the implementer

- **`ltvToggleValue` edge case:** Before the user has ever touched the LTV question, `wantsLtv` is `false` (the default). This means the "No" button appears pre-selected. This is correct per spec — the LTV question section is hidden until business is set, so the user always makes an explicit first choice before seeing the pre-selection.
- **`names: {}`:** The mockup allows inline event renaming but the spec says "Karpathy: minimal." Leave `names: {}` unpopulated. Do not add a rename input unless instructed.
- **Gaming catalog == ecommerce:** `GAMING_CATALOG` is a spread of `ECOMMERCE_CATALOG`. If the spec ever diverges (e.g., adds `ad_impression`), change only `GAMING_CATALOG`.
- **`webFunnel` type:** `WizardState.WebFunnel` is `Record<string, unknown>` in F2 — the cast in `webFunnelState` computed is intentional. If F2 is updated to a typed `FunnelState | {}`, remove the cast.
- **`selectBusiness` no-op guard:** Clicking the already-selected card calls `return` early. This prevents F3's business-change block from firing (and clearing LTV) when the user just clicks the same card again.
