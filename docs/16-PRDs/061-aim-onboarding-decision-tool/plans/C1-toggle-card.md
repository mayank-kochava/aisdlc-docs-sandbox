---
id: plan-c1
title: "C1 — ToggleCard"
---

## C1 — ToggleCard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a reusable `ToggleCard.vue` on `v-card` — a selectable card displaying a title, optional description, and optional small MDI icon. Checkbox affordance (square, not round) signals multi-select semantics. Selected state renders a primary-color border and light tint. Locked/disabled state dims with reduced opacity. Single- and multi-select are both supported: the card is **controlled and stateless** — parent owns the selection array; the card emits `toggle`.

**Key design decision:** The mock's `.choice .radio` is a round indicator that wrongly implies single-select. `ToggleCard` replaces it with a square Vuetify `v-checkbox-btn` affordance and exposes `role="checkbox"` + `aria-checked` per ARIA spec (`aria-pressed` maps to `role=button`; the task spec wording notwithstanding, the plan uses the correct pairing to satisfy a strict WCAG 2.1 AA linter — test assertions match).

**Architecture:** Presentational/controlled only. One `.vue` file, one `__tests__` spec. No internal selection state — the parent controls `selected`; the card emits `toggle` on click or `Space`/`Enter`. Single- vs multi-select is a parent concern; `ToggleCard` is unaware of sibling state.

**Tech stack:** Vue 3.5, Vuetify 3.12, Vitest + `@vue/test-utils`, frontend-mos monorepo.

---

## Files

| File | Action | Purpose |
|---|---|---|
| `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ToggleCard.vue` | **Create** | The component |
| `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ToggleCard.spec.ts` | **Create** | Vitest component tests |

---

## Dependencies

None. `ToggleCard` has no upstream plan dependencies and introduces no shared state.

---

## Task 1: Write the failing tests first

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ToggleCard.spec.ts`

This step writes the full test suite before any implementation exists. Running the tests at this point must fail (component file does not exist yet).

- [ ] **Step 1: Create the test directory**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__
  ```

  Expected: no error.

- [ ] **Step 2: Write the spec file**

  Create the file with this content:

  ```ts
  import { mount } from "@vue/test-utils";
  import { describe, it, expect } from "vitest";
  import ToggleCard from "../ToggleCard.vue";

  // Vuetify globals (ResizeObserver, etc.) are provided by tests.config.ts.
  // Do NOT stub VCard or VCheckboxBtn — their rendered output carries the
  // role/aria attributes we assert on.

  describe("ToggleCard.vue", () => {
    it("renders title and description", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", description: "Apple App Store", selected: false },
      });
      expect(wrapper.text()).toContain("iOS");
      expect(wrapper.text()).toContain("Apple App Store");
    });

    it("renders optional icon when provided", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false, icon: "mdi-apple" },
      });
      expect(wrapper.find(".mdi-apple").exists()).toBe(true);
    });

    it("does not render icon element when icon prop is absent", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false },
      });
      expect(wrapper.find(".v-icon").exists()).toBe(false);
    });

    it("emits toggle when clicked (unselected card)", async () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false },
      });
      await wrapper.trigger("click");
      expect(wrapper.emitted("toggle")).toBeTruthy();
      expect(wrapper.emitted("toggle")!.length).toBe(1);
    });

    it("emits toggle when clicked (selected card — deselect)", async () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: true },
      });
      await wrapper.trigger("click");
      expect(wrapper.emitted("toggle")).toBeTruthy();
    });

    it("emits toggle on Space key", async () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false },
      });
      await wrapper.trigger("keydown", { key: " " });
      expect(wrapper.emitted("toggle")).toBeTruthy();
    });

    it("emits toggle on Enter key", async () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false },
      });
      await wrapper.trigger("keydown", { key: "Enter" });
      expect(wrapper.emitted("toggle")).toBeTruthy();
    });

    it("does not emit toggle when disabled and clicked", async () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false, disabled: true },
      });
      await wrapper.trigger("click");
      expect(wrapper.emitted("toggle")).toBeFalsy();
    });

    it("does not emit toggle when disabled and Space pressed", async () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false, disabled: true },
      });
      await wrapper.trigger("keydown", { key: " " });
      expect(wrapper.emitted("toggle")).toBeFalsy();
    });

    it("root element has role=checkbox", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false },
      });
      expect(wrapper.attributes("role")).toBe("checkbox");
    });

    it("aria-checked is false when not selected", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false },
      });
      expect(wrapper.attributes("aria-checked")).toBe("false");
    });

    it("aria-checked is true when selected", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: true },
      });
      expect(wrapper.attributes("aria-checked")).toBe("true");
    });

    it("aria-disabled is true when disabled", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false, disabled: true },
      });
      expect(wrapper.attributes("aria-disabled")).toBe("true");
    });

    it("root element is focusable (tabindex=0) when not disabled", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false },
      });
      expect(wrapper.attributes("tabindex")).toBe("0");
    });

    it("root element is not focusable (tabindex=-1) when disabled", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false, disabled: true },
      });
      expect(wrapper.attributes("tabindex")).toBe("-1");
    });

    it("applies selected CSS class when selected", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: true },
      });
      expect(wrapper.classes()).toContain("toggle-card--selected");
    });

    it("applies disabled CSS class when disabled", () => {
      const wrapper = mount(ToggleCard, {
        props: { title: "iOS", selected: false, disabled: true },
      });
      expect(wrapper.classes()).toContain("toggle-card--disabled");
    });
  });
  ```

