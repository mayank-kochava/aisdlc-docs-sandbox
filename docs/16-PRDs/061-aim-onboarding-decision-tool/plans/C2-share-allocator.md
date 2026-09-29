---
id: plan-c2
title: "C2 — ShareAllocator"
---

## Goal

Build `ShareAllocator.vue` — a set of `v-slider` inputs (one per item) whose values must sum to 100%. Moving one slider proportionally redistributes the others to maintain the invariant; a running total displays green at exactly 100% and red otherwise; an error banner appears when the sum ≠ 100%. The component accepts a `Record<string, number>` model and an `items` descriptor array, making it reusable for both Step 2 (platform conversion share) and Step 3 (campaign-type budget split).

---

## Files

**Create**

- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/redistribute.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/redistribute.spec.ts`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/ShareAllocator.vue`
- `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/ShareAllocator.spec.ts`

**No existing files modified** (C2 is a new, standalone component).

---

## Dependencies

None. This plan is self-contained.

---

## Design notes (read before coding)

**Two-model interaction pattern.** Redistribution is the primary mode: when the user moves a slider, the other sliders scale proportionally so the total stays 100%. The error (red) state is reachable only when the initial `modelValue` prop arrives with a sum ≠ 100 (e.g. a partially-filled or legacy document). Redistribution itself never produces a non-100 sum, so the error state is purely a "bad initial value" guard and a signal the parent step's validation reads.

**Integer rounding.** Naive `Math.round` across N values can produce 99 or 101. Use largest-remainder on the *other* values after the changed slider is pinned so the remaining set sums to exactly `100 - newValue`. See Task 1 for the full algorithm.

**Edge case — all others are zero.** If every other item is at 0, there is no weight to distribute proportionally; divide the remainder evenly across the other slots instead.

**Avoid feedback loops.** `redistribute()` is called once from the `v-slider` `@update:modelValue` handler, producing a fresh complete `Record`. The component replaces `localValues` atomically; the resulting `v-model` bindings read `localValues[key]` and do not re-trigger the handler.

**Tokens.** All colors via Vuetify theme CSS custom properties — `rgb(var(--v-theme-success))` for 100%, `rgb(var(--v-theme-error))` for ≠ 100%. No hardcoded hex. The existing `InputSlider.vue` is index-based (discrete items → label) and cannot do percentage allocation — this component wraps `v-slider` directly.

**Accessibility.** `v-slider` renders `role="slider"` with `aria-valuemin`, `aria-valuemax`, `aria-valuenow`, and full keyboard support (arrow keys ±1%, Home/End) automatically. The `role="slider"` element is a `div.v-slider-thumb` (not a native `<input>`); Vuetify wires its `aria-label` from VSlider's `name` prop (not `aria-label` — verified in `VSliderThumb.js` line 140: `"aria-label": props.name`). Each slider therefore needs `:name="item.label"`, not `:aria-label`. The running total is wrapped in an `aria-live="polite"` region so screen readers announce the updated sum without requiring a re-focus.

---

## Tasks

### Task 1 — Write the failing unit test for `redistribute`

**1.1** Create the test file (the source it imports does not exist yet):

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/redistribute.spec.ts
```

```typescript
import { describe, it, expect } from "vitest";
import { redistribute } from "../redistribute";

