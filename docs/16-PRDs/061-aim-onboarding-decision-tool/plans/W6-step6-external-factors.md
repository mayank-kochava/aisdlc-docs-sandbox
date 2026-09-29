---
id: plan-w6
title: "W6 — Step 6: External Factors"
---

## W6 — Step 6: External Factors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `steps/Step6Factors.vue` — the optional external-factors step that shows two auto-included factors (Seasonality and National Holidays), a 6-item multi-select grid for additional factors, and a free-text notes field that appears when any factor is selected.

**Architecture:** Presentational step component. Reads `wizard.History` to derive Seasonality status (`Partial` if any platform is `12_24`, otherwise `Included`). Binds to `patchWizard` from `useAimOnboarding`. `ToggleCard` (C1) drives the 6-factor grid. Never blocks navigation. `ExternalFactors` is a `string[]` of factor IDs; `ExternalFactorsNotes` is a plain string. No new composable or logic module required.

**Tech Stack:** Vue 3.5, Vuetify 3.12, Vitest + `@vue/test-utils`, frontend-mos monorepo.

---

## Files

| Action | Path | Purpose |
|--------|------|---------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step6Factors.vue` | The wizard step component |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step6Factors.spec.ts` | Vitest component tests |

---

## Dependencies

| ID | Plan | What is needed |
|----|------|----------------|
| **F2** | Service + Interfaces | `WizardState` type (`.ExternalFactors: string[]`, `.ExternalFactorsNotes: string`, `.History: Record<string, unknown>`) |
| **F3** | useAimOnboarding composable | `wizard`, `patchWizard`, `reset` |
| **C1** | ToggleCard | `ToggleCard.vue` — drives the 6-factor multi-select grid |

All three must be merged before implementing this step.

---

## Spec facts locked in for this plan

From `product-spec-v2.md §4.1 Step 6` and `design-consolidated.md §5`:

- Step is **optional** — no validation, never blocks "Next" or output approval.
  `stepStatus[6]` is always `'optional'` (enforced in F3, not here).
- Auto-included factors (non-interactive, informational only):
    - **Seasonality** — status derived from `wizard.History`:
        - `⚠ Partial` if **any** platform's history value is `'12_24'`
        - `✓ Included` otherwise (including when History is empty / no platforms selected)
    - **National Holidays** — always `✓ Included`; shows the `wizard.Region` value
      (falls back to `"your region"` if Region is empty)
- Additional factors — **exactly 6**, rendered as a `ToggleCard` grid:

  | ID | Label |
  |----|-------|
  | `promotional` | Promotional activity |
  | `competitor` | Competitor activity |
  | `major_launch` | Major product launch / rebrand |
  | `pr_spike` | PR / earned media spike |
  | `macro_economic` | Macro economic event |
  | `other` | Other |

  "Regulatory change" and "Natural disaster/crisis" are **not** separate factors — they were
  consolidated into `macro_economic` (spec §4.1 PR comment A4). Do not add them.

- Free-text notes — `v-textarea` rendered only when `ExternalFactors.length > 0`.
  Binds to `ExternalFactorsNotes`. Label: `"Describe the factors"`. Rows: 3.
- State keys (PascalCase wire contract, camelCase in composable):
    - `wizard.ExternalFactors` / `wizard.externalFactors` — `string[]`
    - `wizard.ExternalFactorsNotes` / `wizard.externalFactorsNotes` — `string`

---

## Task 1: Write the failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step6Factors.spec.ts`

Write the full test suite before any implementation. Running these tests must fail because the component does not exist yet.

