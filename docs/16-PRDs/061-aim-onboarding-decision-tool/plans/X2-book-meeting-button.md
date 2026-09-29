---
id: plan-x2
title: "X2 — Book-my-meeting Button"
---

## X2 — Book-my-meeting Button

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the "Book my onboarding meeting" button to the SoW output header and implement the four-state machine — `idle → sending → sent (200) | error (failure)` — with retry on failure.

**Architecture:** A single `BookMeetingButton.vue` component drives the state machine with a `ref<'idle'|'sending'|'sent'|'error'>`. It calls `onboardingService.bookMeeting` (F2) and renders distinct feedback for each state. The component lives in `output/` alongside `SowTab.vue`, which places it in the SoW output header. No Vuex/Pinia store mutation — only local reactive state. No `notificationsStore` (those are CMS-bound global toasts; this is an inline, persistent confirmation per spec mermaid).

**Tech Stack:** Vue 3.5 / TypeScript 5 / Vuetify 3.12, Vitest + `@vue/test-utils`, `onboardingService` from F2.

---

## Files

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/BookMeetingButton.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/BookMeetingButton.spec.ts` |
| Modify | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue` — add `<BookMeetingButton>` to the output header |

**Component path anchor:** All sibling output components (`SowTab.vue`, `TimelineTab.vue`, `DataSchemaTab.vue`) live in:
`packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/`

**Service import path** (from `output/`): `../services/onboarding`

**Interface import path** (from `output/`): `../interfaces/aimOnboarding`

---

## Dependencies

| Plan | Why |
|------|-----|
| **F2** — Service + interfaces | `onboardingService.bookMeeting(advertiserId, sessionId)` |
| **B9** — Book-meeting email | Backend endpoint; mock in tests, real in integration |

**Pending (does not block dev):** Email copy / recipient list — Gary. Use placeholder copy marked with a comment until Gary signs off.

---

## State machine

```text
idle  ──[click]──→  sending  ──[200 OK]──→  sent
                        └──[error/timeout]──→  error  ──[click]──→  sending
```

| State | Button label | Button disabled | Feedback |
|-------|-------------|-----------------|---------|
| `idle` | "Book my onboarding meeting" | `false` | none |
| `sending` | "Sending…" | `true` | none |
| `sent` | "Sent ✓" | `true` | success alert: "Email sent to your onboarding team. Expect a reply within 1 business day." |
| `error` | "Try again" | `false` | error alert: "We could not send your request. Please try again or contact support." |

Token mapping (design-consolidated §8b / product-spec-v2 §6):