describe("redistribute", () => {
  it("pins the changed index and scales others proportionally", () => {
    // Start: iOS=55, Android=45. Move iOS to 70 → Android gets remainder 30.
    const result = redistribute({ iOS: 55, Android: 45 }, ["iOS", "Android"], "iOS", 70);
    expect(result).toEqual({ iOS: 70, Android: 30 });
    expect(Object.values(result).reduce((a, b) => a + b, 0)).toBe(100);
  });

  it("three items — proportional scale and exact integer sum", () => {
    // Start: UA=60, UE=30, Brand=10. Move UA to 50.
    // Remainder = 50. Others sum = 40 (UE=30, Brand=10).
    // UE: 30/40 * 50 = 37.5 → 38; Brand: 10/40 * 50 = 12.5 → 12. Total others = 50. ✓
    const result = redistribute(
      { UA: 60, UE: 30, Brand: 10 },
      ["UA", "UE", "Brand"],
      "UA",
      50
    );
    expect(result.UA).toBe(50);
    expect(result.UE + result.Brand).toBe(50);
    expect(Object.values(result).reduce((a, b) => a + b, 0)).toBe(100);
  });

  it("largest-remainder rounding — never over- or under-shoots by 1", () => {
    // 34/33/33 → move idx 0 to 33. Others split 67. 33.5 each → 34/33. Total = 100.
    const result = redistribute(
      { A: 34, B: 33, C: 33 },
      ["A", "B", "C"],
      "A",
      33
    );
    expect(result.A).toBe(33);
    expect(result.B + result.C).toBe(67);
    expect(Object.values(result).reduce((a, b) => a + b, 0)).toBe(100);
  });

  it("all-others-zero fallback — divides evenly", () => {
    // iOS=100, Android=0, Web=0 → move iOS to 40 → Android and Web share 60 equally.
    const result = redistribute(
      { iOS: 100, Android: 0, Web: 0 },
      ["iOS", "Android", "Web"],
      "iOS",
      40
    );
    expect(result.iOS).toBe(40);
    expect(result.Android).toBe(30);
    expect(result.Web).toBe(30);
    expect(Object.values(result).reduce((a, b) => a + b, 0)).toBe(100);
  });

  it("two-item case — remainder goes entirely to the other item", () => {
    const result = redistribute(
      { iOS: 55, Android: 45 },
      ["iOS", "Android"],
      "Android",
      20
    );
    expect(result.Android).toBe(20);
    expect(result.iOS).toBe(80);
    expect(Object.values(result).reduce((a, b) => a + b, 0)).toBe(100);
  });

  it("clamps new value to [0, 100] before redistributing", () => {
    // Passing 110 → treated as 100; others get 0.
    const result = redistribute({ A: 50, B: 50 }, ["A", "B"], "A", 110);
    expect(result.A).toBe(100);
    expect(result.B).toBe(0);
    expect(Object.values(result).reduce((a, b) => a + b, 0)).toBe(100);
  });
});
```

**1.2** Run the test to confirm it fails (no implementation yet):

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/redistribute.spec.ts --environment jsdom
```

Expected: `Error: Failed to resolve import "../redistribute"` (or similar module-not-found). If it passes, something is wrong — stop and investigate before continuing.

---

### Task 2 — Implement `redistribute.ts`

**2.1** Create the source file:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/redistribute.ts
```

```typescript
/**
 * Pins `changedKey` to `newValue` (clamped to [0, 100]) and proportionally
 * redistributes the remainder across all other keys so the total always equals 100.
 *
 * Uses largest-remainder rounding so integer values sum to exactly 100.
 * Falls back to even distribution when all other values are currently 0.
 *
 * @param current  - existing Record<string, number> (values are integers, sum may vary)
 * @param keys     - ordered key list (defines display + rounding order)
 * @param changedKey - the key the user moved
 * @param newValue - the raw value from the slider (will be clamped)
 */
export function redistribute(
  current: Record<string, number>,
  keys: string[],
  changedKey: string,
  newValue: number
): Record<string, number> {
  const pinned = Math.min(100, Math.max(0, Math.round(newValue)));
  const remainder = 100 - pinned;
  const otherKeys = keys.filter((k) => k !== changedKey);

  if (otherKeys.length === 0) {
    return { [changedKey]: pinned };
  }

  const othersSum = otherKeys.reduce((sum, k) => sum + (current[k] ?? 0), 0);

  // Compute exact (fractional) shares for each other key.
  let exact: number[];
  if (othersSum === 0) {
    // All-others-zero: divide evenly.
    const even = remainder / otherKeys.length;
    exact = otherKeys.map(() => even);
  } else {
    exact = otherKeys.map((k) => ((current[k] ?? 0) / othersSum) * remainder);
  }

  // Largest-remainder rounding: floors first, then distributes leftover 1s.
  const floored = exact.map(Math.floor);
  const leftover = remainder - floored.reduce((a, b) => a + b, 0);
  const remainders = exact.map((v, i) => ({ i, r: v - floored[i] }));
  remainders.sort((a, b) => b.r - a.r);
  for (let idx = 0; idx < leftover; idx++) {
    floored[remainders[idx].i] += 1;
  }

  const result: Record<string, number> = { [changedKey]: pinned };
  otherKeys.forEach((k, idx) => {
    result[k] = floored[idx];
  });
  return result;
}
```

**2.2** Run the redistribute tests and confirm all pass:

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/redistribute.spec.ts --environment jsdom
```

