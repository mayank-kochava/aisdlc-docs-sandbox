---
id: plan-f5
title: "F5 — recommendTier Logic"
---

## F5 — recommendTier Logic: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement a pure-TypeScript `recommendTier(state)` function — the AIM X vs AIM Pro tier algorithm — with `effectiveMonthlyBudget()` as a testable helper, baking in the three `recommendTier`-owned prototype bug-fixes, fully unit-tested with Vitest covering all §4.3 Gherkin scenarios and the bug-fix regression cases.

**Architecture:** Framework-agnostic pure function in `logic/recommendTier.ts` alongside its `__tests__/recommendTier.spec.ts`. No Vue, no composable, no side effects. Consumes the `AimWizardState` type from `interfaces/aimOnboarding.ts` (provided by plan F2); if F2 is not yet merged, stub the minimum interface locally and update the import once F2 lands. Exports two symbols: `effectiveMonthlyBudget` and `recommendTier`.

**Tech Stack:** TypeScript 5, Vitest 4.1.2 (project-wide runner). Test file follows the `__tests__/*.spec.ts` convention established in `ValidateOnboardingTab/composables/__tests__/useValidateOnboarding.spec.ts`.

---

## Bug-fix scope for this file

Of the eight prototype bugs listed in product-spec-v2 §9, **three are owned by `recommendTier.ts`**. The other five live in other plans:

| Bug | Owner |
|-----|-------|
| #1 — reads `state.spend` instead of `budgetMonthly`/`budgetAnnual` | **This plan (F5)** |
| #2 — history threshold 24 months (must be 12 = any value set) | **This plan (F5)** |
| #3 — `paidSplit` defaults 50/50 | Step 3 wizard component (W3) |
| #4 — seasonality always "Included" | Step 6 component (W6) |
| #5 — tier always returns AIM X (broken algorithm) | **This plan (F5)** |
| #6 — objectives not in SoW | SoW output (O1) |
| #7 — ad channels not in output | SoW output (O1) |
| #8 — `campaignGrouping` had no UI | Step 3 wizard component (W3) |

---

## Files

| Action | Path |
|--------|------|
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/recommendTier.ts` |
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts` |
| **Read (do not modify)** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts` *(F2 output — import `AimWizardState` from here)* |

---

## Dependencies

- **F2** — `interfaces/aimOnboarding.ts` must define `AimWizardState` before this plan's import compiles. If F2 is not yet merged, declare a local `MinimalWizardState` stub in the spec file (see Task 1, step 1) and replace with the real import once F2 lands.

---

## State fields consumed (from design-consolidated §5)

`recommendTier` reads only these fields from `AimWizardState`:

```ts
budgetMonthly: number | null
budgetAnnual: number | null
budgetPeriod: 'monthly' | 'annual'
history: Record<string, '12_24' | '24_36' | '36_plus'>  // keyed by platform, e.g. {iOS:'12_24',Android:'24_36'}
webAttrSources: string[]
adSpendSources: string[]
uaUeRoutingFlag: 'aim_pro_eligible' | 'aim_x_preferred' | ''
```

**Key contract differences from the prototype (do NOT copy the prototype):**

| Item | Prototype (`mockup-k4a.html`) | Correct contract |
|------|-------------------------------|-----------------|
| Budget source | `s.budgetMonthly` only (ignores annual) | `budgetMonthly` **or** `budgetAnnual ÷ 12` depending on `budgetPeriod` |
| History type | Scalar string `'12-24'`/`'24-36'` | `Record<platform, '12_24'\|'24_36'\|'36_plus'>` (object, underscores) |
| History threshold | Excludes `'12-24'` (≥24mo) | All three values qualify (`'12_24'` included — CG-007) |
| Return values | `'Pro'` / `'X'` | `'aim_pro'` / `'aim_x'` (design-consolidated §5 `tier.recommended`) |
| Override logic | In same function | **Not in this file** — pure recommendation only |

---

## Task 1: Write failing tests for `effectiveMonthlyBudget`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts`

- [ ] **Step 1: Create the spec file with the `effectiveMonthlyBudget` tests**

