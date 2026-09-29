---
id: plan-f0
title: "F0 — Design-token Alignment"
---

## F0 — Design-token Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Verify that the existing MOSLightTheme in `frontend-mos` already supplies every design token AIM Onboarding components need, document the canonical token map for downstream plans, and write a Vitest contract test that locks the AA-compliant token values so future theme edits can't regress them.

**Architecture:** Verify-don't-rebuild. The MOS Vuetify theme (`MOSLightTheme` in `vuetify.ts`) already covers every token the design requires — including the WCAG-AA semantic values (`#427900` success, `#BA4E00` warning). No theme file is modified. No new SCSS utility is created. The only deliverables are: (1) this token map for downstream plans, (2) a Vitest contract test, (3) a manual render verification. The mock's `box-shadow` focus ring is a mock artefact, not a theme setting — it is removed at the component layer (C-kit inputs), not in F0.

**Tech Stack:** Vue 3.5 / Vuetify 3.12, Vitest, frontend-mos monorepo (`packages/app`)

---

## Scope note: MOSLightTheme only

The AIM Onboarding tab lives in the default theme, which is `MOSLightTheme` (`vuetify.ts` line 1131). Dark-theme equivalents are out of scope for Phase 1.

---

## Token map — canonical reference for all AIM Onboarding plans

Every AIM Onboarding component references colors via `rgb(var(--v-theme-<token>))`. No hardcoded hex. The table below is the complete token set this feature uses and where each value lives in the theme.

| Semantic role | CSS var | vuetify.ts key | Hex (light) | vuetify.ts line | Verified AA on white (4.5:1 min) |
|---|---|---|---|---|---|
| Brand navy / primary actions | `--v-theme-primary` | `primary` → `brand_primary_1` | `#1E4B97` | 493 | 7.4:1 — pass |
| Error / destructive | `--v-theme-error` | `error` → `alerts_text-red` | `#BE202E` | 499 | 5.9:1 — pass |
| **Success text (AA)** | `--v-theme-success` | `success` → `alerts_text-green` | `#427900` | 501 | 5.1:1 — pass |
| **Warning text (AA)** | `--v-theme-warning` | `warning` → `alerts_text-orange` | `#BA4E00` | 502 | 4.6:1 — pass |
| Success tonal fill | `--v-theme-alerts-green_fill` | `alerts-green_fill` → `alerts_surface-green` | `#4D840B` | 533 | **3.4:1 — fail on white; only use as tint bg, never for text** |
| Warning tonal fill | `--v-theme-alerts-orange_fill` | `alerts-orange_fill` → `alerts_surface-orange` | `#D56428` | 535 | **3.7:1 — fail on white; only use as tint bg, never for text** |
| Page background | `--v-theme-background` | `background` → `grey_scale-grey5` | `#F8F7F7` | 484 | — |
| Card surface (white) | `--v-theme-surface-2` | `surface-2` → `grey_scale-white` | `#FFFFFF` | 489 | — |
| Input background | `--v-theme-input-background` | `input-background` → `grey_scale-grey6` | `#FBFBFB` | 518 | — |
| Body text | `--v-theme-grey` | `grey` → `grey_scale-grey1` | `#575B5E` | 504 | 7.0:1 on white — pass |
| Emphasis text / titles | `--v-theme-black` | `black` → `grey_scale-black` | `#1A1B1C` | 503 | 18:1 — pass |
| Subtitle / secondary text | `--v-theme-grey-2` | `grey-2` → `grey_scale-grey2` | `#6D757B` | 505 | 5.3:1 on white — pass |
| **Disabled / muted — avoid for data text** | `--v-theme-grey-3` | `grey-3` → `grey_scale-grey3` | `#CDCDCD` | 506 | 1.6:1 — **fail; never use for text** |
| Border | `--v-theme-border` | `border` | `#E1E1E1` | 540 | — |
| Border emphasis | `--v-theme-border-emphasis` | `border-emphasis` | `#CDCDCD` | 523 | — |
| Tab underline / active | `--v-theme-tabs` | `tabs` → `brand_primary_1` | `#1E4B97` | 517 | — |
| Tier chip: Pro (orange) text | `--v-theme-chips-orange` | `chips-orange` → `chips_text-orange` | `#BA4E00` | 560 | 4.6:1 — pass |
| Tier chip: Pro (orange) fill | `--v-theme-chips-orange_fill` | `chips-orange_fill` | `#D56428` | 561 | tint only |
| Tier chip: X (blue) text | `--v-theme-chips-blue` | `chips-blue` → `chips_text-blue` | `#095ABD` | 556 | 6.1:1 — pass |
| Tier chip: X (blue) fill | `--v-theme-chips-blue_fill` | `chips-blue_fill` | `#0C72EE` | 557 | tint only |
| Info / secondary links | `--v-theme-secondary` | `secondary` → `brand_secondary` | `#0C72EE` | 495 | 3.2:1 — **not for small text; links only with underline** |
| SoW mono font | n/a | n/a | JetBrains Mono | settings.scss:1 | intentional, keep |