- [ ] **Step 1: Create the `steps/__tests__` directory**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__
  ```

  Expected: no error (directory may already exist if other step tests are present).

- [ ] **Step 2: Write the spec file**

  Create this file:

  `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step6Factors.spec.ts`

  ```ts
  import { mount } from "@vue/test-utils";
  import { describe, it, expect, beforeEach, vi } from "vitest";
  import { setActivePinia, createPinia } from "pinia";

  // Mock the composable — we test the component's rendering logic in isolation.
  // Tests that exercise composable binding use a lightweight state object instead.
  const mockPatchWizard = vi.fn();
  const mockWizard = {
    value: {
      ExternalFactors: [] as string[],
      ExternalFactorsNotes: "",
      History: {} as Record<string, unknown>,
      Region: "",
    },
  };

  vi.mock(
    "../../composables/useAimOnboarding",
    () => ({
      useAimOnboarding: () => ({
        wizard: mockWizard,
        patchWizard: mockPatchWizard,
      }),
    }),
  );

  // ToggleCard is a real component — do NOT stub it.
  // Vuetify globals are provided by tests.config.ts.

  import Step6Factors from "../Step6Factors.vue";

  describe("Step6Factors.vue", () => {
    beforeEach(() => {
      setActivePinia(createPinia());
      mockWizard.value.ExternalFactors = [];
      mockWizard.value.ExternalFactorsNotes = "";
      mockWizard.value.History = {};
      mockWizard.value.Region = "";
      mockPatchWizard.mockClear();
    });

    // ── Rendering ─────────────────────────────────────────────────────────────

    it("renders the step heading", () => {
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("External Factors");
    });

    it("renders the Seasonality auto-included row", () => {
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("Seasonality");
    });

    it("renders the National Holidays auto-included row", () => {
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("National Holidays");
    });

    it("renders exactly 6 ToggleCard items for additional factors", () => {
      const wrapper = mount(Step6Factors);
      const cards = wrapper.findAllComponents({ name: "ToggleCard" });
      expect(cards).toHaveLength(6);
    });

    it("renders all 6 factor labels", () => {
      const wrapper = mount(Step6Factors);
      const text = wrapper.text();
      expect(text).toContain("Promotional activity");
      expect(text).toContain("Competitor activity");
      expect(text).toContain("Major product launch / rebrand");
      expect(text).toContain("PR / earned media spike");
      expect(text).toContain("Macro economic event");
      expect(text).toContain("Other");
    });

    // ── Seasonality status logic ──────────────────────────────────────────────

    it("shows Seasonality as Included when History is empty", () => {
      mockWizard.value.History = {};
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("Included");
      expect(wrapper.text()).not.toContain("Partial");
    });

    it("shows Seasonality as Included when all platforms have 24+ months history", () => {
      mockWizard.value.History = { iOS: "24_36", Android: "36_plus" };
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("Included");
      expect(wrapper.text()).not.toContain("Partial");
    });

    it("shows Seasonality as Partial when any platform has 12_24 history", () => {
      mockWizard.value.History = { iOS: "12_24", Android: "24_36" };
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("Partial");
    });

    it("shows Seasonality as Partial when only one platform and it has 12_24 history", () => {
      mockWizard.value.History = { iOS: "12_24" };
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("Partial");
    });

    // ── National Holidays region display ─────────────────────────────────────

    it('shows fallback region "your region" when Region is empty', () => {
      mockWizard.value.Region = "";
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("your region");
    });

    it("shows actual Region value when set", () => {
      mockWizard.value.Region = "US";
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("US");
    });

    // ── Notes field visibility ────────────────────────────────────────────────

    it("does not render the notes textarea when no factors selected", () => {
      mockWizard.value.ExternalFactors = [];
      const wrapper = mount(Step6Factors);
      expect(wrapper.find("textarea").exists()).toBe(false);
    });

    it("renders the notes textarea when at least one factor is selected", () => {
      mockWizard.value.ExternalFactors = ["promotional"];
      const wrapper = mount(Step6Factors);
      expect(wrapper.find("textarea").exists()).toBe(true);
    });

    it("notes textarea label is Describe the factors", () => {
      mockWizard.value.ExternalFactors = ["competitor"];
      const wrapper = mount(Step6Factors);
      expect(wrapper.text()).toContain("Describe the factors");
    });

    // ── patchWizard binding — factor selection ───────────────────────────────

    it("calls patchWizard to add a factor when an unselected ToggleCard is toggled", async () => {
      mockWizard.value.ExternalFactors = [];
      const wrapper = mount(Step6Factors);
      // Trigger the toggle event on the first ToggleCard (promotional)
      const cards = wrapper.findAllComponents({ name: "ToggleCard" });
      await cards[0].trigger("click");
      expect(mockPatchWizard).toHaveBeenCalledWith(
        expect.objectContaining({
          ExternalFactors: expect.arrayContaining(["promotional"]),
        }),
      );
    });

    it("calls patchWizard to remove a factor when a selected ToggleCard is toggled", async () => {
      mockWizard.value.ExternalFactors = ["promotional"];
      const wrapper = mount(Step6Factors);
      const cards = wrapper.findAllComponents({ name: "ToggleCard" });
      // First card (promotional) is selected — clicking should deselect
      await cards[0].trigger("click");
      expect(mockPatchWizard).toHaveBeenCalledWith(
        expect.objectContaining({
          ExternalFactors: expect.not.arrayContaining(["promotional"]),
        }),
      );
    });

    it("calls patchWizard with updated notes on textarea input", async () => {
      mockWizard.value.ExternalFactors = ["other"];
      mockWizard.value.ExternalFactorsNotes = "";
      const wrapper = mount(Step6Factors);
      const textarea = wrapper.find("textarea");
      await textarea.setValue("Annual sale in Q4");
      expect(mockPatchWizard).toHaveBeenCalledWith(
        expect.objectContaining({ ExternalFactorsNotes: "Annual sale in Q4" }),
      );
    });

    // ── Selected card state ───────────────────────────────────────────────────

    it("passes selected=true to a ToggleCard whose ID is in ExternalFactors", () => {
      mockWizard.value.ExternalFactors = ["competitor"];
      const wrapper = mount(Step6Factors);
      const cards = wrapper.findAllComponents({ name: "ToggleCard" });
      // Second card = competitor (index 1)
      expect(cards[1].props("selected")).toBe(true);
    });

    it("passes selected=false to a ToggleCard whose ID is not in ExternalFactors", () => {
      mockWizard.value.ExternalFactors = ["competitor"];
      const wrapper = mount(Step6Factors);
      const cards = wrapper.findAllComponents({ name: "ToggleCard" });
      // First card = promotional (index 0) — not selected
      expect(cards[0].props("selected")).toBe(false);
    });

    // ── Optional step — never blocks ─────────────────────────────────────────

    it("does not render any required-field error indicators", () => {
      const wrapper = mount(Step6Factors);
      // No v-alert with error color — optional step has no validation errors
      const alerts = wrapper.findAll(".v-alert");
      const errorAlerts = alerts.filter((a) =>
        a.attributes("class")?.includes("error") ||
        a.attributes("class")?.includes("alert-red"),
      );
      expect(errorAlerts).toHaveLength(0);
    });
  });
  ```

- [ ] **Step 3: Run the tests — confirm they fail**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step6Factors.spec.ts 2>&1 | tail -30
  ```

  Expected: all tests **fail** with "Cannot find module `../Step6Factors.vue`". This is the correct TDD red state.