```ts
// packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts

import { describe, it, expect } from 'vitest'
import { effectiveMonthlyBudget, recommendTier } from '../recommendTier'

// Minimal state stub — replace with `import type { AimWizardState } from '../../interfaces/aimOnboarding'`
// once plan F2 is merged.
type HistoryVal = '12_24' | '24_36' | '36_plus'
interface WizardStub {
  budgetMonthly: number | null
  budgetAnnual: number | null
  budgetPeriod: 'monthly' | 'annual'
  history: Record<string, HistoryVal>
  webAttrSources: string[]
  adSpendSources: string[]
  uaUeRoutingFlag: 'aim_pro_eligible' | 'aim_x_preferred' | ''
}

function makeState(overrides: Partial<WizardStub> = {}): WizardStub {
  return {
    budgetMonthly: null,
    budgetAnnual: null,
    budgetPeriod: 'monthly',
    history: {},
    webAttrSources: [],
    adSpendSources: [],
    uaUeRoutingFlag: '',
    ...overrides,
  }
}

// ─── effectiveMonthlyBudget ───────────────────────────────────────────────────

describe('effectiveMonthlyBudget', () => {
  it('returns budgetMonthly when period is monthly', () => {
    expect(effectiveMonthlyBudget(makeState({ budgetMonthly: 300_000, budgetPeriod: 'monthly' }))).toBe(300_000)
  })

  it('divides budgetAnnual by 12 and rounds when period is annual', () => {
    // §4.3 Gherkin: $3,000,000 annual → $250,000/month
    expect(effectiveMonthlyBudget(makeState({ budgetAnnual: 3_000_000, budgetPeriod: 'annual' }))).toBe(250_000)
  })

  it('rounds fractional annual division to nearest integer', () => {
    // $2,500,001 / 12 = 208333.416... → 208333
    expect(effectiveMonthlyBudget(makeState({ budgetAnnual: 2_500_001, budgetPeriod: 'annual' }))).toBe(208_333)
  })

  it('returns 0 when no budget is set', () => {
    expect(effectiveMonthlyBudget(makeState())).toBe(0)
  })

  // Bug-fix #1 regression: prototype read state.spend — verify budgetMonthly is used
  it('bug-fix #1: reads budgetMonthly, not a hypothetical state.spend field', () => {
    const state = makeState({ budgetMonthly: 500_000, budgetPeriod: 'monthly' })
    // TypeScript type safety enforces this at compile time; this test documents the intent
    expect(effectiveMonthlyBudget(state)).toBe(500_000)
  })
})
```

- [ ] **Step 2: Run the tests — expect FAIL (function not found)**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts
```

Expected output contains: `Cannot find module '../recommendTier'`

---

## Task 2: Write failing tests for `recommendTier` — all three Gherkin scenarios + bug-fix regressions

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts`

- [ ] **Step 1: Append the `recommendTier` describe block to the spec file**