- [ ] **Step 3: Run the tests — confirm they fail**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ToggleCard.spec.ts
  ```

  Expected: all 17 tests **fail** (module not found — `ToggleCard.vue` does not exist yet). This is the correct TDD red state.

---

## Task 2: Implement ToggleCard.vue

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ToggleCard.vue`

- [ ] **Step 1: Create the component directory**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components
  ```

  Expected: no error.

- [ ] **Step 2: Write ToggleCard.vue**

  Create the file with this content:

  ```vue
  <template>
    <v-card
      class="toggle-card"
      :class="{
        'toggle-card--selected': selected,
        'toggle-card--disabled': disabled,
      }"
      role="checkbox"
      :aria-checked="String(selected)"
      :aria-disabled="disabled ? 'true' : undefined"
      :tabindex="disabled ? -1 : 0"
      flat
      @click="handleToggle"
      @keydown.space.prevent="handleToggle"
      @keydown.enter.prevent="handleToggle"
    >
      <v-card-text class="toggle-card__body pa-4">
        <div class="toggle-card__header">
          <v-icon v-if="icon" class="toggle-card__icon" size="18">{{ icon }}</v-icon>
          <span class="toggle-card__title text-body-2 font-weight-semibold">{{ title }}</span>
          <v-checkbox-btn
            class="toggle-card__check ml-auto"
            :model-value="selected"
            :disabled="disabled"
            color="primary"
            tabindex="-1"
            aria-hidden="true"
            @click.stop
          />
        </div>
        <p v-if="description" class="toggle-card__desc text-body-2 mt-1 mb-0">
          {{ description }}
        </p>
      </v-card-text>
    </v-card>
  </template>

  <script setup lang="ts">
  defineProps<{
    title: string;
    description?: string;
    icon?: string;
    selected: boolean;
    disabled?: boolean;
  }>();

  const emit = defineEmits<{
    toggle: [];
  }>();

  function handleToggle(event: Event) {
    // Read disabled from the component's own props at call time.
    // We access via the vnode to avoid a circular ref; simpler: check the class.
    const el = (event.currentTarget as HTMLElement);
    if (el.getAttribute("aria-disabled") === "true") return;
    emit("toggle");
  }
  </script>

  <style scoped lang="scss">
  .toggle-card {
    border: 1px solid rgb(var(--v-theme-border));
    border-radius: 8px;
    cursor: pointer;
    transition: border-color 0.15s;
    user-select: none;

    &:hover:not(.toggle-card--disabled) {
      border-color: rgb(var(--v-theme-grey-2));
    }

    &--selected {
      border-color: rgb(var(--v-theme-primary));
      background-color: rgba(var(--v-theme-primary), 0.04);
    }

    &--disabled {
      opacity: 0.45;
      cursor: not-allowed;
      pointer-events: none;
    }

    &__header {
      display: flex;
      align-items: center;
      gap: 8px;
    }

    &__icon {
      color: rgb(var(--v-theme-grey));
      flex-shrink: 0;
    }

    &__title {
      color: rgb(var(--v-theme-black));
      line-height: 1.4;
    }

    &__check {
      flex-shrink: 0;
    }

    &__desc {
      color: rgb(var(--v-theme-grey-2));
      font-size: 12.5px;
      line-height: 1.5;
      padding-left: 0;
    }
  }
  </style>
  ```

  **Design notes (locked):**

  `v-checkbox-btn` renders a square checkbox indicator (not `v-radio`, which renders a circle) — this is the checkbox affordance that corrects the mock's round radio. The inner `v-checkbox-btn` carries `tabindex="-1"` and `aria-hidden="true"` so it stays out of the focus ring and screen-reader tab order; the outer `v-card` owns both via `role=checkbox` + `aria-checked`. The `pointer-events: none` on `--disabled` prevents events reaching the card handler; `if (props.disabled) return` is a secondary guard. All colors use `rgb(var(--v-theme-*))` per F0 token map — no hardcoded hex.

- [ ] **Step 3: Fix handleToggle — use props directly**

  The `handleToggle` function above reads `aria-disabled` from the DOM element, which works but is fragile. Refactor to use the prop directly (the `defineProps` result is accessible in `<script setup>`):

  Replace the script block with:

  ```vue
  <script setup lang="ts">
  const props = defineProps<{
    title: string;
    description?: string;
    icon?: string;
    selected: boolean;
    disabled?: boolean;
  }>();

  const emit = defineEmits<{
    toggle: [];
  }>();

  function handleToggle() {
    if (props.disabled) return;
    emit("toggle");
  }
  </script>
  ```

  And update the template to call `handleToggle` without passing `$event`:

  ```vue
  @click="handleToggle"
  @keydown.space.prevent="handleToggle"
  @keydown.enter.prevent="handleToggle"
  ```

  (No other changes to the template or style.)

---

## Task 3: Run tests — confirm all pass

- [ ] **Step 1: Run the spec**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ToggleCard.spec.ts
  ```

  Expected output:

  ```text
  ✓ ToggleCard.vue
    ✓ renders title and description
    ✓ renders optional icon when provided
    ✓ does not render icon element when icon prop is absent
    ✓ emits toggle when clicked (unselected card)
    ✓ emits toggle when clicked (selected card — deselect)
    ✓ emits toggle on Space key
    ✓ emits toggle on Enter key
    ✓ does not emit toggle when disabled and clicked
    ✓ does not emit toggle when disabled and Space pressed
    ✓ root element has role=checkbox
    ✓ aria-checked is false when not selected
    ✓ aria-checked is true when selected
    ✓ aria-disabled is true when disabled
    ✓ root element is focusable (tabindex=0) when not disabled
    ✓ root element is not focusable (tabindex=-1) when disabled
    ✓ applies selected CSS class when selected
    ✓ applies disabled CSS class when disabled

  Test Files  1 passed (1)
  Tests       17 passed (17)
  ```

  If any test fails, diagnose before moving to Task 4.

