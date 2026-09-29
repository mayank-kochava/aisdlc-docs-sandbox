---
id: plan-f3
title: "F3 — useAimOnboarding Composable"
---

## Goal

Implement `composables/useAimOnboarding.ts` inside `AimOnboardingTab/` — the flat wizard state
object (§5 shape), the two computed routing flags (`uaUeRoutingFlag`, `brandRoutingFlag`), step
navigation, and per-step validation/completeness derivation per §8a. Pure state + derivation only;
delegates field-level validation to F7 (`logic/validation.ts`). Autosave (F4) is a separate plan.

---

## Files

**Create**

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding.spec.ts
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.ts
```

**Depends on (must exist)**

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts   ← F2
```

---

## Dependencies

**F2** — `interfaces/aimOnboarding.ts` must be merged before this plan. The composable imports
`WizardState` from that interface file.

**F7** — `logic/validation.ts` is consumed by the composable. This plan ships a minimal stub that
F7 replaces with the full field-rule implementation.

---

## §8a Validation Rules (authoritative — transcribed verbatim)

These are the rules that drive `stepStatus`. No other interpretation is valid.

| State | Condition |
|-------|-----------|
| `done` | Every *active* required field for the step is valid |
| `error` | The step has been visited AND at least one active required field is empty/invalid |
| `pending` | Step not yet visited and not yet done (not an error, not revealed as blocked) |
| `optional` | Steps 6 (External Factors) and 7 (Objectives) — never block, never count toward required-completeness |