```ts
// ─── recommendTier ───────────────────────────────────────────────────────────

describe('recommendTier', () => {
  // ── Condition 1: Spend + history ──────────────────────────────────────────

  describe('condition 1 — spend + history', () => {
    // §4.3 Gherkin: "Annual budget correctly converts for tier check"
    it('Gherkin: $3M annual / 24+ months history → aim_pro', () => {
      const state = makeState({
        budgetAnnual: 3_000_000,
        budgetPeriod: 'annual',
        history: { iOS: '24_36', Android: '24_36' },
      })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    // §4.3 Gherkin: "12-24 months history qualifies for spend path"
    it('Gherkin: 12-24 months history + budget >= 250K → aim_pro', () => {
      // CG-007 fix: prototype excluded '12_24' — spec says it qualifies
      const state = makeState({
        budgetMonthly: 250_000,
        budgetPeriod: 'monthly',
        history: { iOS: '12_24' },
      })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    // §4.3 Gherkin: "Annual budget below threshold"
    it('Gherkin: $2.4M annual (= $200K/month) → aim_x', () => {
      const state = makeState({
        budgetAnnual: 2_400_000,
        budgetPeriod: 'annual',
        history: { iOS: '36_plus' },
      })
      expect(recommendTier(state)).toBe('aim_x')
    })

    it('budget >= 250K with 36+ months history → aim_pro', () => {
      const state = makeState({
        budgetMonthly: 500_000,
        budgetPeriod: 'monthly',
        history: { iOS: '36_plus', Android: '36_plus' },
      })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    it('budget exactly at 250K threshold → aim_pro', () => {
      const state = makeState({
        budgetMonthly: 250_000,
        budgetPeriod: 'monthly',
        history: { iOS: '24_36' },
      })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    it('budget 249_999 (below threshold) → aim_x even with history set', () => {
      const state = makeState({
        budgetMonthly: 249_999,
        budgetPeriod: 'monthly',
        history: { iOS: '36_plus' },
      })
      expect(recommendTier(state)).toBe('aim_x')
    })

    // Bug-fix #2 regression: prototype required 24mo minimum, excluded 12_24
    it('bug-fix #2: history 12_24 qualifies (prototype excluded it)', () => {
      const state = makeState({
        budgetMonthly: 300_000,
        budgetPeriod: 'monthly',
        history: { Android: '12_24' },
      })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    it('empty history object → condition1 false even with high budget', () => {
      const state = makeState({
        budgetMonthly: 500_000,
        budgetPeriod: 'monthly',
        history: {},
      })
      // history not yet set — do not recommend Pro from condition1 alone
      expect(recommendTier(state)).toBe('aim_x')
    })

    it('one platform has history, one does not → condition1 true (any platform qualifies)', () => {
      const state = makeState({
        budgetMonthly: 300_000,
        budgetPeriod: 'monthly',
        history: { iOS: '24_36' },  // Android not yet set
      })
      expect(recommendTier(state)).toBe('aim_pro')
    })
  })

  // ── Condition 2: Additional non-MMP data sources ─────────────────────────

  describe('condition 2 — >= 2 additional non-MMP data sources', () => {
    it('2 webAttrSources → aim_pro', () => {
      const state = makeState({ webAttrSources: ['ga4', 'adobe_analytics'] })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    it('2 non-mmp_direct adSpendSources → aim_pro', () => {
      const state = makeState({ adSpendSources: ['supermetrics', 'rockerbox'] })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    it('1 webAttrSource + 1 non-mmp_direct adSpendSource → aim_pro', () => {
      const state = makeState({
        webAttrSources: ['ga4'],
        adSpendSources: ['supermetrics'],
      })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    it('mmp_direct in adSpendSources does NOT count toward condition 2', () => {
      const state = makeState({
        webAttrSources: [],
        adSpendSources: ['mmp_direct'],
      })
      expect(recommendTier(state)).toBe('aim_x')
    })

    it('1 webAttrSource + mmp_direct only → aim_x (total non-MMP = 1)', () => {
      const state = makeState({
        webAttrSources: ['ga4'],
        adSpendSources: ['mmp_direct'],
      })
      expect(recommendTier(state)).toBe('aim_x')
    })

    it('exactly 1 non-MMP source total → aim_x', () => {
      const state = makeState({ webAttrSources: ['ga4'], adSpendSources: [] })
      expect(recommendTier(state)).toBe('aim_x')
    })
  })

  // ── Condition 3: UA/UE routing flag ───────────────────────────────────────

  describe('condition 3 — uaUeRoutingFlag', () => {
    it('aim_pro_eligible → aim_pro', () => {
      const state = makeState({ uaUeRoutingFlag: 'aim_pro_eligible' })
      expect(recommendTier(state)).toBe('aim_pro')
    })

    it('aim_x_preferred → aim_x', () => {
      const state = makeState({ uaUeRoutingFlag: 'aim_x_preferred' })
      expect(recommendTier(state)).toBe('aim_x')
    })

    it('empty uaUeRoutingFlag → aim_x', () => {
      const state = makeState({ uaUeRoutingFlag: '' })
      expect(recommendTier(state)).toBe('aim_x')
    })
  })

  // ── Default / no conditions ───────────────────────────────────────────────

  describe('default', () => {
    // Bug-fix #5 regression: prototype always returned 'X'
    it('bug-fix #5: returns aim_x by default (not always aim_x — conditions must fire)', () => {
      // All conditions false → X (correct)
      expect(recommendTier(makeState())).toBe('aim_x')
    })

    it('any single condition true upgrades to aim_pro regardless of others', () => {
      // Only condition 3
      expect(recommendTier(makeState({ uaUeRoutingFlag: 'aim_pro_eligible' }))).toBe('aim_pro')
    })
  })

  // ── Return type contract ───────────────────────────────────────────────────

  describe('return type', () => {
    it('returns "aim_pro" not "Pro" (design-consolidated §5 contract)', () => {
      const result = recommendTier(makeState({ uaUeRoutingFlag: 'aim_pro_eligible' }))
      expect(result).toBe('aim_pro')
      expect(result).not.toBe('Pro')
    })

    it('returns "aim_x" not "X" (design-consolidated §5 contract)', () => {
      const result = recommendTier(makeState())
      expect(result).toBe('aim_x')
      expect(result).not.toBe('X')
    })
  })
})
```