- Success alert → `v-alert variant="tonal" type="success"` (token `--alert-green #427900`)
- Error alert → `v-alert variant="tonal" type="error"` (token `--alert-red #BE202E`)
- Button in `idle`/`error` state → `v-btn color="primary"` (variant `outlined` to match the SoW header's secondary actions)
- Button in `sent` state → `v-btn color="success"` variant `tonal` (non-interactive)

---

## Task 1: Failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/BookMeetingButton.spec.ts`

- [ ] **Step 1: Write all failing tests**

```ts
// output/__tests__/BookMeetingButton.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { mount, flushPromises } from '@vue/test-utils';
import { createVuetify } from 'vuetify';
import * as components from 'vuetify/components';
import * as directives from 'vuetify/directives';

// Mock the service BEFORE importing the component so the module resolution lands on the mock
vi.mock('../../services/onboarding', () => ({
  onboardingService: {
    bookMeeting: vi.fn(),
  },
}));

import BookMeetingButton from '../BookMeetingButton.vue';
import { onboardingService } from '../../services/onboarding';

const vuetify = createVuetify({ components, directives });

const defaultProps = {
  advertiserId: 'adv-001',
  sessionId: 'sess-abc',
};

function mountComponent(props = defaultProps) {
  return mount(BookMeetingButton, {
    props,
    global: { plugins: [vuetify] },
  });
}

beforeEach(() => {
  vi.clearAllMocks();
});

describe('BookMeetingButton', () => {
  it('idle state — renders the primary button with correct label', () => {
    const wrapper = mountComponent();
    expect(wrapper.find('[data-testid="book-meeting-btn"]').text()).toContain(
      'Book my onboarding meeting',
    );
    expect(wrapper.find('[data-testid="book-meeting-btn"]').attributes('disabled')).toBeUndefined();
    expect(wrapper.find('[data-testid="sent-alert"]').exists()).toBe(false);
    expect(wrapper.find('[data-testid="error-alert"]').exists()).toBe(false);
  });

  it('sending state — button is disabled while request is in-flight', async () => {
    // Never resolves during this test so we can observe the sending state
    vi.mocked(onboardingService.bookMeeting).mockReturnValue(new Promise(() => {}));
    const wrapper = mountComponent();
    await wrapper.find('[data-testid="book-meeting-btn"]').trigger('click');
    await wrapper.vm.$nextTick();
    expect(wrapper.find('[data-testid="book-meeting-btn"]').attributes('disabled')).toBeDefined();
    expect(wrapper.find('[data-testid="book-meeting-btn"]').text()).toContain('Sending');
  });

  it('sent state — shows success alert and locked button on 200 OK', async () => {
    vi.mocked(onboardingService.bookMeeting).mockResolvedValue({ status: 200 } as any);
    const wrapper = mountComponent();
    await wrapper.find('[data-testid="book-meeting-btn"]').trigger('click');
    await flushPromises();
    expect(wrapper.find('[data-testid="sent-alert"]').exists()).toBe(true);
    expect(wrapper.find('[data-testid="sent-alert"]').text()).toContain(
      'Expect a reply within 1 business day',
    );
    expect(wrapper.find('[data-testid="book-meeting-btn"]').text()).toContain('Sent');
    expect(wrapper.find('[data-testid="book-meeting-btn"]').attributes('disabled')).toBeDefined();
    expect(wrapper.find('[data-testid="error-alert"]').exists()).toBe(false);
  });

  it('error state — shows error alert and re-enabled "Try again" button on failure', async () => {
    vi.mocked(onboardingService.bookMeeting).mockRejectedValue(new Error('Network error'));
    const wrapper = mountComponent();
    await wrapper.find('[data-testid="book-meeting-btn"]').trigger('click');
    await flushPromises();
    expect(wrapper.find('[data-testid="error-alert"]').exists()).toBe(true);
    expect(wrapper.find('[data-testid="error-alert"]').text()).toContain(
      'We could not send your request',
    );
    expect(wrapper.find('[data-testid="book-meeting-btn"]').text()).toContain('Try again');
    expect(wrapper.find('[data-testid="book-meeting-btn"]').attributes('disabled')).toBeUndefined();
    expect(wrapper.find('[data-testid="sent-alert"]').exists()).toBe(false);
  });

  it('retry — transitions error → sending → sent on second click', async () => {
    vi.mocked(onboardingService.bookMeeting)
      .mockRejectedValueOnce(new Error('first failure'))
      .mockResolvedValueOnce({ status: 200 } as any);

    const wrapper = mountComponent();
    // First click → error
    await wrapper.find('[data-testid="book-meeting-btn"]').trigger('click');
    await flushPromises();
    expect(wrapper.find('[data-testid="error-alert"]').exists()).toBe(true);

    // Second click → sent
    await wrapper.find('[data-testid="book-meeting-btn"]').trigger('click');
    await flushPromises();
    expect(wrapper.find('[data-testid="sent-alert"]').exists()).toBe(true);
    expect(wrapper.find('[data-testid="error-alert"]').exists()).toBe(false);
  });

  it('passes correct advertiserId and sessionId to bookMeeting', async () => {
    vi.mocked(onboardingService.bookMeeting).mockResolvedValue({ status: 200 } as any);
    const wrapper = mountComponent({ advertiserId: 'adv-999', sessionId: 'sess-xyz' });
    await wrapper.find('[data-testid="book-meeting-btn"]').trigger('click');
    await flushPromises();
    expect(onboardingService.bookMeeting).toHaveBeenCalledWith('adv-999', 'sess-xyz');
  });
});
```

- [ ] **Step 2: Run — confirm all 6 tests fail**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/BookMeetingButton.spec.ts
```

Expected: 6 tests fail. Error: `Cannot find module '../BookMeetingButton.vue'`.

- [ ] **Step 3: Commit failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/BookMeetingButton.spec.ts
git commit -m "test(aim-onboarding): add failing BookMeetingButton tests (X2)"
```

---

## Task 2: Component implementation

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/BookMeetingButton.vue`

- [ ] **Step 1: Write the component**

```vue
<!-- output/BookMeetingButton.vue -->
<!-- PENDING: button label, alert copy — await Gary sign-off before production -->
<template>
  <div class="d-flex flex-column gap-3">
    <v-btn
      data-testid="book-meeting-btn"
      :color="buttonColor"
      :variant="buttonVariant"
      :disabled="state === 'sending' || state === 'sent'"
      :loading="state === 'sending'"
      @click="book"
    >
      <v-icon v-if="state === 'sent'" start>mdi-check</v-icon>
      {{ buttonLabel }}
    </v-btn>

    <v-alert
      v-if="state === 'sent'"
      data-testid="sent-alert"
      type="success"
      variant="tonal"
      density="compact"
    >
      <!-- TODO: replace with Gary-approved copy -->
      Email sent to your onboarding team. Expect a reply within 1 business day.
    </v-alert>

    <v-alert
      v-if="state === 'error'"
      data-testid="error-alert"
      type="error"
      variant="tonal"
      density="compact"
    >
      <!-- TODO: replace with Gary-approved copy -->
      We could not send your request. Please try again or contact support.
    </v-alert>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';
import { onboardingService } from '../services/onboarding';

type BookState = 'idle' | 'sending' | 'sent' | 'error';

const props = defineProps<{
  advertiserId: string;
  sessionId: string;
}>();

const state = ref<BookState>('idle');

const buttonLabel = computed(() => {
  switch (state.value) {
    case 'sending': return 'Sending…';
    case 'sent':    return 'Sent ✓';
    case 'error':   return 'Try again';
    default:        return 'Book my onboarding meeting';
  }
});

const buttonColor = computed(() =>
  state.value === 'sent' ? 'success' : 'primary',
);

const buttonVariant = computed(() =>
  state.value === 'sent' ? 'tonal' : 'outlined',
);

async function book(): Promise<void> {
  if (state.value === 'sending' || state.value === 'sent') return;
  state.value = 'sending';
  try {
    await onboardingService.bookMeeting(props.advertiserId, props.sessionId);
    state.value = 'sent';
  } catch {
    state.value = 'error';
  }
}
</script>
```

- [ ] **Step 2: Run tests — confirm all 6 pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/BookMeetingButton.spec.ts
```

Expected output:

```text
 ✓ BookMeetingButton > idle state — renders the primary button with correct label
 ✓ BookMeetingButton > sending state — button is disabled while request is in-flight
 ✓ BookMeetingButton > sent state — shows success alert and locked button on 200 OK
 ✓ BookMeetingButton > error state — shows error alert and re-enabled "Try again" button on failure
 ✓ BookMeetingButton > retry — transitions error → sending → sent on second click
 ✓ BookMeetingButton > passes correct advertiserId and sessionId to bookMeeting

Test Files  1 passed (1)
Tests       6 passed (6)
```

- [ ] **Step 3: Verify TypeScript**

```bash
npx tsc --noEmit --project packages/advertiser/tsconfig.json \
  2>&1 | grep -i "BookMeeting\|book-meeting"
```

Expected: no output.

- [ ] **Step 4: Commit the component**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/BookMeetingButton.vue
git commit -m "feat(aim-onboarding): add BookMeetingButton component with sent/retry UX (X2)"
```

---

## Task 3: Wire into SoW output header

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue`

- [ ] **Step 1: Read SowTab.vue** (required before editing)

```bash
cat packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue
```

Locate the output header block. In the ui-preview.html the header is:

```html
<div class="ohead">
  <div class="name">BrightFit</div>
  <span class="sp"></span>
  <span class="chip-pro">AIM Pro</span>
  <button class="btn btn-o">Edit answers</button>
  <!-- Book my onboarding meeting goes here -->
</div>
```

In the Vue component this will be a `<div>` with `class="d-flex align-center"` (or equivalent) wrapping the company name, tier chip, and action buttons. Add `<BookMeetingButton>` after the tier chip and before the Edit-answers button (or at the right edge of the header row — follow the existing layout of the header block exactly).

- [ ] **Step 2: Import and register the component**

Add at the top of `<script setup>` in `SowTab.vue`:

```ts
import BookMeetingButton from './BookMeetingButton.vue';
```

- [ ] **Step 3: Add the component to the header template**

The component accepts `advertiserId` and `sessionId`. These must be passed down from the composable / parent. In SowTab the session is available via the `useAimOnboarding` composable (or props from `AimOnboardingTab.vue`). Confirm the actual prop/composable name by reading SowTab.vue in Step 1; then wire accordingly.

Example (adjust variable names to match what SowTab.vue actually has):

```vue
<BookMeetingButton
  :advertiser-id="advertiserId"
  :session-id="sessionId"
/>
```

- [ ] **Step 4: Run the full test suite to confirm no regressions**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab
```

Expected: all tests in the AimOnboardingTab tree pass.

- [ ] **Step 5: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowTab.vue
git commit -m "feat(aim-onboarding): wire BookMeetingButton into SoW header (X2)"
```

---

## Plan dependencies

| ID | Plan | What X2 needs |
|----|------|---------------|
| F2 | Service + interfaces | `onboardingService.bookMeeting` — tests mock it; real calls need the service implemented |
| B9 | Book-meeting email | Backend endpoint — mocked in unit tests; required for integration / QA |

---

## Self-review

**Spec coverage:**

- product-spec-v2 §4.2 Tab 1 mermaid: `idle → sending → sent (200) | error (failure) → retry` — all four states implemented and tested.
- Confirmation copy: "Email sent to your onboarding team. Expect a reply within 1 business day." — matches spec.
- Error copy: "We could not send your request. Please try again or contact support." — matches spec.
- Retry path: `error → sending → sent` — covered by dedicated retry test.
- Recipient list in env/config (D1) — no frontend concern; handled by B9.

**Placeholder scan:** All code blocks are complete. One `<!-- TODO: replace with Gary-approved copy -->` comment is intentional and flagged — it is a pending item (Gary), not a placeholder standing in for missing logic.

**Type consistency:** `advertiserId: string` / `sessionId: string` props in Task 1 test match `defineProps` in Task 2 component. `onboardingService.bookMeeting(advertiserId, sessionId)` call site in component matches the F2 service signature `bookMeeting(advertiserId: string, id: string)` (note: F2 uses `id` as the parameter name, X2 uses `sessionId` as the prop name — the call passes `props.sessionId` as the second argument, which is correct).