---

## Task 2: Implement Step6Factors.vue

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step6Factors.vue`

- [ ] **Step 1: Create the `steps/` directory if it does not exist**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps
  ```

  Expected: no error.

- [ ] **Step 2: Write the component**

  Create the file at:

  `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step6Factors.vue`

  ```vue
  <template>
    <div class="step6-factors">
      <h2 class="text-h6 font-weight-semibold mb-1">External Factors</h2>
      <p class="text-body-2 mb-6" style="color: rgb(var(--v-theme-grey-2))">
        Tell us about any external events that may have affected your marketing
        performance. This step is optional.
      </p>

      <!-- Auto-included factors ─────────────────────────────────────────── -->
      <p class="text-body-2 font-weight-semibold mb-3">Auto-included</p>
      <div class="step6-factors__auto-row mb-2">
        <v-icon size="16" :color="seasonalityPartial ? 'warning' : 'success'">
          {{ seasonalityPartial ? "mdi-alert-circle-outline" : "mdi-check-circle-outline" }}
        </v-icon>
        <span class="text-body-2 ml-2">
          Seasonality —
          <span
            :style="{
              color: seasonalityPartial
                ? 'rgb(var(--v-theme-warning))'
                : 'rgb(var(--v-theme-success))',
            }"
          >
            {{ seasonalityPartial ? "Partial" : "Included" }}
          </span>
          <template v-if="seasonalityPartial">
            <span class="text-caption ml-1" style="color: rgb(var(--v-theme-grey-2))">
              (12–24 months of history detected — seasonality will be partially modelled)
            </span>
          </template>
        </span>
      </div>

      <div class="step6-factors__auto-row mb-6">
        <v-icon size="16" color="success">mdi-check-circle-outline</v-icon>
        <span class="text-body-2 ml-2">
          National Holidays —
          <span style="color: rgb(var(--v-theme-success))">Included</span>
          <span class="text-caption ml-1" style="color: rgb(var(--v-theme-grey-2))">
            ({{ regionLabel }})
          </span>
        </span>
      </div>

      <!-- Additional factors ─────────────────────────────────────────────── -->
      <p class="text-body-2 font-weight-semibold mb-3">
        Additional factors
        <span class="text-caption font-weight-regular ml-1" style="color: rgb(var(--v-theme-grey-2))">
          — optional, select any that apply
        </span>
      </p>

      <div class="step6-factors__grid mb-4">
        <toggle-card
          v-for="factor in FACTORS"
          :key="factor.id"
          :title="factor.label"
          :selected="selectedFactors.includes(factor.id)"
          @toggle="toggleFactor(factor.id)"
        />
      </div>

      <!-- Free-text notes — visible only when at least one factor selected -->
      <v-textarea
        v-if="selectedFactors.length > 0"
        :model-value="wizard.ExternalFactorsNotes"
        label="Describe the factors"
        :rows="3"
        auto-grow
        variant="outlined"
        density="compact"
        class="step6-factors__notes"
        @update:model-value="(val: string) => patchWizard({ ExternalFactorsNotes: val })"
      />
    </div>
  </template>

  <script setup lang="ts">
  import { computed } from "vue";
  import { useAimOnboarding } from "../composables/useAimOnboarding";
  import ToggleCard from "../components/ToggleCard.vue";

  const { wizard, patchWizard } = useAimOnboarding();

  // ── Exactly 6 factors — PR comment A4 consolidation applied ───────────────
  const FACTORS = [
    { id: "promotional", label: "Promotional activity" },
    { id: "competitor", label: "Competitor activity" },
    { id: "major_launch", label: "Major product launch / rebrand" },
    { id: "pr_spike", label: "PR / earned media spike" },
    { id: "macro_economic", label: "Macro economic event" },
    { id: "other", label: "Other" },
  ] as const;

  // ── Seasonality — Partial if any platform history is 12_24 ────────────────
  const seasonalityPartial = computed<boolean>(() => {
    const history = wizard.value.History ?? {};
    return Object.values(history).some((v) => v === "12_24");
  });

  // ── National Holidays region label ────────────────────────────────────────
  const regionLabel = computed<string>(() => wizard.value.Region || "your region");

  // ── Selected factors helper ───────────────────────────────────────────────
  const selectedFactors = computed<string[]>(() => wizard.value.ExternalFactors ?? []);

  // ── Toggle a factor in/out of the selection array ────────────────────────
  function toggleFactor(id: string): void {
    const current = selectedFactors.value.slice();
    const idx = current.indexOf(id);
    if (idx === -1) {
      current.push(id);
    } else {
      current.splice(idx, 1);
    }
    patchWizard({ ExternalFactors: current });
  }
  </script>

  <style scoped lang="scss">
  .step6-factors {
    max-width: 640px;

    &__auto-row {
      display: flex;
      align-items: flex-start;
    }

    &__grid {
      display: grid;
      grid-template-columns: repeat(2, 1fr);
      gap: 12px;
    }

    &__notes {
      margin-top: 4px;
    }
  }
  </style>
  ```

  **Design notes:** `FACTORS` is `as const` — the 6 IDs are compile-time literals, preventing typos.
  `seasonalityPartial` checks `Object.values(history).some(v => v === '12_24')` — matches the spec's "any platform" wording exactly; empty History → `false` → "Included".
  `v-textarea` uses `variant="outlined"` and `density="compact"` matching the existing `InputText` pattern in frontend-mos; no custom focus halo.
  All colors via `rgb(var(--v-theme-*))` — no hardcoded hex (F0 token requirement). `warning` and `success` map to Vuetify theme colors; the semantic token hex values (`#BA4E00`, `#427900`) must be configured in F0 to satisfy WCAG AA — this step references the theme key, not the hex.
  The inner `v-checkbox-btn` inside `ToggleCard` already carries `aria-hidden="true"` and `tabindex="-1"` (C1). The factor grid needs no additional ARIA role because each `ToggleCard` exposes `role="checkbox"` and `aria-checked`.