- [ ] **Step 2: Run the tests — expect FAIL (function not found, or all tests fail)**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts
```

Expected: all tests fail with module resolution error.

---

## Task 3: Implement `recommendTier.ts`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/recommendTier.ts`

- [ ] **Step 1: Create the implementation file**

```ts
// packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/recommendTier.ts

/**
 * Pure-function tier recommendation for the AIM Onboarding wizard.
 *
 * Returns 'aim_pro' or 'aim_x'. No side-effects, no Vue imports.
 * Override logic (CSM asymmetric) is NOT in this file — see X1 / B6.
 *
 * Spec: product-spec-v2.md §4.3
 * Bug-fixes baked in: #1 (budget source), #2 (history threshold 12mo), #5 (always AIM X)
 */

// ── Types ────────────────────────────────────────────────────────────────────

/** Values stored in wizard.history (per-platform). All three qualify for condition 1. */
type HistoryValue = '12_24' | '24_36' | '36_plus'

/**
 * Minimal shape consumed by recommendTier.
 * Import AimWizardState from '../interfaces/aimOnboarding' once plan F2 is merged,
 * and use it here instead.
 */
export interface TierInputState {
  /** Monthly budget in dollars (set when budgetPeriod === 'monthly'). */
  budgetMonthly: number | null
  /** Annual budget in dollars (set when budgetPeriod === 'annual'). */
  budgetAnnual: number | null
  /** Which budget field is active. */
  budgetPeriod: 'monthly' | 'annual'
  /**
   * History per platform — keys are platform names ('iOS', 'Android', 'Web'),
   * values are one of the three enum strings. Empty object = not yet answered.
   * (design-consolidated §5 — not a scalar; all three values qualify, CG-007.)
   */
  history: Record<string, HistoryValue>
  /** Web attribution sources selected by the client. */
  webAttrSources: string[]
  /** Ad spend sources. 'mmp_direct' does NOT count toward condition 2. */
  adSpendSources: string[]
  /** Computed by calcFlags() in useAimOnboarding composable (plan F3). */
  uaUeRoutingFlag: 'aim_pro_eligible' | 'aim_x_preferred' | ''
}

export type Tier = 'aim_x' | 'aim_pro'

// ── Helpers ───────────────────────────────────────────────────────────────────

/**
 * Converts the wizard's budget answer to a monthly dollar amount.
 *
 * Bug-fix #1: prototype read `state.budgetMonthly` only — this function reads
 * `budgetAnnual` when `budgetPeriod === 'annual'` and divides by 12.
 * §4.3: "annual budget divided by 12 before comparison" (CG-008).
 */
export function effectiveMonthlyBudget(state: TierInputState): number {
  if (state.budgetPeriod === 'annual') {
    return Math.round((state.budgetAnnual ?? 0) / 12)
  }
  return state.budgetMonthly ?? 0
}

// ── Algorithm ─────────────────────────────────────────────────────────────────

/**
 * Recommends a tier based on three conditions. Starts at AIM X; upgrades to
 * AIM Pro if any condition is met. Returns the design-consolidated §5 enum
 * strings ('aim_x' | 'aim_pro') — NOT the prototype's 'X'/'Pro'.
 *
 * Condition 1 — Spend + history:
 *   effectiveMonthlyBudget >= $250K AND at least one platform has history set.
 *   CG-007: all three history values ('12_24', '24_36', '36_plus') qualify.
 *   Bug-fix #2: prototype required 24mo minimum; correct threshold is 12 months.
 *
 * Condition 2 — Additional non-MMP data sources:
 *   >= 2 sources across webAttrSources + adSpendSources (excluding 'mmp_direct').
 *
 * Condition 3 — UA/UE routing flag:
 *   uaUeRoutingFlag === 'aim_pro_eligible'.
 *
 * Bug-fix #5: prototype always returned 'X' due to broken conditions; this
 *   function returns 'aim_pro' whenever any condition is true.
 */
export function recommendTier(state: TierInputState): Tier {
  // Condition 1 — spend + history
  const monthly = effectiveMonthlyBudget(state)
  const historyValues = Object.values(state.history)
  const historyIsSet = historyValues.length > 0 && historyValues.some(Boolean)
  const condition1 = monthly >= 250_000 && historyIsSet

  // Condition 2 — >= 2 additional non-MMP data sources
  const nonMmpCount =
    (state.webAttrSources ?? []).length +
    (state.adSpendSources ?? []).filter((s) => s !== 'mmp_direct').length
  const condition2 = nonMmpCount >= 2

  // Condition 3 — UA/UE routing flag
  const condition3 = state.uaUeRoutingFlag === 'aim_pro_eligible'

  return condition1 || condition2 || condition3 ? 'aim_pro' : 'aim_x'
}
```