Expected: `6 passed`.

---

### Task 3 — Commit the pure-logic layer

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/redistribute.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/redistribute.spec.ts

git commit -m "feat(onboarding): add redistribute() — sum-to-100 proportional algorithm with tests (C2)"
```

---

### Task 4 — Write the failing component test for `ShareAllocator.vue`

**4.1** Create the component test file (the `.vue` file does not exist yet):

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/ShareAllocator.spec.ts
```

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { mount } from "@vue/test-utils";
import { createVuetify } from "vuetify";
import ShareAllocator from "../ShareAllocator.vue";

const vuetify = createVuetify();

const twoItems = [
  { key: "iOS", label: "iOS" },
  { key: "Android", label: "Android" },
];

const threeItems = [
  { key: "UA", label: "User Acquisition" },
  { key: "UE", label: "User Engagement" },
  { key: "Brand", label: "Brand" },
];

function mountAllocator(
  items: { key: string; label: string }[],
  modelValue: Record<string, number>
) {
  return mount(ShareAllocator, {
    props: { items, modelValue },
    global: { plugins: [vuetify] },
  });
}

describe("ShareAllocator.vue", () => {
  describe("running total display", () => {
    it("shows 'Total 100%' in success color when values sum to 100", () => {
      const wrapper = mountAllocator(twoItems, { iOS: 55, Android: 45 });
      const total = wrapper.find("[data-testid='share-total']");
      expect(total.text()).toContain("100%");
      expect(total.classes()).toContain("text-success");
    });

    it("shows the actual sum and error color when values do not sum to 100", () => {
      const wrapper = mountAllocator(twoItems, { iOS: 60, Android: 20 });
      const total = wrapper.find("[data-testid='share-total']");
      expect(total.text()).toContain("80%");
      expect(total.classes()).toContain("text-error");
    });
  });

  describe("error banner", () => {
    it("is absent when sum is 100", () => {
      const wrapper = mountAllocator(twoItems, { iOS: 70, Android: 30 });
      expect(wrapper.find("[data-testid='share-error']").exists()).toBe(false);
    });

    it("appears when initial modelValue does not sum to 100", () => {
      const wrapper = mountAllocator(twoItems, { iOS: 60, Android: 20 });
      expect(wrapper.find("[data-testid='share-error']").exists()).toBe(true);
    });
  });

  describe("slider rendering", () => {
    it("renders one v-slider per item", () => {
      const wrapper = mountAllocator(threeItems, { UA: 60, UE: 30, Brand: 10 });
      // v-slider renders as a div with class v-slider
      const sliders = wrapper.findAll(".v-slider");
      expect(sliders).toHaveLength(3);
    });

    it("renders item labels", () => {
      const wrapper = mountAllocator(twoItems, { iOS: 55, Android: 45 });
      expect(wrapper.text()).toContain("iOS");
      expect(wrapper.text()).toContain("Android");
    });

    it("displays percentage values next to each slider", () => {
      const wrapper = mountAllocator(twoItems, { iOS: 70, Android: 30 });
      expect(wrapper.text()).toContain("70%");
      expect(wrapper.text()).toContain("30%");
    });
  });

  describe("redistribution on slider change", () => {
    it("emits update:modelValue with redistributed values summing to 100", async () => {
      const wrapper = mountAllocator(twoItems, { iOS: 55, Android: 45 });
      // Simulate the v-slider emitting update:modelValue for the iOS slider.
      // The component listens and calls redistribute internally.
      const iosSlider = wrapper.findAllComponents({ name: "VSlider" })[0];
      await iosSlider.vm.$emit("update:modelValue", 70);

      const emitted = wrapper.emitted("update:modelValue");
      expect(emitted).toBeTruthy();
      const payload = emitted![emitted!.length - 1][0] as Record<string, number>;
      expect(payload.iOS).toBe(70);
      expect(payload.Android).toBe(30);
      expect(Object.values(payload).reduce((a, b) => a + b, 0)).toBe(100);
    });

    it("after redistribution, total display shows 100% in success color", async () => {
      // Start with bad initial values so error is visible.
      const wrapper = mountAllocator(twoItems, { iOS: 60, Android: 20 });
      expect(wrapper.find("[data-testid='share-total']").classes()).toContain("text-error");

      // Simulate moving the iOS slider to 70 → Android becomes 30.
      const iosSlider = wrapper.findAllComponents({ name: "VSlider" })[0];
      await iosSlider.vm.$emit("update:modelValue", 70);
      await wrapper.setProps({ modelValue: { iOS: 70, Android: 30 } });

      expect(wrapper.find("[data-testid='share-total']").classes()).toContain("text-success");
      expect(wrapper.find("[data-testid='share-error']").exists()).toBe(false);
    });
  });

  describe("accessibility", () => {
    it("each v-slider has :name prop equal to the item label (wired to aria-label on the thumb)", () => {
      // VSliderThumb renders role="slider" on a div and sets aria-label from VSlider's `name` prop
      // (verified: VSliderThumb.js line ~140 `"aria-label": props.name`).
      // We assert the prop binding rather than querying rendered ARIA DOM in jsdom.
      const wrapper = mountAllocator(twoItems, { iOS: 55, Android: 45 });
      const sliders = wrapper.findAllComponents({ name: "VSlider" });
      expect(sliders).toHaveLength(2);
      const names = sliders.map((s) => s.props("name"));
      expect(names).toContain("iOS");
      expect(names).toContain("Android");
    });

    it("total region has aria-live=polite", () => {
      const wrapper = mountAllocator(twoItems, { iOS: 55, Android: 45 });
      const live = wrapper.find("[aria-live='polite']");
      expect(live.exists()).toBe(true);
    });
  });
});
```

**4.2** Run the test to confirm it fails:

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/ShareAllocator.spec.ts --environment jsdom
```