**Dimmed** is a field-level concept from §8a ("a conditional question whose parent isn't answered
yet — excluded from validation until revealed"). It is enforced by `validateStep` in F7, which
excludes inactive conditional fields from the `invalidFields` list. The step-level statuses are
`done | error | pending | optional`.

Output "Approve" is blocked until all required steps (`1–5`, `8`) have status `done`.

---

## Routing Flag Rules (from `mockup-k4a.html` `calcFlags()`)

```ts
// uaUeRoutingFlag
if (!wizard.ua && !wizard.ue) {
  uaUeRoutingFlag = ''
} else if (wizard.ua && wizard.ue) {
  const u = campaignShare?.ua ?? 0
  const e = campaignShare?.ue ?? 0
  uaUeRoutingFlag =
    u >= 10 && u <= 90 && e >= 10 && e <= 90
      ? 'aim_pro_eligible'
      : 'aim_x_preferred'
} else {
  uaUeRoutingFlag = 'aim_x_preferred'
}

// brandRoutingFlag
brandRoutingFlag = !!wizard.brand
```

`campaignShare` is a numeric per-type budget-split object (`{ ua, ue, brand }`) added to the §5
`wizard` contract alongside the existing `uaShare`/`ueShare` string fields. Confirm with F2 that
`wizard.campaignShare` is added to `WizardState` and the §5 Mongo shape; it must survive F4
autosave round-trips.

---

## Singleton State Pattern (mirror: `useValidateOnboarding.ts`)

Like the mirror composable, refs are declared **at module scope** so every component that calls
`useAimOnboarding()` shares the same wizard instance. This is required because the step
components, StepRail, SummaryPanel, and F4 autosave all consume this composable and must operate
on one shared state.

```ts
// Module-scope singletons (outside the function body)
const wizard = ref<WizardState>(defaultWizard())
const currentStep = ref(1)
const visitedSteps = ref<Set<number>>(new Set())
const campaignShare = ref<CampaignShare>({ ua: 0, ue: 0, brand: 0 })

export function useAimOnboarding() { ... }
```

A `reset()` function (returned from the composable) restores all module-scope refs to their
defaults. Tests call `reset()` in `beforeEach` to prevent cross-test state leakage.

---

## Tasks

### Task 1 — Write the failing spec

**1.1** Create the spec file at:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding.spec.ts
```

Contents:

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'

// Stub F7 logic/validation — replaced by the real implementation in F7.
vi.mock('../../logic/validation', () => ({
  validateStep: vi.fn(
    (_stepKey: string, _wizard: unknown): { valid: boolean; invalidFields: string[] } => ({
      valid: false,
      invalidFields: [],
    }),
  ),
}))

import { useAimOnboarding } from '../useAimOnboarding'

describe('useAimOnboarding', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    const { reset } = useAimOnboarding()
    reset()
    vi.clearAllMocks()
  })

  // ── State shape ─────────────────────────────────────────────────────────────

  describe('initial state', () => {
    it('wizard initialises to the §5 defaults', () => {
      const { wizard } = useAimOnboarding()
      expect(wizard.value.companyName).toBe('')
      expect(wizard.value.platforms).toEqual([])
      expect(wizard.value.ua).toBe(false)
      expect(wizard.value.ue).toBe(false)
      expect(wizard.value.brand).toBe(false)
      expect(wizard.value.uaUeRoutingFlag).toBe('')
      expect(wizard.value.brandRoutingFlag).toBe(false)
      expect(wizard.value.budgetPeriod).toBe('monthly')
      expect(wizard.value.offlineSplitPct).toBe(20)
      expect(wizard.value.paidSplit).toEqual({ iOS: 65, Android: 80, Web: 50 })
      expect(wizard.value.funnel).toEqual({ items: [], kpi: '', kpiConfirmed: false, names: {} })
      expect(wizard.value.projectLeads).toEqual([])
      expect(wizard.value.dataLeads).toEqual([])
    })

    it('currentStep starts at 1', () => {
      const { currentStep } = useAimOnboarding()
      expect(currentStep.value).toBe(1)
    })

    it('visitedSteps starts empty', () => {
      const { visitedSteps } = useAimOnboarding()
      expect(visitedSteps.value.size).toBe(0)
    })
  })

  // ── Routing flags ────────────────────────────────────────────────────────────

  describe('uaUeRoutingFlag', () => {
    it('is empty string when neither UA nor UE selected', () => {
      const { wizard, uaUeRoutingFlag } = useAimOnboarding()
      wizard.value.ua = false
      wizard.value.ue = false
      expect(uaUeRoutingFlag.value).toBe('')
    })

    it('is aim_x_preferred when only UA selected', () => {
      const { wizard, uaUeRoutingFlag } = useAimOnboarding()
      wizard.value.ua = true
      wizard.value.ue = false
      expect(uaUeRoutingFlag.value).toBe('aim_x_preferred')
    })

    it('is aim_x_preferred when only UE selected', () => {
      const { wizard, uaUeRoutingFlag } = useAimOnboarding()
      wizard.value.ua = false
      wizard.value.ue = true
      expect(uaUeRoutingFlag.value).toBe('aim_x_preferred')
    })

    it('is aim_pro_eligible when both UA+UE each within 10–90%', () => {
      const { wizard, campaignShare, uaUeRoutingFlag } = useAimOnboarding()
      wizard.value.ua = true
      wizard.value.ue = true
      campaignShare.value.ua = 65
      campaignShare.value.ue = 35
      expect(uaUeRoutingFlag.value).toBe('aim_pro_eligible')
    })

    it('is aim_x_preferred when both UA+UE but one share outside 10–90%', () => {
      const { wizard, campaignShare, uaUeRoutingFlag } = useAimOnboarding()
      wizard.value.ua = true
      wizard.value.ue = true
      campaignShare.value.ua = 95
      campaignShare.value.ue = 5
      expect(uaUeRoutingFlag.value).toBe('aim_x_preferred')
    })
  })

  describe('brandRoutingFlag', () => {
    it('is false when brand not selected', () => {
      const { wizard, brandRoutingFlag } = useAimOnboarding()
      wizard.value.brand = false
      expect(brandRoutingFlag.value).toBe(false)
    })

    it('is true when brand selected', () => {
      const { wizard, brandRoutingFlag } = useAimOnboarding()
      wizard.value.brand = true
      expect(brandRoutingFlag.value).toBe(true)
    })
  })

  // ── Step navigation ──────────────────────────────────────────────────────────

  describe('navigation', () => {
    it('goToStep sets currentStep and marks previous step visited', () => {
      const { currentStep, visitedSteps, goToStep } = useAimOnboarding()
      goToStep(2)
      expect(currentStep.value).toBe(2)
      expect(visitedSteps.value.has(1)).toBe(true)
    })

    it('nextStep advances by 1 and marks current visited', () => {
      const { currentStep, visitedSteps, nextStep } = useAimOnboarding()
      nextStep()
      expect(currentStep.value).toBe(2)
      expect(visitedSteps.value.has(1)).toBe(true)
    })

    it('prevStep decrements by 1', () => {
      const { currentStep, nextStep, prevStep } = useAimOnboarding()
      nextStep()
      prevStep()
      expect(currentStep.value).toBe(1)
    })

    it('prevStep does not go below 1', () => {
      const { currentStep, prevStep } = useAimOnboarding()
      prevStep()
      expect(currentStep.value).toBe(1)
    })

    it('nextStep does not advance past step 8', () => {
      const { currentStep, goToStep, nextStep } = useAimOnboarding()
      goToStep(8)
      nextStep()
      expect(currentStep.value).toBe(8)
    })
  })

  // ── Per-step validation derivation (§8a) ────────────────────────────────────
  //
  // The composable derives done/error/pending/optional from validateStep() results.
  // validateStep() is mocked; these tests cover the composable's mapping logic only.
  // Real field rules land in F7.

  describe('stepStatus', () => {
    it('unvisited step with invalid result is pending (not error)', () => {
      const { stepStatus } = useAimOnboarding()
      // validateStep stub returns valid: false — but step 1 not visited
      expect(stepStatus.value[1]).toBe('pending')
    })

    it('visited step with invalid result is error', () => {
      const { visitedSteps, stepStatus } = useAimOnboarding()
      visitedSteps.value.add(1)
      // validateStep stub returns valid: false → error
      expect(stepStatus.value[1]).toBe('error')
    })

    it('visited step with valid result is done', () => {
      const validateStep = vi.mocked(
        (await import('../../logic/validation')).validateStep,
      )
      validateStep.mockReturnValueOnce({ valid: true, invalidFields: [] })

      const { visitedSteps, stepStatus } = useAimOnboarding()
      visitedSteps.value.add(1)
      // stepStatus re-evaluates after mock change — trigger by reading
      expect(stepStatus.value[1]).toBe('done')
    })

    it('optional steps (6, 7) are always optional regardless of visited/field state', () => {
      const { visitedSteps, stepStatus } = useAimOnboarding()
      visitedSteps.value.add(6)
      visitedSteps.value.add(7)
      expect(stepStatus.value[6]).toBe('optional')
      expect(stepStatus.value[7]).toBe('optional')
    })

    it('allRequiredDone is false when any required step is not done', () => {
      const { allRequiredDone } = useAimOnboarding()
      expect(allRequiredDone.value).toBe(false)
    })
  })

  // ── patchWizard ──────────────────────────────────────────────────────────────

  describe('patchWizard', () => {
    it('merges partial updates into wizard state', () => {
      const { wizard, patchWizard } = useAimOnboarding()
      patchWizard({ companyName: 'Kochava', platforms: ['iOS'] })
      expect(wizard.value.companyName).toBe('Kochava')
      expect(wizard.value.platforms).toEqual(['iOS'])
      expect(wizard.value.ua).toBe(false)
    })

    it('business model change clears LTV and webFunnel fields', () => {
      const { wizard, patchWizard } = useAimOnboarding()
      wizard.value.wantsLtv = true
      wizard.value.ltvCohortAvail = 'yes'
      wizard.value.ltvPartialChoice = 'partial'
      wizard.value.webFunnel = { someKey: 'someValue' }
      patchWizard({ business: 'gaming' })
      expect(wizard.value.wantsLtv).toBe(false)
      expect(wizard.value.ltvCohortAvail).toBe('')
      expect(wizard.value.ltvPartialChoice).toBe('')
      expect(wizard.value.webFunnel).toEqual({})
    })

    it('patch that does not change business does not clear LTV fields', () => {
      const { wizard, patchWizard } = useAimOnboarding()
      wizard.value.business = 'subscription'
      wizard.value.wantsLtv = true
      patchWizard({ companyName: 'Acme' })
      expect(wizard.value.wantsLtv).toBe(true)
    })
  })

  // ── reset ────────────────────────────────────────────────────────────────────

  describe('reset', () => {
    it('restores wizard and navigation to defaults', () => {
      const { wizard, currentStep, visitedSteps, patchWizard, goToStep, reset } = useAimOnboarding()
      patchWizard({ companyName: 'Acme' })
      goToStep(3)
      reset()
      expect(wizard.value.companyName).toBe('')
      expect(currentStep.value).toBe(1)
      expect(visitedSteps.value.size).toBe(0)
    })
  })
})
```

**1.2** Run (expect compile failure — composable does not exist yet):

```bash
cd packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding.spec.ts 2>&1 | tail -30
```

---

### Task 2 — Create the validation stub (F7 placeholder)

**2.1** If `AimOnboardingTab/logic/validation.ts` does not yet exist, create it:

```ts
// logic/validation.ts — stub; replaced by full field-rule implementation in F7.

export interface ValidationResult {
  valid: boolean
  /** Keys of active required fields that are empty/invalid. */
  invalidFields: string[]
}

/**
 * Returns a per-step validation result for the given wizard state.
 * Conditional (dimmed) fields are excluded from invalidFields when their parent is unset.
 * Stub — F7 supplies the real rules.
 */
// eslint-disable-next-line @typescript-eslint/no-unused-vars
export function validateStep(_stepKey: string, _wizard: unknown): ValidationResult {
  return { valid: false, invalidFields: [] }
}
```

---

### Task 3 — Create the composable

**3.1** Create:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts
```

Contents:

```ts
import { ref, computed } from 'vue'
import type { WizardState } from '../interfaces/aimOnboarding'
import { validateStep } from '../logic/validation'

// ── Types ────────────────────────────────────────────────────────────────────

/**
 * Step-level statuses per §8a:
 *   done     = every active required field is valid
 *   error    = visited + at least one active required field invalid
 *   pending  = not yet visited and not yet done
 *   optional = steps 6 + 7 — never block, never count toward required-completeness
 *
 * "dimmed" is a field-level concept handled by validateStep() in F7 — it excludes
 * conditional fields from invalidFields when their parent condition is unset.
 */
export type StepStatus = 'done' | 'error' | 'pending' | 'optional'

/**
 * Numeric budget split across campaign types.
 * Separate from uaShare/ueShare string fields; used only for routing-flag computation.
 * Must be added to WizardState in F2 so F4 autosave persists it.
 */
export interface CampaignShare {
  ua: number
  ue: number
  brand: number
}

// ── Constants ────────────────────────────────────────────────────────────────

const TOTAL_STEPS = 8
const OPTIONAL_STEPS = new Set([6, 7])

// ── Default state ────────────────────────────────────────────────────────────

function defaultWizard(): WizardState {
  return {
    companyName: '',
    projectLeads: [],
    dataLeads: [],
    appName: '',
    platforms: [],
    platformShares: {},
    modelling: '',
    region: '',
    regionOther: '',
    budgetMonthly: 0,
    budgetAnnual: 0,
    budgetPeriod: 'monthly',
    usesOffline: false,
    offlineSplitPct: 20,
    digitalMediaTypes: [],
    hasAttrGaps: '',
    attrGapCategories: [],
    coverageConfidence: '',
    paidSplit: { iOS: 65, Android: 80, Web: 50 },
    ua: false,
    uaShare: '',
    ue: false,
    ueShare: '',
    brand: false,
    brandShare: '',
    campaignGrouping: false,
    campaignGroupingChoice: '',
    campaignShare: { ua: 0, ue: 0, brand: 0 },
    business: '',
    funnel: { items: [], kpi: '', kpiConfirmed: false, names: {} },
    webFunnel: {},
    wantsLtv: false,
    ltvCohortAvail: '',
    ltvPartialChoice: '',
    mmp: '',
    mmpCollection: '',
    mmpFileStorage: '',
    appsflyerCohortAccess: false,
    webAttrSources: [],
    spendCollection: '',
    adSpendSources: [],
    history: {},
    externalFactors: [],
    externalFactorsNotes: '',
    objectivesGoal: '',
    objectivesSuccess: '',
    objectivesMarketing: '',
    uaUeRoutingFlag: '',
    brandRoutingFlag: false,
  }
}

// ── Module-scope singletons (shared across all callers, mirrors useValidateOnboarding pattern) ──

const wizard = ref<WizardState>(defaultWizard())
const currentStep = ref(1)
const visitedSteps = ref<Set<number>>(new Set())

// ── Composable ───────────────────────────────────────────────────────────────

export function useAimOnboarding() {

  // ── Routing flags ────────────────────────────────────────────────────────────

  /**
   * uaUeRoutingFlag (computed, §8a — stored into wizard.uaUeRoutingFlag for persistence):
   *   ''                  neither UA nor UE selected
   *   'aim_x_preferred'   only one of UA/UE selected, OR both selected but either share
   *                       is outside 10–90%
   *   'aim_pro_eligible'  both UA+UE selected AND ua ∈ [10,90] AND ue ∈ [10,90]
   */
  const uaUeRoutingFlag = computed<string>(() => {
    const { ua, ue, campaignShare: cs } = wizard.value
    if (!ua && !ue) return ''
    if (ua && ue) {
      const u = cs?.ua ?? 0
      const e = cs?.ue ?? 0
      return u >= 10 && u <= 90 && e >= 10 && e <= 90
        ? 'aim_pro_eligible'
        : 'aim_x_preferred'
    }
    return 'aim_x_preferred'
  })

  /** brandRoutingFlag — true when brand campaign type is selected. */
  const brandRoutingFlag = computed<boolean>(() => !!wizard.value.brand)

  // ── Per-step validation derivation (§8a) ─────────────────────────────────────
  //
  // Delegates field-level rules to validateStep() (F7).
  // validateStep() handles dimmed (conditional) field exclusion.
  // This composable maps the result → step-level StepStatus.

  const stepStatus = computed<Record<number, StepStatus>>(() => {
    const result: Record<number, StepStatus> = {}
    for (let step = 1; step <= TOTAL_STEPS; step++) {
      if (OPTIONAL_STEPS.has(step)) {
        result[step] = 'optional'
        continue
      }
      const { valid } = validateStep(String(step), wizard.value)
      if (valid) {
        result[step] = 'done'
      } else if (visitedSteps.value.has(step)) {
        result[step] = 'error'
      } else {
        result[step] = 'pending'
      }
    }
    return result
  })

  /** True when all required steps (1–5, 8) are done. Gates output "Approve". */
  const allRequiredDone = computed<boolean>(() => {
    for (let step = 1; step <= TOTAL_STEPS; step++) {
      if (OPTIONAL_STEPS.has(step)) continue
      if (stepStatus.value[step] !== 'done') return false
    }
    return true
  })

  // ── Navigation ───────────────────────────────────────────────────────────────

  function goToStep(step: number): void {
    if (step < 1 || step > TOTAL_STEPS) return
    visitedSteps.value.add(currentStep.value)
    currentStep.value = step
  }

  function nextStep(): void {
    if (currentStep.value < TOTAL_STEPS) {
      visitedSteps.value.add(currentStep.value)
      currentStep.value++
    }
  }

  function prevStep(): void {
    if (currentStep.value > 1) {
      currentStep.value--
    }
  }

  // ── patchWizard ───────────────────────────────────────────────────────────────
  //
  // Merges partial updates. Applies cross-step resets per §4.1:
  // business model change clears LTV + webFunnel.

  function patchWizard(patch: Partial<WizardState>): void {
    const isBusinessChange =
      'business' in patch && patch.business !== wizard.value.business

    Object.assign(wizard.value, patch)

    if (isBusinessChange) {
      wizard.value.wantsLtv = false
      wizard.value.ltvCohortAvail = ''
      wizard.value.ltvPartialChoice = ''
      wizard.value.webFunnel = {}
    }
  }

  // ── reset ─────────────────────────────────────────────────────────────────────

  function reset(): void {
    wizard.value = defaultWizard()
    currentStep.value = 1
    visitedSteps.value = new Set()
  }

  return {
    // State (module-scope singletons — shared)
    wizard,
    currentStep,
    visitedSteps,
    // Derived (from wizard — exposed for direct binding in StepRail/Step components)
    campaignShare: computed({
      get: () => wizard.value.campaignShare ?? { ua: 0, ue: 0, brand: 0 },
      set: (v) => { wizard.value.campaignShare = v },
    }),
    // Computed flags
    uaUeRoutingFlag,
    brandRoutingFlag,
    // Computed status
    stepStatus,
    allRequiredDone,
    // Navigation
    goToStep,
    nextStep,
    prevStep,
    // Mutation
    patchWizard,
    reset,
  }
}
```

---

### Task 4 — Run tests (expect pass)

**4.1** Run the composable spec:

```bash
cd packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding.spec.ts 2>&1 | tail -30
```

All tests must pass.

**4.2** Run the full advertiser test suite to confirm no regressions:

```bash
cd packages/advertiser
npx vitest run 2>&1 | tail -20
```

**4.3** Type-check:

```bash
cd packages/advertiser
npx tsc --noEmit 2>&1 | tail -20
```

---

### Task 5 — Lint

**5.1** Run the linter from the monorepo root:

```bash
cd /path/to/frontend-mos
pnpm --filter @mos/advertiser lint 2>&1 | tail -20
```

Fix any reported issues before committing.

---

### Task 6 — Commit

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding.spec.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/validation.ts

git commit -m "feat(onboarding): add useAimOnboarding composable — state, routing flags, nav, §8a validation (F3)"
```