- [ ] **Step 2: Run the tests — expect all PASS**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts
```

Expected output (all 25 tests pass):

```text
✓ effectiveMonthlyBudget > returns budgetMonthly when period is monthly
✓ effectiveMonthlyBudget > divides budgetAnnual by 12 and rounds when period is annual
✓ effectiveMonthlyBudget > rounds fractional annual division to nearest integer
✓ effectiveMonthlyBudget > returns 0 when no budget is set
✓ effectiveMonthlyBudget > bug-fix #1: reads budgetMonthly, not a hypothetical state.spend field
✓ recommendTier > condition 1 > Gherkin: $3M annual / 24+ months history → aim_pro
✓ recommendTier > condition 1 > Gherkin: 12-24 months history + budget >= 250K → aim_pro
✓ recommendTier > condition 1 > Gherkin: $2.4M annual (= $200K/month) → aim_x
✓ recommendTier > condition 1 > budget >= 250K with 36+ months history → aim_pro
✓ recommendTier > condition 1 > budget exactly at 250K threshold → aim_pro
✓ recommendTier > condition 1 > budget 249_999 (below threshold) → aim_x even with history set
✓ recommendTier > condition 1 > bug-fix #2: history 12_24 qualifies (prototype excluded it)
✓ recommendTier > condition 1 > empty history object → condition1 false even with high budget
✓ recommendTier > condition 1 > one platform has history, one does not → condition1 true
✓ recommendTier > condition 2 > 2 webAttrSources → aim_pro
✓ recommendTier > condition 2 > 2 non-mmp_direct adSpendSources → aim_pro
✓ recommendTier > condition 2 > 1 webAttrSource + 1 non-mmp_direct adSpendSource → aim_pro
✓ recommendTier > condition 2 > mmp_direct in adSpendSources does NOT count toward condition 2
✓ recommendTier > condition 2 > 1 webAttrSource + mmp_direct only → aim_x (total non-MMP = 1)
✓ recommendTier > condition 2 > exactly 1 non-MMP source total → aim_x
✓ recommendTier > condition 3 > aim_pro_eligible → aim_pro
✓ recommendTier > condition 3 > aim_x_preferred → aim_x
✓ recommendTier > condition 3 > empty uaUeRoutingFlag → aim_x
✓ recommendTier > default > bug-fix #5: returns aim_x by default (not always aim_x)
✓ recommendTier > default > any single condition true upgrades to aim_pro
✓ recommendTier > return type > returns "aim_pro" not "Pro" (design-consolidated §5 contract)
✓ recommendTier > return type > returns "aim_x" not "X" (design-consolidated §5 contract)

