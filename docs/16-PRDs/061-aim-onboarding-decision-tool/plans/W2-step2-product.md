---
id: plan-w2
title: "W2 — Step 2: About Your Product"
---

## W2 — Step 2: About Your Product Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `steps/Step2Product.vue` — the second wizard step that collects app/brand name,
platform multi-select (iOS/Android/Web), conversion-share allocation, spend-separability,
cross-journey flag, auto-recommended model structure (Unified/Separate), and primary market.
Progressive disclosure gates each sub-section on its parent answer. Binds to `useAimOnboarding`.
Includes a full Vitest component test suite.

**Architecture:** One thin orchestration component (`Step2Product.vue`) plus two supporting
fixtures — a `REGIONS` constant and a helper that re-seeds `platformShares` on platform-set
changes. All redistribution logic delegates to the already-built `ShareAllocator` (C2).
Model-structure recommendation is a four-line pure `computed`. The component owns no internal
selection state; every field reads from the composable `wizard` ref and writes via `set()`.

**Tech Stack:** Vue 3.5, Vuetify 3.12, Vitest + `@vue/test-utils`, frontend-mos monorepo.
Composable: `useAimOnboarding` (F3) — camelCase state keys. Components: `ToggleCard` (C1),
`ShareAllocator` (C2), `InputSelect` (`packages/core/src/components/inputs/InputSelect.vue`).
Banner: `v-alert variant="tonal"`.

---

## Casing contract (read first)