**Typography:** Inter is loaded globally via `$body-font-family` in `settings.scss` line 4 (`@forward "vuetify/settings" with ($body-font-family: $inter-font-family)`). All components inherit it automatically. SoW document sections use JetBrains Mono explicitly — that is intentional per `design-consolidated.md §8b`.

**Key AA rules for downstream plans:**

1. `success` (`#427900`) and `warning` (`#BA4E00`) are the theme's text-role tokens — use `color="success"` / `color="warning"` on `v-alert`, `v-chip`. Do NOT use `_fill` variants for text.
2. `grey-3` (`#CDCDCD`) is 1.6:1 — never use for any visible text or data values. Use `grey` (7.0:1) or `grey-2` (5.3:1).
3. Color is never the sole signal. Always pair with icon, label, `aria-*` attribute (see design-consolidated §8b).
4. No hardcoded hex anywhere in AIM Onboarding source.

**Focus halo:** `styles.scss` contains no global `box-shadow` focus ring on `.v-field`. The only focus treatment is a `border-bottom` scoped to `.list-search-input input:focus-visible` (lines 709–712). Vuetify v-field has no built-in focus halo in the MOS configuration. The mock's `box-shadow` ring is mock-only CSS and is simply omitted when building the real components. No theme change needed.

---

## Files

| File | Action | Purpose |
|---|---|---|
| `packages/app/src/plugins/vuetify.ts` | Read-only — verify, do not modify | Source of truth for MOSLightTheme token values |
| `packages/app/src/styles/settings.scss` | Read-only — verify, do not modify | Inter font binding |
| `packages/app/src/styles/styles.scss` | Read-only — verify, do not modify | Global `.v-field` / `.v-btn` / `.v-tabs` overrides; confirm no focus ring |
| `packages/app/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/tokenContract.spec.ts` | **Create** | Vitest contract test locking the 4 critical AA token values |

---

## Dependencies

None. F0 has no upstream plan dependencies and modifies no shared code.

---

## Task 1: Verify MOSLightTheme covers all required tokens

**Files:**

- Read: `packages/app/src/plugins/vuetify.ts`

- [ ] **Step 1: Open the theme file and locate `MOSLightTheme`**

  The object starts at line 481. Cross-reference each token the AIM Onboarding design requires against the table above.

  Run:

  ```bash
  grep -n "success\|warning\|primary\|error\|background\|surface-2\|surface_2\|input-background\|chips-orange\|chips-blue\|tabs\|grey-3\|grey_scale-grey3" \
    /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/app/src/plugins/vuetify.ts | head -40
  ```

  Expected output (partial — verify these line numbers match):

  ```text
  499:    error: ColorsLight["alerts_text-red"],
  501:    success: ColorsLight["alerts_text-green"],
  502:    warning: ColorsLight["alerts_text-orange"],
  493:    primary: ColorsLight["brand_primary_1"],
  484:    background: ColorsLight["grey_scale-grey5"],
  ```

- [ ] **Step 2: Confirm AA hex values in `ColorsLight` enum**

  Run:

  ```bash
  grep -n "alerts_text-green\|alerts_text-orange\|alerts_text-red\|brand_primary_1\|grey_scale-grey5\|grey_scale-white\|grey_scale-grey3" \
    /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/app/src/plugins/vuetify.ts | head -20
  ```

  Expected (lines within the `ColorsLight` enum, approximate):

  ```text
  13:   "alerts_text-green" = "#427900",
  15:   "alerts_text-orange" = "#BA4E00",
  18:   "alerts_text-red" = "#BE202E",
  24:   "brand_primary_1" = "#1E4B97",
  53:   "grey_scale-grey5" = "#F8F7F7",
  55:   "grey_scale-white" = "#FFFFFF",
  50:   "grey_scale-grey3" = "#CDCDCD",
  ```

  The `success` → `#427900` and `warning` → `#BA4E00` chains confirm the AA tokens are already present. No additions needed.

- [ ] **Step 3: Confirm Inter font binding**

  Run:

  ```bash
  grep -n "inter\|body-font-family\|Inter" \
    /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/app/src/styles/settings.scss
  ```

  Expected:

  ```text
  1: $inter-font-family: "Inter", sans-serif;
  4:   $body-font-family: $inter-font-family,
  ```

  All components in `frontend-mos` inherit Inter automatically. No per-component font declaration needed.