---

## Task 3: Run tests — confirm all pass

- [ ] **Step 1: Run the Step6Factors spec**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step6Factors.spec.ts 2>&1 | tail -40
  ```

  Expected output:

  ```text
  ✓ Step6Factors.vue
    ✓ renders the step heading
    ✓ renders the Seasonality auto-included row
    ✓ renders the National Holidays auto-included row
    ✓ renders exactly 6 ToggleCard items for additional factors
    ✓ renders all 6 factor labels
    ✓ shows Seasonality as Included when History is empty
    ✓ shows Seasonality as Included when all platforms have 24+ months history
    ✓ shows Seasonality as Partial when any platform has 12_24 history
    ✓ shows Seasonality as Partial when only one platform and it has 12_24 history
    ✓ shows fallback region "your region" when Region is empty
    ✓ shows actual Region value when set
    ✓ does not render the notes textarea when no factors selected
    ✓ renders the notes textarea when at least one factor is selected
    ✓ notes textarea label is Describe the factors
    ✓ calls patchWizard to add a factor when an unselected ToggleCard is toggled
    ✓ calls patchWizard to remove a factor when a selected ToggleCard is toggled
    ✓ calls patchWizard with updated notes on textarea input
    ✓ passes selected=true to a ToggleCard whose ID is in ExternalFactors
    ✓ passes selected=false to a ToggleCard whose ID is not in ExternalFactors
    ✓ does not render any required-field error indicators

  Test Files  1 passed (1)
  Tests       20 passed (20)
  ```

  If any test fails, read the failure message carefully before changing the implementation.
  Common causes: (1) `ToggleCard` not resolved — check the import path relative to `steps/`.
  (2) `vi.mock` path mismatch — the mock path must exactly match the string in the `import` inside the component; if the component imports from `../composables/useAimOnboarding`, the mock path must be `../../composables/useAimOnboarding`.
  (3) `textarea` not found — `v-textarea` renders a `<textarea>` only after Vuetify mounts; confirm `tests.config.ts` registers Vuetify globally.

- [ ] **Step 2: Run the broader advertiser package test suite — confirm no regressions**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run --project packages/advertiser 2>&1 | tail -20
  ```

  Expected: all pre-existing tests still pass; Step6Factors adds 20 new passing tests.

