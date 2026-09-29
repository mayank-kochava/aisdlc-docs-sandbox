# Task Packet: TG1 — Run tests & validate build

**Matrix:** `wi_11b559ca94dbff521ec5-r1` · **Repo:** `Kochava/frontend-mos` · **Stack:** `unknown` · **Owner:** `test-change-gate` (confidence 0.90)

## Objective

Run tests & validate build

## Dependencies

- Depends on `T103` — Register the third tab (`aimOnboarding` v-tab and v-window-item, i18n key `mmm_config.tab…
- Depends on `T107` — Add Vitest unit tests for the three logic modules (annual to monthly, 12 to 24 month qual…
- Depends on `T124` — Build shared `ToggleCard` (checkbox affordance for multi-select), `ShareAllocator` (slide…
- Depends on `T110` — Add Vitest tests for autosave (debounce, failure retry, load on mount, 409) with a mocked…
- Depends on `T111` — Complete the `AimOnboardingTab` wrapper: status routing from the latest session (wizard,…
- Depends on `T123` — Add component tests for output tabs (SoW render, schema rows, CSM gating, book-meeting re…

## Allowed Files

Touch only these paths. Anything else is out of scope for this task.

- (none captured from source -- verify against the plan before editing)

## Context

**Technology (frontend-mos):** Vue 3.5 / Vuetify 3.12, Pinia, Vitest + @vue/test-utils. Tab embed in MmmInsightsConfiguration.

From eng-plan:

> ### Kochava/frontend-mos
> **Technology**: Vue 3.5 / Vuetify 3.12, Pinia, Vitest
> **Key Files**:
> - `AimOnboardingTab/composables/useAimOnboarding.ts` - flat state, flags, step nav, autosave
> - `AimOnboardingTab/logic/{recommendTier,buildProvisionPlan,validation}.ts` - pure, unit-tested
> - `AimOnboardingTab/output/{SowTab,TimelineTab,DataSchemaTab,OutputScreen}.vue`
> **Changes**:
> | Change | Type | Complexity | Notes |
> |--------|------|------------|-------|
> | Types, `onboarding` service, interceptor, tab registration | New/Modify | L | Interceptor must add `x-user-id` for `/onboarding-sessions` |
> | Logic modules + Vitest (8 prototype bug-fixes) | New | M | annual/12, 12-month history threshold |
> | Composable + hybrid autosave | New | M | localStorage buffer + debounced PUT; overlap guard |
> | Shared ToggleCard / ShareAllocator / BudgetSlider | New | M | Built on `v-card` / `v-slider` |
> | 8 wizard steps + summary panel | New | H | Progressive disclosure; model change clears LTV/webFunnel |
> | SoW, PDF, Timeline, Data Schema tabs | New | H | PDF = print stylesheet; Data Schema from `buildProvisionPlan` |
> | CSM mode + book-meeting UI, output shell | New | M | Gate on `isASAdmin`; 403-aware |
>

## Acceptance Criteria

- [ ] Build and existing tests still pass.

## Definition of Done

- Tests pass for every file listed above under Test.
- No file outside Allowed Files is modified.
- Commit uses the conventional-commits format; PR title references `TG1`.

## Escalation

Stop and record an open question instead of guessing if: the files listed above do not match the current checkout, an interface this task consumes does not exist yet, or completing the task requires touching a file outside Allowed Files.