`WizardState` in F2 (`interfaces/aimOnboarding.ts`) is **PascalCase** on the wire (C# POCO).
`useAimOnboarding` (F3) exposes the flat state as a `ref<WizardState>` but its internal
`defaultWizard()` uses camelCase keys matching the §5 JSON document (`appName`, `platforms`,
`platformShares`, `modelling`, `region`, `regionOther`). The Step 2 component binds to
`wizard.value.*` — those keys are **camelCase** throughout this plan.

The mockup's `conversionShare` key is renamed to `platformShares` in the canonical §5 contract.
The mockup's `modelStructure` key maps to the existing `modelling` field — no new field is needed.

---

## Contract gap — flag before implementation

`spendSeparable` and `crossJourney` are absent from F2's `WizardState` and F3's `defaultWizard()`.
They appear in `mockup-k4a.html` line 218 and are persisted there. The design-consolidated §5 doc
omits them — an omission, not a deliberate exclusion (no other field covers them).

**Task 0 (below) patches F2 and F3 before any Step 2 code is written.**
The F2/F3 owners must merge this patch or coordinate; it is a prerequisite, not optional.

`modelling` already exists in both F2 (`Modelling: string`) and F3 (`modelling: ''`). Step 2
writes `'unified' | 'separate' | ''` to it — no new field required.

---

## Files

| Action | Path | Purpose |
|--------|------|---------|
| **Patch** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts` | Add `SpendSeparable` and `CrossJourney` to `WizardState` (camelCase in F3 defaultWizard) |
| **Patch** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts` | Add `spendSeparable` and `crossJourney` to `defaultWizard()` |
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step2Product.vue` | Step 2 orchestration component |
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step2Product.spec.ts` | Vitest component tests |

---

## Dependencies

| ID | Plan | What is needed |
|----|------|----------------|
| F2 | Onboarding Service + Interfaces | `WizardState` import (+ patch from Task 0) |
| F3 | useAimOnboarding Composable | `useAimOnboarding()`, `wizard`, `set()` (referred to as `patchWizard` in F3 but aliased as `set` locally) |
| C1 | ToggleCard | Platform and model-structure cards |
| C2 | ShareAllocator | Conversion-share sliders |

All four must be merged before Task 1 tests can be run against the real component.

---

## Progressive disclosure map

| Section | Shown when | Dimmed / hidden when |
|---------|-----------|----------------------|
| App / brand name | Always | — |
| Platform cards (2.1) | Always | — |
| iOS ATT/SKAN banner | `platforms` includes `'ios'` | Hidden |
| Conversion share (2.2) | `platforms.length >= 2` | Hidden entirely |
| Spend separable (2.3) | `platforms` has Web + ≥1 mobile | Hidden entirely |
| Cross-journey (2.3b) | `spendSeparable` is not null | Hidden entirely |
| Model structure (2.4) | `spendSeparable` not null AND `crossJourney` not null | Hidden entirely |
| Primary market (2.5) | Always | `platforms.length === 0` → `opacity: 0.32` |

---

## Auto-recommend rule (from `mockup-k4a.html` `autoRec()`, line 527)

```text
if spendSeparable === 'no'  OR  crossJourney === 'yes'  → recommend 'unified'
if spendSeparable === 'yes' AND crossJourney === 'no'   → recommend 'separate'
otherwise                                                → null (no recommendation)
```

`modelling` stores the user's final selection (`'unified' | 'separate' | ''`). When the
auto-recommend changes (due to a `spendSeparable`/`crossJourney` change) the stored
`modelling` value is NOT automatically overwritten — the user's explicit override is preserved.
The Recommended badge moves to the newly recommended card; the user may re-click.

---

## Platform re-seed rule (from `mockup-k4a.html` `toggleP()`, line 518)

When the platform set changes:

```text
1 platform  → platformShares = { [platform]: 100 }
2 platforms → platformShares = { [p0]: 50, [p1]: 50 }
3 platforms → platformShares = { [p0]: 33, [p1]: 33, [p2]: 34 } (largest-remainder)
```

If Web is removed from a Web+mobile set, `modelling` is reset to `''` (the
spend/cross-journey questions no longer apply).

---

## Task 0: Patch F2 and F3

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts`
- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts`

- [ ] **Step 1: Read F2 `WizardState`**

  Open `interfaces/aimOnboarding.ts` and locate the `WizardState` interface. Confirm
  `SpendSeparable` and `CrossJourney` are absent. Confirm `Modelling: string` already exists.

- [ ] **Step 2: Add the missing fields to `WizardState`**

  After `Modelling: string;` add:

  ```ts
  SpendSeparable: 'yes' | 'no' | 'not_sure' | null;
  CrossJourney: 'yes' | 'no' | 'not_sure' | null;
  ```

- [ ] **Step 3: Patch `defaultWizard()` in `useAimOnboarding.ts`**

  Open `composables/useAimOnboarding.ts` and locate `defaultWizard()`. After `modelling: '',`
  add:

  ```ts
  spendSeparable: null,
  crossJourney: null,
  ```

- [ ] **Step 4: Run existing tests to confirm nothing broke**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos
  npm run test:ci -- --reporter=verbose 2>&1 | tail -20
  ```

  Expected: same pass count as before the patch. No new failures.

- [ ] **Step 5: Commit the patch**

  ```bash
  git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts \
          packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts
  git commit -m "feat(aim-onboarding): add spendSeparable/crossJourney to WizardState"
  ```

---

## Task 1: Write the failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step2Product.spec.ts`

- [ ] **Step 1: Create the test directory**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__
  ```

- [ ] **Step 2: Write the spec file**

  ```ts
  import { mount } from "@vue/test-utils";
  import { describe, it, expect, beforeEach, vi } from "vitest";
  import { createVuetify } from "vuetify";
  import Step2Product from "../Step2Product.vue";

  // -- Stub composable so tests control state directly --
  // Keys are camelCase — matching F3's defaultWizard() and wizard.value.* bindings.
  const mockWizard = {
    appName: "",
    platforms: [] as string[],
    platformShares: {} as Record<string, number>,
    spendSeparable: null as "yes" | "no" | "not_sure" | null,
    crossJourney: null as "yes" | "no" | "not_sure" | null,
    modelling: "" as string,
    region: "",
    regionOther: "",
  };
  const mockSet = vi.fn((patch: Partial<typeof mockWizard>) => {
    Object.assign(mockWizard, patch);
  });

  vi.mock("../../composables/useAimOnboarding", () => ({
    useAimOnboarding: () => ({
      wizard: { value: mockWizard },
      set: mockSet,
    }),
  }));

  const vuetify = createVuetify();
  const global = { plugins: [vuetify] };

  function resetWizard() {
    mockWizard.appName = "";
    mockWizard.platforms = [];
    mockWizard.platformShares = {};
    mockWizard.spendSeparable = null;
    mockWizard.crossJourney = null;
    mockWizard.modelling = "";
    mockWizard.region = "";
    mockWizard.regionOther = "";
    mockSet.mockClear();
  }

  describe("Step2Product", () => {
    beforeEach(resetWizard);

    // ── App name ──────────────────────────────────────────────────────────────

    it("renders app-name input with required asterisk", () => {
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="app-name-label"]').text()).toContain("*");
      expect(wrapper.find('[data-testid="app-name-input"]').exists()).toBe(true);
    });

    it("calls set({ appName }) when app-name input changes", async () => {
      const wrapper = mount(Step2Product, { global });
      await wrapper.find('[data-testid="app-name-input"]').setValue("MyApp");
      expect(mockSet).toHaveBeenCalledWith(expect.objectContaining({ appName: "MyApp" }));
    });

    // ── Platform cards (2.1) ─────────────────────────────────────────────────

    it("renders three platform ToggleCards: iOS, Android, Web", () => {
      const wrapper = mount(Step2Product, { global });
      const cards = wrapper.findAllComponents({ name: "ToggleCard" });
      const platformCards = cards.filter((c) =>
        ["iOS", "Android", "Web"].includes(c.props("title") as string),
      );
      expect(platformCards).toHaveLength(3);
    });

    it("calls set with platforms + re-seeded platformShares when iOS toggled on", async () => {
      const wrapper = mount(Step2Product, { global });
      const iosCard = wrapper
        .findAllComponents({ name: "ToggleCard" })
        .find((c) => c.props("title") === "iOS");
      await iosCard!.trigger("click");
      expect(mockSet).toHaveBeenCalledWith(
        expect.objectContaining({
          platforms: ["ios"],
          platformShares: { ios: 100 },
        }),
      );
    });

    it("re-seeds platformShares 50/50 when two platforms selected", async () => {
      mockWizard.platforms = ["ios"];
      mockWizard.platformShares = { ios: 100 };
      const wrapper = mount(Step2Product, { global });
      const androidCard = wrapper
        .findAllComponents({ name: "ToggleCard" })
        .find((c) => c.props("title") === "Android");
      await androidCard!.trigger("click");
      expect(mockSet).toHaveBeenCalledWith(
        expect.objectContaining({
          platforms: ["ios", "android"],
          platformShares: { ios: 50, android: 50 },
        }),
      );
    });

    it("resets modelling to '' when Web is removed", async () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.modelling = "unified";
      const wrapper = mount(Step2Product, { global });
      const webCard = wrapper
        .findAllComponents({ name: "ToggleCard" })
        .find((c) => c.props("title") === "Web");
      await webCard!.trigger("click");
      expect(mockSet).toHaveBeenCalledWith(
        expect.objectContaining({ modelling: "" }),
      );
    });

    // ── iOS ATT/SKAN banner ───────────────────────────────────────────────────

    it("shows iOS ATT/SKAN info banner when iOS is in platforms", () => {
      mockWizard.platforms = ["ios"];
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="ios-banner"]').exists()).toBe(true);
    });

    it("hides iOS ATT/SKAN banner when iOS is not in platforms", () => {
      mockWizard.platforms = ["android"];
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="ios-banner"]').exists()).toBe(false);
    });

    // ── Conversion share (2.2) ────────────────────────────────────────────────

    it("hides ShareAllocator when fewer than 2 platforms selected", () => {
      mockWizard.platforms = ["ios"];
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.findComponent({ name: "ShareAllocator" }).exists()).toBe(false);
    });

    it("shows ShareAllocator when 2 or more platforms selected", () => {
      mockWizard.platforms = ["ios", "android"];
      mockWizard.platformShares = { ios: 50, android: 50 };
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.findComponent({ name: "ShareAllocator" }).exists()).toBe(true);
    });

    // ── Spend separable (2.3) ─────────────────────────────────────────────────

    it("hides spend-separable section when no Web platform", () => {
      mockWizard.platforms = ["ios", "android"];
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="spend-separable"]').exists()).toBe(false);
    });

    it("shows spend-separable section when Web + at least one mobile platform", () => {
      mockWizard.platforms = ["ios", "web"];
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="spend-separable"]').exists()).toBe(true);
    });

    it("hides cross-journey until spendSeparable is answered", () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = null;
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="cross-journey"]').exists()).toBe(false);
    });

    it("shows cross-journey once spendSeparable is answered", () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = "yes";
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="cross-journey"]').exists()).toBe(true);
    });

    // ── Model structure (2.4) ─────────────────────────────────────────────────

    it("hides model structure until both separable + cross-journey answered", () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = "yes";
      mockWizard.crossJourney = null;
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="model-structure"]').exists()).toBe(false);
    });

    it("shows model structure once both questions answered", () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = "yes";
      mockWizard.crossJourney = "no";
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="model-structure"]').exists()).toBe(true);
    });

    it("auto-recommends 'separate' when spendSeparable=yes and crossJourney=no", () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = "yes";
      mockWizard.crossJourney = "no";
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="rec-badge-separate"]').exists()).toBe(true);
    });

    it("auto-recommends 'unified' when spendSeparable=no", () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = "no";
      mockWizard.crossJourney = "yes";
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="rec-badge-unified"]').exists()).toBe(true);
    });

    it("auto-recommends 'unified' when crossJourney=yes", () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = "yes";
      mockWizard.crossJourney = "yes";
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="rec-badge-unified"]').exists()).toBe(true);
    });

    it("calls set({ modelling: 'unified' }) when unified card clicked", async () => {
      mockWizard.platforms = ["ios", "web"];
      mockWizard.spendSeparable = "yes";
      mockWizard.crossJourney = "no";
      const wrapper = mount(Step2Product, { global });
      await wrapper.find('[data-testid="model-card-unified"]').trigger("click");
      expect(mockSet).toHaveBeenCalledWith(
        expect.objectContaining({ modelling: "unified" }),
      );
    });

    // ── Primary market (2.5) ──────────────────────────────────────────────────

    it("market select is always rendered", () => {
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="market-select"]').exists()).toBe(true);
    });

    it("market section has dim class when no platforms selected", () => {
      mockWizard.platforms = [];
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="market-section"]').classes()).toContain("dim");
    });

    it("market section is not dimmed once a platform is selected", () => {
      mockWizard.platforms = ["ios"];
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="market-section"]').classes()).not.toContain("dim");
    });

    it("shows regionOther input when region is 'Other'", () => {
      mockWizard.platforms = ["ios"];
      mockWizard.region = "Other";
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="region-other-input"]').exists()).toBe(true);
    });

    it("hides regionOther input when region is not 'Other'", () => {
      mockWizard.platforms = ["ios"];
      mockWizard.region = "US";
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="region-other-input"]').exists()).toBe(false);
    });

    // ── Accessibility ─────────────────────────────────────────────────────────

    it("app-name input has aria-required=true", () => {
      const wrapper = mount(Step2Product, { global });
      expect(
        wrapper.find('[data-testid="app-name-input"]').attributes("aria-required"),
      ).toBe("true");
    });

    it("platform section legend is present for screen readers", () => {
      const wrapper = mount(Step2Product, { global });
      expect(wrapper.find('[data-testid="platform-legend"]').exists()).toBe(true);
    });
  });
  ```

- [ ] **Step 3: Run the tests and confirm they all FAIL**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos
  npm run test:ci -- packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step2Product.spec.ts 2>&1 | tail -20
  ```

  Expected: all tests fail with "Cannot find module '../Step2Product.vue'" or equivalent.