- [ ] **Step 3: TypeScript check**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx tsc --noEmit -p packages/advertiser/tsconfig.json 2>&1 | tail -20
  ```

  Expected: no errors. If `ExternalFactors` or `ExternalFactorsNotes` are not on
  `WizardState` in `interfaces/aimOnboarding.ts` (F2), add them there first — they are in
  the §5 Mongo shape.

---

## Task 4: Lint and commit

- [ ] **Step 1: Run ESLint on the two new files**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx eslint \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step6Factors.vue \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step6Factors.spec.ts
  ```

  Expected: no errors or warnings. Fix any issues before committing.
  Common lint issues: (1) `as const` on the FACTORS array satisfies `@typescript-eslint/no-explicit-any` if the type is later widened — keep it.
  (2) The `(val: string)` inline type annotation in the template satisfies strict-mode template type checking; if lint flags it, move the handler to a named function in `<script setup>`.

- [ ] **Step 2: Lint the markdown plan file (this file) if the project has markdown lint CI**

  ```bash
  npx markdownlint-cli2 \
    "/Users/mukey/Documents/kochava-projects/aim-onboarding-tool/docs/docs/16-PRDs/061-aim-onboarding-decision-tool/plans/W6-step6-external-factors.md"
  ```

  Expected: no errors.