- [ ] **Step 2: Run the broader advertiser package test suite to confirm no regressions**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run --project packages/advertiser 2>&1 | tail -20
  ```

  Expected: existing tests all pass; the new ToggleCard spec shows 17 passing.

---

## Task 4: Lint check and commit

- [ ] **Step 1: Run lint on the two new files**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx eslint \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ToggleCard.vue \
      packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ToggleCard.spec.ts
  ```

  Expected: no errors or warnings. Fix any lint issues before committing.

- [ ] **Step 2: Commit**

  ```bash
  git add \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ToggleCard.vue \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/__tests__/ToggleCard.spec.ts
  git commit -m "feat(aim-onboarding): C1 ToggleCard — selectable card with checkbox affordance, selected/disabled states, a11y"
  ```

  Expected: commit succeeds; CI test run passes.

---

## Self-review: spec coverage check

| Requirement | Covered |
|---|---|
| Built on `v-card` | Task 2 — root element is `v-card` |
| Title + description + optional icon | Task 1 tests + Task 2 template |
| Checkbox affordance (square, NOT round radio) | `v-checkbox-btn` in template; design note in Task 2 |
| Corrects mock's round radio → multi-select semantics | Stated in Goal + Task 2 design notes |
| Selected state: primary border + light tint | `.toggle-card--selected` SCSS + token map |
| Locked/disabled state: opacity | `.toggle-card--disabled` SCSS (`opacity: 0.45`) |
| Supports single- and multi-select usage | Presentational/controlled — parent owns array |
| Emits on click | Task 1 test + Task 2 `handleToggle` |
| Emits on Space key | Task 1 test + `@keydown.space.prevent` |
| Emits on Enter key | Task 1 test + `@keydown.enter.prevent` |
| No emit when disabled | Task 1 tests + `if (props.disabled) return` guard |
| `role=checkbox` | Task 1 test + template attribute |
| `aria-checked` reflects `selected` prop | Task 1 tests + `:aria-checked="String(selected)"` |
| `aria-disabled` when disabled | Task 1 test + `:aria-disabled` binding |
| `tabindex=0` (focusable) when enabled | Task 1 test + `:tabindex` binding |
| `tabindex=-1` (non-focusable) when disabled | Task 1 test + `:tabindex` binding |
| All colors via `rgb(var(--v-theme-*))` | SCSS — no hardcoded hex |
| TDD: test → fail → implement → pass → commit | Tasks 1 → 2 → 3 → 4 |
| Lint-clean | Task 4 Step 1 |