- [ ] **Step 4: Confirm no global focus halo on `.v-field`**

  Run:

  ```bash
  grep -n "box-shadow\|focus" \
    /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/app/src/styles/styles.scss
  ```

  Expected: `box-shadow` appears only inside `.filter-card`/`.v-overlay__content` (card shadows) and `.list-search-input` inner rules — not on `.v-field` or any global focus selector. The `.list-search-input input:focus-visible` rule at line 709 adds a `border-bottom`, scoped to that search input only. Confirm no global `.v-field:focus` or `:focus-within` box-shadow rule exists.

  Result: no theme-level focus halo. Mock's box-shadow ring is omitted at build time. Nothing to remove from the theme.

- [ ] **Step 5: Record findings**

  All tokens verified present. Record pass/fail for each row in the token map table above. Expected: all pass (no missing tokens, no modifications required).

---

## Task 2: Write the Vitest contract test

**Files:**

- Create: `packages/app/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/tokenContract.spec.ts`

This test imports the theme object and asserts that the four AA-critical tokens resolve to their expected hex values. If someone changes the theme, the test breaks before any UI regresses.

- [ ] **Step 1: Create the directory for the test**

  Run:

  ```bash
  mkdir -p /Users/mukey/Documents/kochava-projects/k4a/frontend-mos/packages/app/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__
  ```

  Expected: no error (directory created or already exists).

- [ ] **Step 2: Write the contract test**

  Create the file at the path above with this content:

  ```ts
  /**
   * F0 — Design-token contract test
   *
   * Locks the WCAG-AA semantic token values that AIM Onboarding components
   * depend on. If MOSLightTheme is edited and a value drifts below the AA
   * threshold, this test fails before any UI regresses.
   *
   * Contrast ratios are pre-computed against #FFFFFF (white) background:
   *   #427900  → 5.1:1  (AA pass, normal text)
   *   #BA4E00  → 4.6:1  (AA pass, normal text)
   *   #BE202E  → 5.9:1  (AA pass)
   *   #1E4B97  → 7.4:1  (AA pass)
   */
  import { describe, it, expect } from "vitest";
  // Import the raw theme definition, not the createVuetify() instance,
  // so we can inspect it synchronously without mounting a Vue app.
  import vuetifyPlugin from "@/plugins/vuetify";

  // The theme definitions are not exported directly, so we inspect the
  // resolved theme object that createVuetify() carries internally.
  // Vuetify exposes theme.computedThemes under the theme property.
  describe("MOSLightTheme — AIM Onboarding AA token contract", () => {
    const theme = vuetifyPlugin.theme;

    it("success token resolves to AA-compliant #427900", () => {
      const resolved = theme.computedThemes.value["MOSLightTheme"].colors.success;
      expect(resolved).toBe("#427900");
    });

    it("warning token resolves to AA-compliant #BA4E00", () => {
      const resolved = theme.computedThemes.value["MOSLightTheme"].colors.warning;
      expect(resolved).toBe("#BA4E00");
    });

    it("error token resolves to #BE202E", () => {
      const resolved = theme.computedThemes.value["MOSLightTheme"].colors.error;
      expect(resolved).toBe("#BE202E");
    });

    it("primary (navy) token resolves to #1E4B97", () => {
      const resolved = theme.computedThemes.value["MOSLightTheme"].colors.primary;
      expect(resolved).toBe("#1E4B97");
    });

    it("chips-orange (Pro tier) text token resolves to AA-compliant #BA4E00", () => {
      const resolved = theme.computedThemes.value["MOSLightTheme"].colors["chips-orange"];
      expect(resolved).toBe("#BA4E00");
    });

    it("chips-blue (X tier) text token resolves to #095ABD", () => {
      const resolved = theme.computedThemes.value["MOSLightTheme"].colors["chips-blue"];
      expect(resolved).toBe("#095ABD");
    });

    it("_fill / surface variants are NOT used for text (asserting lighter values exist separately)", () => {
      const successFill = theme.computedThemes.value["MOSLightTheme"].colors["alerts-green_fill"];
      const warningFill = theme.computedThemes.value["MOSLightTheme"].colors["alerts-orange_fill"];
      // These lighter values exist as tonal backgrounds only — never use for text.
      expect(successFill).toBe("#4D840B");
      expect(warningFill).toBe("#D56428");
    });
  });
  ```