- [ ] **Step 3: Commit**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos
  git add \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step6Factors.vue \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step6Factors.spec.ts
  git commit -m "feat(aim-onboarding): W6 Step6Factors — optional external factors, 6-factor grid, seasonality from history, free-text notes"
  ```

  Expected: commit succeeds; CI test run passes.

---

## Self-review: spec coverage check

| Requirement | Source | Covered by |
|---|---|---|
| Optional step — never blocks | spec §4.1, design §8a | No validation binding; `stepStatus[6] = 'optional'` (enforced in F3, not here); no error alerts in template |
| Seasonality auto-included | spec §4.1 Step 6 | `seasonalityPartial` computed; `✓ Included` / `⚠ Partial` rendering |
| Seasonality Partial when 12_24 | spec §4.1 Step 6, §9 prototype bug #4 | `Object.values(history).some(v => v === '12_24')` |
| National Holidays auto-included + region | spec §4.1 Step 6 | `regionLabel` computed; fallback "your region" |
| Exactly 6 additional factors | spec §4.1 PR comment A4 | `FACTORS` array with 6 entries; test asserts 6 ToggleCards |
| Factor IDs match spec | spec §4.1 Step 6 | `promotional`, `competitor`, `major_launch`, `pr_spike`, `macro_economic`, `other` |
| "Regulatory change" / "Natural disaster" NOT present | spec §4.1 PR comment A4 | Not in `FACTORS`; test only renders 6 cards |
| ToggleCard multi-select grid | design §8b, C1 | `toggle-card` for each factor, `@toggle` → `toggleFactor` |
| Free-text notes visible only when factor selected | spec §4.1 Step 6 | `v-if="selectedFactors.length > 0"` |
| Notes bind to `ExternalFactorsNotes` | spec §7 state shape, design §5 | `@update:model-value` → `patchWizard({ ExternalFactorsNotes: val })` |
| `ExternalFactors` is a string array of IDs | spec §7, design §5 | `toggleFactor` builds `string[]`; `patchWizard({ ExternalFactors: current })` |
| `patchWizard` from `useAimOnboarding` | design §3, F3 | composable import + destructure |
| Checkbox affordance (not radio) | design §8b, C1 | `ToggleCard` (not `v-radio`) |
| All colors via `rgb(var(--v-theme-*))` | design §8b, F0 | No hardcoded hex in template or SCSS |
| Feeds SoW output | spec §4.1 | State written to `wizard.ExternalFactors` / `wizard.ExternalFactorsNotes` — SoW reads these in O1 |
| TDD: test → fail → implement → pass → commit | writing-plans format | Tasks 1 → 2 → 3 → 4 |
| Lint-clean | — | Task 4 Step 1 |