---

## Task 2: Implement `Step2Product.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step2Product.vue`

- [ ] **Step 1: Create the steps directory (if absent)**

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps
  ```

- [ ] **Step 2: Write `Step2Product.vue`**

  ```vue
  <template>
    <div class="aim-step2">
      <!-- ── Header ──────────────────────────────────────────────────────── -->
      <div class="aim-step__eyebrow">
        <strong>Step 2 of 8</strong> · About your product
      </div>
      <h2 class="aim-step__title">Tell us about your product</h2>
      <p class="aim-step__lead">
        Basic information about your product, where it runs, and the markets
        you advertise in.
      </p>

      <!-- ── Q: App / brand name ────────────────────────────────────────── -->
      <div class="aim-qcard">
        <label
          data-testid="app-name-label"
          class="aim-field-label"
          for="aim-app-name"
        >
          App / brand name <span class="aim-req" aria-hidden="true">*</span>
        </label>
        <v-text-field
          id="aim-app-name"
          data-testid="app-name-input"
          :model-value="wizard.appName"
          placeholder="e.g. MyApp"
          variant="outlined"
          density="compact"
          aria-required="true"
          style="max-width: 360px"
          @update:model-value="(v: string) => set({ appName: v })"
        />
      </div>

      <!-- ── Q 2.1: Platforms ───────────────────────────────────────────── -->
      <div class="aim-qcard">
        <div class="aim-qhead">
          <span class="aim-qnum">2.1</span>
          <span
            id="aim-platform-heading"
            class="aim-qtitle"
          >
            Which platforms does your product run on?
            <span class="aim-req" aria-hidden="true">*</span>
          </span>
        </div>
        <p class="aim-qhelp">
          The platforms your product runs on shape how we structure your model
          — selecting accurately avoids rework later in onboarding.
        </p>

        <!-- role=group + aria-labelledby provides the screen-reader legend -->
        <div
          role="group"
          aria-labelledby="aim-platform-heading"
          data-testid="platform-legend"
          class="aim-cards"
        >
          <ToggleCard
            v-for="p in PLATFORMS"
            :key="p.key"
            :title="p.label"
            :description="p.description"
            :selected="wizard.platforms.includes(p.key)"
            @click="togglePlatform(p.key)"
          />
        </div>

        <!-- iOS ATT/SKAN banner -->
        <v-alert
          v-if="wizard.platforms.includes('ios')"
          data-testid="ios-banner"
          variant="tonal"
          type="info"
          class="mt-3"
          density="compact"
        >
          iOS models include ATT / SKAN modelling to handle post-privacy
          attribution gaps — no extra setup required.
        </v-alert>
      </div>

      <!-- ── Q 2.2: Conversion share (≥2 platforms) ─────────────────────── -->
      <div v-if="wizard.platforms.length >= 2" class="aim-qcard">
        <div class="aim-qhead">
          <span class="aim-qnum">2.2</span>
          <span class="aim-qtitle">Conversion share across platforms</span>
        </div>
        <p class="aim-qhelp">
          Allocate the share of conversions across your platforms — must total
          100%.
        </p>
        <ShareAllocator
          :model-value="wizard.platformShares"
          :items="platformShareItems"
          @update:model-value="(v: Record<string, number>) => set({ platformShares: v })"
        />
      </div>

      <!-- ── Q 2.3: Spend separable (Web + ≥1 mobile) ───────────────────── -->
      <div v-if="showSep" data-testid="spend-separable" class="aim-qcard">
        <div class="aim-qhead">
          <span class="aim-qnum">2.3</span>
          <span class="aim-qtitle">Is your spend separable by platform?</span>
        </div>
        <p class="aim-qhelp">
          Can you attribute ad spend separately to web vs mobile?
        </p>
        <div role="group" aria-label="Spend separable">
          <v-btn-toggle
            :model-value="wizard.spendSeparable"
            density="compact"
            variant="outlined"
            @update:model-value="(v: string) => set({ spendSeparable: v as 'yes' | 'no' | 'not_sure' })"
          >
            <v-btn value="yes">Yes</v-btn>
            <v-btn value="no">No</v-btn>
            <v-btn value="not_sure">Not sure</v-btn>
          </v-btn-toggle>
        </div>

        <!-- Q 2.3b: Cross-journey (shown once spendSeparable answered) -->
        <div
          v-if="wizard.spendSeparable"
          data-testid="cross-journey"
          class="aim-subq mt-5"
        >
          <div class="aim-qhead">
            <span class="aim-qnum">2.3b</span>
            <span class="aim-qtitle">Cross-journey migration?</span>
          </div>
          <p class="aim-qhelp">
            Do users typically start on one platform and convert on another?
          </p>
          <div role="group" aria-label="Cross-journey migration">
            <v-btn-toggle
              :model-value="wizard.crossJourney"
              density="compact"
              variant="outlined"
              @update:model-value="(v: string) => set({ crossJourney: v as 'yes' | 'no' | 'not_sure' })"
            >
              <v-btn value="yes">Yes</v-btn>
              <v-btn value="no">No</v-btn>
              <v-btn value="not_sure">Not sure</v-btn>
            </v-btn-toggle>
          </div>
        </div>
      </div>

      <!-- ── Q 2.4: Model structure (both 2.3 questions answered) ──────── -->
      <div
        v-if="showRec"
        data-testid="model-structure"
        class="aim-qcard"
      >
        <div class="aim-qhead">
          <span class="aim-qnum">2.4</span>
          <span class="aim-qtitle">Recommended model structure</span>
        </div>
        <p class="aim-qhelp">
          Based on your answers, we recommend a model structure. You can
          override this choice.
        </p>
        <div class="aim-cards">
          <div
            data-testid="model-card-unified"
            role="checkbox"
            :aria-checked="wizard.modelling === 'unified'"
            class="aim-choice"
            :class="{ 'aim-choice--sel': wizard.modelling === 'unified' }"
            tabindex="0"
            @click="set({ modelling: 'unified' })"
            @keydown.space.prevent="set({ modelling: 'unified' })"
            @keydown.enter.prevent="set({ modelling: 'unified' })"
          >
            <span class="aim-choice__chk" aria-hidden="true" />
            <h4 class="aim-choice__title">Unified</h4>
            <p class="aim-choice__desc">One model across all platforms</p>
            <span
              v-if="autoRec === 'unified'"
              data-testid="rec-badge-unified"
              class="aim-badge-rec"
            >
              Recommended
            </span>
          </div>
          <div
            data-testid="model-card-separate"
            role="checkbox"
            :aria-checked="wizard.modelling === 'separate'"
            class="aim-choice"
            :class="{ 'aim-choice--sel': wizard.modelling === 'separate' }"
            tabindex="0"
            @click="set({ modelling: 'separate' })"
            @keydown.space.prevent="set({ modelling: 'separate' })"
            @keydown.enter.prevent="set({ modelling: 'separate' })"
          >
            <span class="aim-choice__chk" aria-hidden="true" />
            <h4 class="aim-choice__title">Separate</h4>
            <p class="aim-choice__desc">Separate model per platform</p>
            <span
              v-if="autoRec === 'separate'"
              data-testid="rec-badge-separate"
              class="aim-badge-rec"
            >
              Recommended
            </span>
          </div>
        </div>
      </div>

      <!-- ── Q 2.5: Primary market ──────────────────────────────────────── -->
      <div
        data-testid="market-section"
        class="aim-qcard"
        :class="{ dim: wizard.platforms.length === 0 }"
      >
        <div class="aim-qhead">
          <span class="aim-qnum">2.5</span>
          <span class="aim-qtitle">
            Which market do you want the model for?
            <span class="aim-req" aria-hidden="true">*</span>
          </span>
        </div>
        <p class="aim-qhelp">
          AIM models are built per market. We recommend starting with one of
          your primary markets.
        </p>
        <label class="aim-field-label" for="aim-market-select">
          Primary market
        </label>
        <InputSelect
          id="aim-market-select"
          data-testid="market-select"
          :model-value="wizard.region"
          :items="REGIONS"
          no-search
          placeholder="Select…"
          style="max-width: 320px"
          :disabled="wizard.platforms.length === 0"
          aria-required="true"
          @update:model-value="(v: string) => set({ region: v })"
        />
        <v-text-field
          v-if="wizard.region === 'Other'"
          data-testid="region-other-input"
          :model-value="wizard.regionOther"
          placeholder="Please specify…"
          variant="outlined"
          density="compact"
          class="mt-2"
          style="max-width: 320px"
          @update:model-value="(v: string) => set({ regionOther: v })"
        />
      </div>
    </div>
  </template>

  <script setup lang="ts">
  import { computed } from "vue";
  import { useAimOnboarding } from "../composables/useAimOnboarding";
  import ToggleCard from "../components/ToggleCard.vue";
  import ShareAllocator from "../components/ShareAllocator/ShareAllocator.vue";
  import InputSelect from "@core/components/inputs/InputSelect.vue";

  const { wizard, patchWizard: set } = useAimOnboarding();

  // ── Constants ────────────────────────────────────────────────────────────

  const PLATFORMS = [
    { key: "ios", label: "iOS", description: "iPhone & iPad / App Store" },
    { key: "android", label: "Android", description: "Google Play Store" },
    { key: "web", label: "Web", description: "Browser based" },
  ] as const;

  const REGIONS = ["US", "UK", "Germany", "France", "Canada", "Other"];

  // ── Derived state ────────────────────────────────────────────────────────

  /** Items descriptor for ShareAllocator — one per active platform. */
  const platformShareItems = computed(() =>
    wizard.value.platforms.map((key) => ({
      key,
      label: key.toUpperCase(),
    })),
  );

  /** Show spend-separable section: Web AND at least one mobile platform. */
  const showSep = computed(() => {
    const pf = wizard.value.platforms;
    return pf.includes("web") && pf.some((p) => p !== "web");
  });

  /** Show model-structure section: both 2.3 questions answered. */
  const showRec = computed(
    () =>
      showSep.value &&
      wizard.value.spendSeparable !== null &&
      wizard.value.crossJourney !== null,
  );

  /**
   * Auto-recommend rule (mockup-k4a.html autoRec(), line 527):
   *   spendSeparable=no OR crossJourney=yes  → 'unified'
   *   spendSeparable=yes AND crossJourney=no → 'separate'
   *   otherwise                               → null
   */
  const autoRec = computed<"unified" | "separate" | null>(() => {
    if (!showRec.value) return null;
    const { spendSeparable: sep, crossJourney: cross } = wizard.value;
    if (sep === "no" || cross === "yes") return "unified";
    if (sep === "yes" && cross === "no") return "separate";
    return null;
  });

  // ── Handlers ─────────────────────────────────────────────────────────────

  /**
   * Toggle a platform on/off and re-seed platformShares to equal split.
   * If Web is removed from a Web+mobile set, modelling resets to ''.
   */
  function togglePlatform(key: string): void {
    const current = wizard.value.platforms;
    const next = current.includes(key)
      ? current.filter((p) => p !== key)
      : [...current, key];

    // Re-seed platformShares to equal split across new platform set.
    const shares: Record<string, number> = {};
    if (next.length === 1) {
      shares[next[0]] = 100;
    } else if (next.length === 2) {
      shares[next[0]] = 50;
      shares[next[1]] = 50;
    } else {
      // 3 platforms: largest-remainder to ensure sum = 100.
      const base = Math.floor(100 / next.length);
      const extra = 100 - base * next.length;
      next.forEach((p, i) => {
        shares[p] = i < extra ? base + 1 : base;
      });
    }

    // Reset modelling if Web is being removed from the set.
    const wasWeb = current.includes("web");
    const isWeb = next.includes("web");
    const modellingPatch = wasWeb && !isWeb ? { modelling: "" } : {};

    set({ platforms: next, platformShares: shares, ...modellingPatch });
  }
  </script>

  <style scoped>
  .aim-step2 {
    max-width: 680px;
  }
  .aim-step__eyebrow {
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.07em;
    text-transform: uppercase;
    color: rgb(var(--v-theme-on-surface-variant));
    margin-bottom: 4px;
  }
  .aim-step__eyebrow strong {
    color: rgb(var(--v-theme-primary));
  }
  .aim-step__title {
    font-size: 24px;
    font-weight: 700;
    letter-spacing: -0.015em;
    margin: 8px 0 6px;
  }
  .aim-step__lead {
    color: rgb(var(--v-theme-on-surface-variant));
    font-size: 14px;
    max-width: 600px;
    margin: 0 0 4px;
  }
  .aim-qcard {
    background: rgb(var(--v-theme-surface));
    border: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
    border-radius: 8px;
    padding: 22px 24px;
    margin-top: 20px;
  }
  .aim-qcard.dim {
    opacity: 0.32;
    pointer-events: none;
  }
  .aim-qhead {
    display: flex;
    align-items: center;
    gap: 9px;
    margin-bottom: 4px;
  }
  .aim-qnum {
    font-size: 10px;
    font-weight: 600;
    color: rgb(var(--v-theme-on-surface-variant));
    background: rgba(var(--v-border-color), 0.3);
    border: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
    border-radius: 5px;
    padding: 1px 6px;
  }
  .aim-qtitle {
    font-size: 15.5px;
    font-weight: 600;
  }
  .aim-qhelp {
    font-size: 13px;
    color: rgb(var(--v-theme-on-surface-variant));
    margin: 0 0 16px;
    max-width: 560px;
  }
  .aim-field-label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 10.5px;
    font-weight: 600;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: rgb(var(--v-theme-on-surface-variant));
    margin-bottom: 8px;
  }
  .aim-req {
    color: rgb(var(--v-theme-error));
  }
  .aim-cards {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
  }
  .aim-choice {
    position: relative;
    padding: 18px 18px 20px;
    border: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
    border-radius: 8px;
    cursor: pointer;
    background: rgb(var(--v-theme-surface));
    transition: border-color 0.12s;
  }
  .aim-choice:hover {
    border-color: rgba(var(--v-theme-on-surface), 0.4);
  }
  .aim-choice--sel {
    border-color: rgb(var(--v-theme-primary));
    background: rgba(var(--v-theme-primary), 0.035);
  }
  .aim-choice__chk {
    display: block;
    position: absolute;
    top: 16px;
    right: 16px;
    width: 16px;
    height: 16px;
    border: 1.5px solid rgba(var(--v-border-color), var(--v-border-opacity));
    border-radius: 3px;
    background: rgb(var(--v-theme-surface));
  }
  .aim-choice--sel .aim-choice__chk {
    background: rgb(var(--v-theme-primary));
    border-color: rgb(var(--v-theme-primary));
  }
  .aim-choice__title {
    font-size: 14px;
    font-weight: 600;
    margin: 0 0 3px;
    padding-right: 24px;
  }
  .aim-choice__desc {
    font-size: 12.5px;
    color: rgb(var(--v-theme-on-surface-variant));
    margin: 0;
  }
  .aim-badge-rec {
    display: inline-block;
    margin-top: 8px;
    font-size: 10px;
    font-weight: 600;
    color: rgb(var(--v-theme-success));
    background: rgba(var(--v-theme-success), 0.1);
    border-radius: 9999px;
    padding: 2px 9px;
  }
  .aim-subq {
    padding-top: 16px;
    border-top: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
    margin-top: 16px;
  }
  </style>
  ```

- [ ] **Step 3: Run the tests**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos
  npm run test:ci -- packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step2Product.spec.ts 2>&1 | tail -30
  ```

  Expected: all tests pass. Investigate any failure before proceeding.