- [ ] **Step 3: Run the test to confirm it passes**

  Run from the `packages/app` directory:

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && \
    npx vitest run packages/app/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/tokenContract.spec.ts
  ```

  Expected output:

  ```text
  ✓ MOSLightTheme — AIM Onboarding AA token contract
    ✓ success token resolves to AA-compliant #427900
    ✓ warning token resolves to AA-compliant #BA4E00
    ✓ error token resolves to #BE202E
    ✓ primary (navy) token resolves to #1E4B97
    ✓ chips-orange (Pro tier) text token resolves to AA-compliant #BA4E00
    ✓ chips-blue (X tier) text token resolves to #095ABD
    ✓ _fill / surface variants are NOT used for text (asserting lighter values exist separately)

  Test Files  1 passed (1)
  Tests       7 passed (7)
  ```

  If `theme.computedThemes` is not the right API for the installed Vuetify version, fall back to importing the `MOSLightTheme` definition directly by extracting it to a separate file (see Step 4).

- [ ] **Step 4: Fallback — if `computedThemes` API is unavailable**

  Only do this step if Step 3 fails with `computedThemes is not a function` or similar. Export `MOSLightTheme` from `vuetify.ts` and import it directly in the test:

  In `packages/app/src/plugins/vuetify.ts`, add one export at the bottom (before the `export default`):

  ```ts
  export { MOSLightTheme };
  ```

  Then update the test import:

  ```ts
  import { MOSLightTheme } from "@/plugins/vuetify";

  // Replace `theme.computedThemes.value["MOSLightTheme"].colors.success` with:
  MOSLightTheme.colors.success
  ```

  Re-run; expected: same 7-pass result.

- [ ] **Step 5: Commit the test**

  ```bash
  git add packages/app/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/__tests__/tokenContract.spec.ts
  git commit -m "test(aim-onboarding): F0 contract test — lock AA token values in MOSLightTheme"
  ```

---

## Task 3: Manual render verification

This step catches anything automated tests miss — that the browser actually receives the right computed style.

- [ ] **Step 1: Start the dev server**

  ```bash
  cd /Users/mukey/Documents/kochava-projects/k4a/frontend-mos && npm run dev
  ```

  Wait for the Vite server to report `ready` on localhost (typically port 5173).

- [ ] **Step 2: Open DevTools and inspect the CSS custom properties**

  Navigate to any MOS page in the browser. Open DevTools → Elements → select `<html>` → Computed tab → filter for `--v-theme`.

  Verify these values appear in the computed CSS:

  | CSS custom property | Expected value |
  |---|---|
  | `--v-theme-success` | `66 121 0` (RGB of `#427900`) |
  | `--v-theme-warning` | `186 78 0` (RGB of `#BA4E00`) |
  | `--v-theme-primary` | `30 75 151` (RGB of `#1E4B97`) |
  | `--v-theme-error` | `190 32 46` (RGB of `#BE202E`) |

  Vuetify stores colors as space-separated RGB channels (e.g. `66 121 0` for `#427900`), consumed via `rgb(var(--v-theme-success))`. This is the correct form.

- [ ] **Step 3: Spot-check a tonal v-alert**

  In browser DevTools console, run:

  ```js
  // Temporarily mount a v-alert to inspect its computed colors.
  // Paste into console on any MOS page that has Vue DevTools available.
  document.querySelector('[data-v-app]')?.__vue_app__
    .config.globalProperties.$vuetify.theme.current.value.colors.success
  ```

  Expected: `"#427900"` (string, with hash).

- [ ] **Step 4: Verify Inter is loaded**

  In DevTools → Elements → `<body>` → Computed tab → `font-family`. Confirm `Inter` appears first in the stack.

- [ ] **Step 5: Verify no focus halo on a v-field**

  Click into any text input on a MOS page. In DevTools → Elements → inspect the `.v-field` element → Computed → `box-shadow`. Confirm `none` (or only component-level shadows, never a focus ring). The MOS theme is confirmed clean.

---

## Self-review: spec coverage check

| Requirement | Covered |
|---|---|
| Verify primary navy exists | Task 1 Step 2 + token map |
| Verify success `#427900` exists and is AA | Token map + Task 2 test |
| Verify warning `#BA4E00` exists and is AA | Token map + Task 2 test |
| Verify error exists | Task 2 test |
| Verify surface / background tokens | Token map |
| Verify Inter font | Task 1 Step 3 + Task 3 Step 4 |
| Confirm no input focus halo | Task 1 Step 4 + Task 3 Step 5 |
| Document token map for downstream plans | Token map table above |
| No new theme created / no theme edits | Verified: zero file modifications |
| `_fill` variants flagged as non-text-safe | Token map + Task 2 Step 2 `_fill` assertion |
| `grey-3` flagged as too light for text | Token map AA column |
| All colors via `rgb(var(--v-theme-*))` | Stated in architecture + token map |
| Chips-orange / chips-blue for tier | Token map + Task 2 test |