Expected: `Error: Failed to resolve import "../ShareAllocator.vue"`. If it passes, something is wrong — stop and investigate.

---

### Task 5 — Implement `ShareAllocator.vue`

**5.1** Create the component:

```text
packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/ShareAllocator.vue
```

```vue
<template>
  <div class="share-allocator">
    <div
      v-for="item in items"
      :key="item.key"
      class="share-allocator__row"
    >
      <span class="share-allocator__label text-body-2 text-medium-emphasis">
        {{ item.label }}
      </span>
      <v-slider
        :model-value="localValues[item.key] ?? 0"
        :min="0"
        :max="100"
        :step="1"
        :name="item.label"
        color="primary"
        hide-details
        class="share-allocator__slider"
        @update:model-value="onSliderChange(item.key, $event as number)"
      />
      <span class="share-allocator__value text-body-2 font-weight-medium">
        {{ localValues[item.key] ?? 0 }}%
      </span>
    </div>

    <div aria-live="polite" class="share-allocator__footer mt-2">
      <span
        data-testid="share-total"
        class="text-caption font-weight-semibold"
        :class="isValid ? 'text-success' : 'text-error'"
      >
        Total {{ total }}%{{ isValid ? "" : " — must total 100%" }}
      </span>
    </div>

    <v-alert
      v-if="!isValid"
      data-testid="share-error"
      type="error"
      variant="tonal"
      density="compact"
      class="mt-2"
      icon="mdi-alert-circle-outline"
    >
      Share allocation must total 100% before you continue.
    </v-alert>
  </div>
</template>

<script setup lang="ts">
import { computed, watch, ref } from "vue";
import { redistribute } from "./redistribute";

export interface ShareAllocatorItem {
  key: string;
  label: string;
}

const props = defineProps<{
  modelValue: Record<string, number>;
  items: ShareAllocatorItem[];
}>();

const emit = defineEmits<{
  (e: "update:modelValue", value: Record<string, number>): void;
}>();

// Local mirror of the prop — updated atomically on redistribution.
// Initialized from prop; kept in sync via watcher.
const localValues = ref<Record<string, number>>({ ...props.modelValue });

watch(
  () => props.modelValue,
  (next) => {
    localValues.value = { ...next };
  }
);

const keys = computed(() => props.items.map((i) => i.key));

const total = computed(() =>
  keys.value.reduce((sum, k) => sum + (localValues.value[k] ?? 0), 0)
);

const isValid = computed(() => total.value === 100);

function onSliderChange(changedKey: string, newValue: number): void {
  const next = redistribute(localValues.value, keys.value, changedKey, newValue);
  localValues.value = next;
  emit("update:modelValue", { ...next });
}
</script>

<style scoped>
.share-allocator__row {
  display: grid;
  grid-template-columns: 80px 1fr 46px;
  align-items: center;
  gap: 12px;
  margin-bottom: 4px;
}

.share-allocator__label {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.share-allocator__value {
  text-align: right;
  font-variant-numeric: tabular-nums;
}
</style>
```