- [ ] **Step 4: Run the full test suite**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos
  npm run test:ci 2>&1 | tail -10
  ```

  Expected: same pass count as before. No regressions.

- [ ] **Step 5: Commit**

  ```bash
  git add \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/Step2Product.vue \
    packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/steps/__tests__/Step2Product.spec.ts
  git commit -m "feat(aim-onboarding): W2 Step2Product — platforms, share allocator, model structure, market"
  ```

---

## Self-review

### Spec coverage check

| Spec requirement (§4.1 Step 2) | Task that implements it |
|---|---|
| App/brand name — required | Task 2 template, Task 1 test |
| Platform multi-select (iOS/Android/Web) — min 1 required | Task 2 `togglePlatform`, Task 1 platform tests |
| iOS ATT/SKAN banner | Task 2 `v-alert`, Task 1 banner tests |
| Conversion share (≥2 platforms) — sliders sum to 100% | Task 2 `ShareAllocator` (delegates to C2), Task 1 show/hide test |
| Spend separable — Web + ≥1 mobile only | Task 2 `showSep`, Task 1 spend tests |
| Cross-journey — after spend answered | Task 2 `wizard.spendSeparable !== null` gate, Task 1 test |
| Model structure — Unified/Separate, auto-recommend badge, override | Task 2 `autoRec`, Task 1 rec-badge tests |
| Primary market — required, locked until platform selected | Task 2 `.dim` class + `:disabled`, Task 1 dim/Other tests |
| `regionOther` text field when "Other" selected | Task 2 `v-if="wizard.region === 'Other'"`, Task 1 regionOther tests |
| Progressive disclosure + opacity 0.32 for locked | Task 2 `.aim-qcard.dim` CSS |
| Platform re-seed on platform-set change | Task 2 `togglePlatform`, Task 1 re-seed tests |
| `modelling` reset to `''` when Web removed | Task 2 `modellingPatch`, Task 1 test |
| Binds to composable (`useAimOnboarding`) | Task 2 `const { wizard, patchWizard: set } = useAimOnboarding()` |
| `spendSeparable`, `crossJourney` in WizardState | Task 0 F2+F3 patch |

### Placeholder scan

No TBD, TODO, "implement later," "add validation," or "similar to task N" found.

### Type consistency

- `wizard.value.spendSeparable: 'yes' | 'no' | 'not_sure' | null` — matches F2 `SpendSeparable` wire type and `v-btn value="not_sure"` in template; matches composable mock `spendSeparable` key in spec.
- `wizard.value.modelling: string` — already in F2 (`Modelling: string`) and F3 `defaultWizard()`. Set as `'unified' | 'separate' | ''` by template clicks; matches `set({ modelling: ... })` in both template and tests.
- `wizard.value.platformShares: Record<string, number>` (from F2/F3) — matches `ShareAllocator` `:model-value` type and `togglePlatform` return shape.
- `autoRec: ComputedRef<'unified' | 'separate' | null>` — matches badge conditionals `v-if="autoRec === 'unified'"`.
- `platformShareItems` shape: `Array<{ key: string; label: string }>` — matches C2 `ShareAllocator` `items` prop.
- `InputSelect` props: `:items="REGIONS"` (string array), `no-search` (prop `noSearch` in kebab form), `@update:model-value` — all verified against `packages/core/src/components/inputs/InputSelect.vue`.
- `patchWizard` is the mutation function name in F3; aliased to `set` locally — consistent with all call sites in the template.