Test Files  1 passed (1)
Tests       27 passed (27)
```

- [ ] **Step 3: Run the full test suite to confirm no regressions**

```bash
cd packages/advertiser && npm run test:ci
```

Expected: all previously-passing tests still pass.

- [ ] **Step 4: Commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/recommendTier.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts
git commit -m "feat(aim-onboarding): add recommendTier pure function with bug-fixes and Vitest coverage

- effectiveMonthlyBudget converts annual÷12 (bug-fix #1)
- all history values qualify incl. 12_24 (bug-fix #2 / CG-007)
- three-condition algorithm fires correctly (bug-fix #5)
- returns aim_x / aim_pro per design-consolidated §5 contract
- 27 Vitest tests covering all §4.3 Gherkin scenarios and regression cases"
```

---

## Task 4: Wire F2 import (do after F2 merges)

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/recommendTier.ts`

- [ ] **Step 1: Replace the local stub type with the F2 import**

In `recommendTier.ts`, replace the `TierInputState` interface declaration with an import from F2:

```ts
// Remove this block:
// export interface TierInputState { ... }

// Add at the top of the file (adjust path if F2 uses a barrel export):
import type { AimWizardState } from '../interfaces/aimOnboarding'

// Change the function signatures from TierInputState to AimWizardState:
export function effectiveMonthlyBudget(state: AimWizardState): number { ... }
export function recommendTier(state: AimWizardState): Tier { ... }
```

> Only the type annotation changes. All logic stays identical. The `Tier` export and `TierInputState` doc comment can be retained as JSDoc — or deleted if `AimWizardState` already exports a `Tier` type.

- [ ] **Step 2: Run tests to confirm nothing broke**

```bash
cd packages/advertiser && npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts
```

Expected: same 27 tests pass.

- [ ] **Step 3: Update the spec import if needed**

If `AimWizardState` has different optional-ness on budget/history fields, update `makeState()` defaults to match. The test logic does not change.

- [ ] **Step 4: Run full suite and commit**

```bash
cd packages/advertiser && npm run test:ci
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/recommendTier.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/logic/__tests__/recommendTier.spec.ts
git commit -m "refactor(aim-onboarding): wire recommendTier to AimWizardState from F2 interfaces"
```

---

## Self-review checklist

**Spec coverage:**

| §4.3 requirement | Covered by |
|-----------------|-----------|
| `effectiveMonthlyBudget = budgetAnnual / 12` (CG-008) | Task 1 tests + implementation |
| `'12_24'` included in qualifying history values (CG-007) | Task 2 Gherkin test + bug-fix #2 regression |
| Condition 1: `monthly >= 250K && historyIsSet` | Task 2 + implementation |
| Condition 2: `>= 2 non-MMP sources` | Task 2 + implementation |
| Condition 3: `uaUeRoutingFlag === 'aim_pro_eligible'` | Task 2 + implementation |
| Default = `aim_x` | Task 2 default tests |
| Return `aim_x`/`aim_pro` (not `'X'`/`'Pro'`) | Task 2 return-type tests |
| Bug-fix #1 (reads `budgetMonthly`/`budgetAnnual`) | Task 1 bug-fix test |
| Bug-fix #2 (12mo threshold) | Task 2 bug-fix #2 regression |
| Bug-fix #5 (always AIM X) | Task 2 default + conditions tests |
| Tier override / audit logging | **Out of scope** — see X1 (CSM UI) + B6 (audit endpoint) |
| Gherkin: override is logged | **Out of scope** — `recommendTier` is pure; no side-effects |

**Not in scope for this file:** bugs #3 (paidSplit defaults), #4 (seasonality display), #6 (objectives in SoW), #7 (channels in output), #8 (campaignGrouping UI). Each has its own plan.

**Placeholder scan:** None found.

**Type consistency:** `TierInputState` used consistently in Tasks 1–3. Replaced with `AimWizardState` in Task 4. `Tier = 'aim_x' | 'aim_pro'` consistent throughout. `effectiveMonthlyBudget` called by name in tests and implementation — no rename.