---

### Task 6 — Run all ShareAllocator tests and confirm they pass

**6.1** Run the component tests:

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/ShareAllocator.spec.ts --environment jsdom
```

Expected: all tests pass.

**6.2** Run the redistribute tests to confirm nothing regressed:

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/redistribute.spec.ts --environment jsdom
```

Expected: `6 passed`.

**6.3** Run the full CI suite to confirm no regressions:

```bash
cd /path/to/frontend-mos
npm run test:ci 2>&1 | tail -30
```

Expected: previous pass count unchanged or higher.

---

### Task 7 — Commit

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/ShareAllocator.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/components/ShareAllocator/__tests__/ShareAllocator.spec.ts

git commit -m "feat(onboarding): add ShareAllocator.vue — v-slider sum-to-100 with redistribute + a11y (C2)"
```

---

## Self-review

**Spec coverage check:**

| Requirement | Task covering it |
|---|---|
| `v-slider` (not styled divs) — real Vuetify semantics | Task 5 — `v-slider` with `:min/:max/:step` |
| Moving one proportionally redistributes others | Task 2 (`redistribute`) + Task 5 (`onSliderChange`) |
| Running total — green at 100%, red otherwise | Task 5 (`isValid` computed + `text-success`/`text-error`) |
| Error state when ≠ 100% | Task 5 (`v-alert variant="tonal"`) |
| Error state reachable from bad initial modelValue | Task 4 test "appears when initial modelValue does not sum to 100" |
| Sum-to-100 Vitest | Tasks 1–2 (6 redistribute tests) |
| Redistribute Vitest | Tasks 1–2 |
| Error state Vitest | Task 4 (component tests: total color, banner existence) |
| `role=slider` + `aria-valuenow/min/max` + arrow keys | Provided by Vuetify `v-slider` automatically |
| Per-slider `aria-label` on thumb | Task 5 `:name="item.label"` (Vuetify wires `name` → thumb `aria-label`) + Task 4 a11y test asserts VSlider `name` prop |
| `aria-live="polite"` on total | Task 5 + Task 4 a11y test |
| Colors via `rgb(var(--v-theme-*))` / Vuetify classes | Task 5 — `text-success` / `text-error` Vuetify utility classes |
| AA-compliant tokens (no hardcoded hex) | Task 5 — no hex in template or style |
| Reusable for Step 2 (platformShares) + Step 3 (campaign split) | `items: ShareAllocatorItem[]` + `modelValue: Record<string,number>` |

**Placeholder scan:** No TBDs, no "similar to Task N", no missing code blocks.

**Type consistency:**

- `ShareAllocatorItem` defined once in `ShareAllocator.vue` and exported.
- `redistribute` signature: `(current: Record<string,number>, keys: string[], changedKey: string, newValue: number) => Record<string,number>` — identical in implementation and all test call sites.
- `localValues` type: `Record<string, number>` — consistent with prop, emit, and `redistribute` return.
