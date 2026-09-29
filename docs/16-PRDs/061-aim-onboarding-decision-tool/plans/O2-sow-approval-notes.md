---
id: plan-o2
title: "O2 — SoW Approval + Notes"
---

## O2 — SoW Approval + Notes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the SoW approval block (`SowApprovalBlock.vue`) and per-section inline note textareas (`SowSectionNote.vue`) to the Scope of Work tab — name + job-title inputs, "Approve Scope of Work" button, terminal locked state, and always-editable section notes that autosave with the wizard state and appear in the PDF.

**Architecture:** Two focused SFCs. `SowApprovalBlock.vue` owns the approve form, the approval banner, and the disabled-until-ready guard (requires `csmApproved === true` **and** both name + title filled). `SowSectionNote.vue` is a single-textarea wrapper keyed by section. Both bind into `useAimOnboarding`'s `scopeOfWork` ref via `patchSoW()`. On submit, `SowApprovalBlock` calls `onboardingService.sowApproval()` with name + title + a client-generated `ApprovedAt` ISO timestamp, then patches local state from the values it sent (B7 returns 204 — no body). The locked state is derived from `scopeOfWork.ApprovedAt !== null`. Notes remain editable in all states; they ride F4 autosave pre-approval (post-approval autosave gap is noted below). Neither component handles CSM actions, PDF, or portal routing — those are X1 and O3 respectively.

**Tech Stack:** Vue 3.5 / TypeScript 5, Vuetify 3.12, `@mos/core` (`InputText`), Vitest + @vue/test-utils.

---

## Files

| Action | Path |
|--------|------|
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowApprovalBlock.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowSectionNote.vue` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowApprovalBlock.spec.ts` |
| Create | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowSectionNote.spec.ts` |

**Integration contract with O1 (`SowTab.vue`, future plan):**

O1 will mount these components inside `SowTab.vue`. Use the following `SectionNote` keys — derived from spec §4.2 Tab 1 — when wiring the ten `<SowSectionNote>` instances:

| Key | SoW section |
|-----|-------------|
| `engagement` | Engagement details |
| `platformRegion` | Platform & region |
| `campaignTypes` | Campaign types |
| `adChannels` | Client ad channels |
| `businessKpi` | Business model & KPI |
| `funnel` | Funnel events |
| `dataSources` | Data sources |
| `externalFactors` | External factors |
| `modelStructure` | Model structure |
| `objectives` | Objectives |

O1 mounts `<SowApprovalBlock />` once, after all sections, at the foot of the `.paper` document.

---

## Dependencies

**O1** — `SowTab.vue` is where these components render. O2 delivers independently-testable SFCs; O1 mounts them. O2 ships before O1 and O1 imports it.

**F2** — `interfaces/aimOnboarding.ts` defines `ScopeOfWork`, `SowApprovalPayload`, `OnboardingSession`. Verify before implementing:

- `SowApprovalPayload` must include `ApprovedAt: string` (B7 DTO marks it `[Required]`). F2 currently defines only `ApproverName`/`ApproverJobTitle`. Coordinate: add `ApprovedAt: string` to `SowApprovalPayload` in `interfaces/aimOnboarding.ts` before Task 3 of this plan.

**F3** — `composables/useAimOnboarding.ts` must expose `{ scopeOfWork, patchSoW, advertiserId, sessionId }`. Mock this composable in tests (same pattern as W1).

**B7** — `PUT /{id}/sow-approval` accepts `{ApproverName, ApproverJobTitle, ApprovedAt}` and returns **204 No Content** on success; **409 Conflict** if already approved. There is no response body — patch local state from the values sent.

---

## Design notes for the implementer

### The `csmApproved` gate

The approve button must stay disabled until **both** conditions hold: (1) `scopeOfWork.CsmApproved === true` (a CSM has activated signoff — set by X1's CSM Approve button calling `POST /{id}/approve`), **and** (2) both name and job title are non-empty strings. If `csmApproved` is false, the block shows a "Waiting for your onboarding team to activate approval…" caption below the inputs. Tests cover both conditions separately.

### Approval is terminal — no Unlock

The ui-preview.html line 426 shows a CSM "Unlock" button. That button was removed by the 2026-06-11 design decision (§8a, §6 API contract). **Do not implement Unlock.** Post-approval the wizard answers lock; inline notes remain editable; the approval banner is permanent.

### Post-approval note persistence gap

`PUT /{id}` (autosave) returns 409 once `approvedAt` is set — so note changes entered after approval cannot be saved via F4's autosave. This is a known Phase 1 limitation. Do not build a workaround here. Keep the textareas enabled; silence the 409 gracefully in F4. Call this out in a `<!-- TODO: Phase 2 — notes-only endpoint needed post-approval -->` comment in `SowSectionNote.vue`.

### Inputs: `InputText`, not styled divs

Use `InputText` from `@mos/core` (wraps `v-text-field`, 8px radius, no focus-halo via global SCSS). Apply `aria-required="true"` on both inputs. The preview's `.inp` HTML element and placeholder-only labels (line 374) are accessibility failures per §8b — do not copy them. Use the `label` prop, not `placeholder` as label.

### Approved-by banner

On success, replace the form with a `v-alert variant="tonal" color="success"` (design token `--alert-green #427900`). Content: `Approved by {approverName} ({approverJobTitle}) on {formattedDate}`. Format `approvedAt` as `D MMM YYYY` using `Intl.DateTimeFormat` — no external date library.

### Section note autosave

`SowSectionNote` calls `patchSoW({ SectionNotes: { ...existing, [sectionKey]: value } })` on each `@input`. F4's debounced PUT flush carries the update to Mongo. No separate save button per note.

---

## Task 1 — Failing tests: `SowApprovalBlock`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowApprovalBlock.spec.ts`

- [ ] **Step 1: Write the spec file**

```ts
// output/__tests__/SowApprovalBlock.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import { ref } from 'vue'
import SowApprovalBlock from '../SowApprovalBlock.vue'

// ── Mock useAimOnboarding ────────────────────────────────────────────────────

const mockScopeOfWork = ref({
  SectionNotes: {} as Record<string, string>,
  ApproverName: '',
  ApproverJobTitle: '',
  ApprovedAt: null as string | null,
  CsmApproved: false,
})

const mockPatchSoW = vi.fn((patch: object) => {
  Object.assign(mockScopeOfWork.value, patch)
})

vi.mock('../../composables/useAimOnboarding', () => ({
  useAimOnboarding: () => ({
    scopeOfWork: mockScopeOfWork,
    patchSoW: mockPatchSoW,
    advertiserId: ref('adv-001'),
    sessionId: ref('sess-abc'),
  }),
}))

// ── Mock onboardingService ───────────────────────────────────────────────────

vi.mock('../../services/onboarding', () => ({
  onboardingService: {
    sowApproval: vi.fn().mockResolvedValue({ status: 204 }),
  },
}))

import { onboardingService } from '../../services/onboarding'

// ── Helper ───────────────────────────────────────────────────────────────────

function mountBlock() {
  return mount(SowApprovalBlock, {
    global: {
      plugins: [createVuetify()],
    },
  })
}

// ── Tests ────────────────────────────────────────────────────────────────────

describe('SowApprovalBlock', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockScopeOfWork.value = {
      SectionNotes: {},
      ApproverName: '',
      ApproverJobTitle: '',
      ApprovedAt: null,
      CsmApproved: false,
    }
  })

  it('renders two labeled inputs — name and job title', () => {
    const wrapper = mountBlock()
    // Labels must be visible text, not just placeholders
    expect(wrapper.text()).toContain('Full name')
    expect(wrapper.text()).toContain('Job title')
  })

  it('approve button is disabled when name and title are empty', () => {
    const wrapper = mountBlock()
    const btn = wrapper.find('[data-testid="approve-btn"]')
    expect(btn.attributes('disabled')).toBeDefined()
  })

  it('approve button stays disabled when csmApproved is false even with name + title', async () => {
    mockScopeOfWork.value.CsmApproved = false
    const wrapper = mountBlock()
    await wrapper.find('[data-testid="name-input"] input').setValue('Jane Doe')
    await wrapper.find('[data-testid="title-input"] input').setValue('CMO')
    const btn = wrapper.find('[data-testid="approve-btn"]')
    expect(btn.attributes('disabled')).toBeDefined()
  })

  it('shows "waiting for CSM" caption when csmApproved is false', () => {
    const wrapper = mountBlock()
    expect(wrapper.text()).toContain('Waiting for your onboarding team to activate approval')
  })

  it('approve button is enabled when csmApproved is true and both fields filled', async () => {
    mockScopeOfWork.value.CsmApproved = true
    const wrapper = mountBlock()
    await wrapper.find('[data-testid="name-input"] input').setValue('Jane Doe')
    await wrapper.find('[data-testid="title-input"] input').setValue('CMO')
    const btn = wrapper.find('[data-testid="approve-btn"]')
    expect(btn.attributes('disabled')).toBeUndefined()
  })

  it('calls onboardingService.sowApproval with name, title, and a client-generated ApprovedAt', async () => {
    mockScopeOfWork.value.CsmApproved = true
    const wrapper = mountBlock()
    await wrapper.find('[data-testid="name-input"] input').setValue('Jane Doe')
    await wrapper.find('[data-testid="title-input"] input').setValue('CMO')
    await wrapper.find('[data-testid="approve-btn"]').trigger('click')
    expect(onboardingService.sowApproval).toHaveBeenCalledWith(
      'adv-001',
      'sess-abc',
      expect.objectContaining({
        ApproverName: 'Jane Doe',
        ApproverJobTitle: 'CMO',
        ApprovedAt: expect.any(String),
      })
    )
    // ApprovedAt must be a valid ISO string
    const payload = (onboardingService.sowApproval as ReturnType<typeof vi.fn>).mock.calls[0][2]
    expect(() => new Date(payload.ApprovedAt)).not.toThrow()
  })

  it('on success: patches scopeOfWork and shows approved banner with name, title, date', async () => {
    mockScopeOfWork.value.CsmApproved = true
    const wrapper = mountBlock()
    await wrapper.find('[data-testid="name-input"] input').setValue('Jane Doe')
    await wrapper.find('[data-testid="title-input"] input').setValue('CMO')
    await wrapper.find('[data-testid="approve-btn"]').trigger('click')
    await wrapper.vm.$nextTick()
    expect(wrapper.find('[data-testid="approved-banner"]').exists()).toBe(true)
    expect(wrapper.text()).toContain('Approved by Jane Doe (CMO)')
    // Form inputs should no longer be present
    expect(wrapper.find('[data-testid="approve-btn"]').exists()).toBe(false)
  })

  it('on 409: shows "already approved" error, does not crash', async () => {
    ;(onboardingService.sowApproval as ReturnType<typeof vi.fn>).mockRejectedValueOnce({
      response: { status: 409 },
    })
    mockScopeOfWork.value.CsmApproved = true
    const wrapper = mountBlock()
    await wrapper.find('[data-testid="name-input"] input').setValue('Jane Doe')
    await wrapper.find('[data-testid="title-input"] input').setValue('CMO')
    await wrapper.find('[data-testid="approve-btn"]').trigger('click')
    await wrapper.vm.$nextTick()
    expect(wrapper.text()).toContain('already been approved')
  })

  it('in locked state: shows approval banner and hides the form', async () => {
    mockScopeOfWork.value = {
      SectionNotes: {},
      ApproverName: 'Jane Doe',
      ApproverJobTitle: 'CMO',
      ApprovedAt: '2026-06-15T10:00:00.000Z',
      CsmApproved: true,
    }
    const wrapper = mountBlock()
    expect(wrapper.find('[data-testid="approved-banner"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="approve-btn"]').exists()).toBe(false)
    expect(wrapper.text()).toContain('Approved by Jane Doe (CMO)')
  })
})
```

- [ ] **Step 2: Run tests — expect compile failure (SowApprovalBlock.vue does not exist)**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowApprovalBlock.spec.ts 2>&1 | tail -20
```

Expected: error referencing missing module `../SowApprovalBlock`.

- [ ] **Step 3: Commit failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowApprovalBlock.spec.ts
git commit -m "test(aim-onboarding): add failing SowApprovalBlock tests (O2)"
```

---

## Task 2 — Failing tests: `SowSectionNote`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowSectionNote.spec.ts`

- [ ] **Step 1: Write the spec file**

```ts
// output/__tests__/SowSectionNote.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import { ref } from 'vue'
import SowSectionNote from '../SowSectionNote.vue'

const mockScopeOfWork = ref({
  SectionNotes: {} as Record<string, string>,
  ApproverName: '',
  ApproverJobTitle: '',
  ApprovedAt: null as string | null,
  CsmApproved: false,
})

const mockPatchSoW = vi.fn((patch: { SectionNotes: Record<string, string> }) => {
  mockScopeOfWork.value.SectionNotes = patch.SectionNotes
})

vi.mock('../../composables/useAimOnboarding', () => ({
  useAimOnboarding: () => ({
    scopeOfWork: mockScopeOfWork,
    patchSoW: mockPatchSoW,
  }),
}))

function mountNote(sectionKey = 'engagement') {
  return mount(SowSectionNote, {
    props: { sectionKey },
    global: { plugins: [createVuetify()] },
  })
}

describe('SowSectionNote', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockScopeOfWork.value = {
      SectionNotes: {},
      ApproverName: '',
      ApproverJobTitle: '',
      ApprovedAt: null,
      CsmApproved: false,
    }
  })

  it('renders a textarea', () => {
    const wrapper = mountNote()
    expect(wrapper.find('textarea').exists()).toBe(true)
  })

  it('initialises textarea value from SectionNotes[sectionKey]', () => {
    mockScopeOfWork.value.SectionNotes = { engagement: 'Previous note' }
    const wrapper = mountNote('engagement')
    expect((wrapper.find('textarea').element as HTMLTextAreaElement).value).toBe('Previous note')
  })

  it('calls patchSoW with updated SectionNotes map on input', async () => {
    mockScopeOfWork.value.SectionNotes = { platformRegion: 'Old note' }
    const wrapper = mountNote('engagement')
    await wrapper.find('textarea').setValue('New note')
    expect(mockPatchSoW).toHaveBeenCalledWith({
      SectionNotes: { platformRegion: 'Old note', engagement: 'New note' },
    })
  })

  it('textarea remains editable when approvedAt is set (locked state)', async () => {
    mockScopeOfWork.value.ApprovedAt = '2026-06-15T10:00:00.000Z'
    const wrapper = mountNote()
    const textarea = wrapper.find('textarea').element as HTMLTextAreaElement
    expect(textarea.disabled).toBe(false)
  })

  it('renders with a visible "Notes" label above the textarea', () => {
    const wrapper = mountNote()
    expect(wrapper.text()).toContain('Notes')
  })
})
```

- [ ] **Step 2: Run tests — expect compile failure**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowSectionNote.spec.ts 2>&1 | tail -20
```

Expected: error referencing missing module `../SowSectionNote`.

- [ ] **Step 3: Commit failing tests**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowSectionNote.spec.ts
git commit -m "test(aim-onboarding): add failing SowSectionNote tests (O2)"
```

---

## Task 3 — Coordinate: add `ApprovedAt` to `SowApprovalPayload` in F2

Before writing the components, `SowApprovalPayload` in `interfaces/aimOnboarding.ts` must include `ApprovedAt`. B7's DTO marks it `[Required]`. F2 currently defines only `ApproverName` / `ApproverJobTitle`.

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts`

- [ ] **Step 1: Add `ApprovedAt` to `SowApprovalPayload`**

Open `interfaces/aimOnboarding.ts`. Find the `SowApprovalPayload` interface (currently lines with `ApproverName` and `ApproverJobTitle`) and add the `ApprovedAt` field:

```ts
/** Payload for PUT /{id}/sow-approval */
export interface SowApprovalPayload {
  ApproverName: string;
  ApproverJobTitle: string;
  /** Client-generated ISO 8601 timestamp. B7 stores it as-is and returns 204 — no response body. */
  ApprovedAt: string;
}
```

- [ ] **Step 2: Verify TypeScript is clean**

```bash
cd /path/to/frontend-mos
npx tsc --noEmit --project packages/advertiser/tsconfig.json 2>&1 | grep -i "SowApproval\|onboarding"
```

Expected: no output (no errors).

- [ ] **Step 3: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts
git commit -m "feat(aim-onboarding): add ApprovedAt to SowApprovalPayload interface (O2/F2)"
```

---

## Task 4 — Implement `SowSectionNote.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowSectionNote.vue`

- [ ] **Step 1: Write the component**

```vue
<template>
  <div class="sow-section-note">
    <label
      :for="`note-${sectionKey}`"
      class="sow-section-note__label"
    >Notes</label>
    <!-- TODO: Phase 2 — notes-only endpoint needed post-approval so edits survive after approvedAt is set -->
    <v-textarea
      :id="`note-${sectionKey}`"
      :model-value="noteValue"
      variant="outlined"
      density="compact"
      rows="3"
      hide-details
      placeholder="Add notes for this section…"
      @update:model-value="handleInput"
    />
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useAimOnboarding } from '../composables/useAimOnboarding'

const props = defineProps<{
  sectionKey: string
}>()

const { scopeOfWork, patchSoW } = useAimOnboarding()

const noteValue = computed(
  () => scopeOfWork.value.SectionNotes[props.sectionKey] ?? ''
)

function handleInput(value: string) {
  patchSoW({
    SectionNotes: {
      ...scopeOfWork.value.SectionNotes,
      [props.sectionKey]: value,
    },
  })
}
</script>

<style scoped>
.sow-section-note {
  margin-top: 8px;
}

.sow-section-note__label {
  display: block;
  font-size: 10.5px;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: rgb(var(--v-theme-text2, 92, 94, 96));
  margin-bottom: 6px;
}
</style>
```

- [ ] **Step 2: Run `SowSectionNote` tests — expect pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowSectionNote.spec.ts
```

Expected:

```text
 ✓ SowSectionNote > renders a textarea
 ✓ SowSectionNote > initialises textarea value from SectionNotes[sectionKey]
 ✓ SowSectionNote > calls patchSoW with updated SectionNotes map on input
 ✓ SowSectionNote > textarea remains editable when approvedAt is set (locked state)
 ✓ SowSectionNote > renders with a visible "Notes" label above the textarea

Test Files  1 passed (1)
Tests       5 passed (5)
```

If a test fails, read the error message before proceeding. Common cause: the `v-textarea` emits `update:model-value`, not `input` — confirm the `@update:model-value` handler is wired.

- [ ] **Step 3: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowSectionNote.vue
git commit -m "feat(aim-onboarding): add SowSectionNote — always-editable per-section inline notes (O2)"
```

---

## Task 5 — Implement `SowApprovalBlock.vue`

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowApprovalBlock.vue`

- [ ] **Step 1: Write the component**

```vue
<template>
  <div class="sow-approval-block">
    <!-- Locked state: approval already recorded -->
    <v-alert
      v-if="isApproved"
      data-testid="approved-banner"
      variant="tonal"
      color="success"
      :icon="mdiCheckCircleOutline"
      class="sow-approval-block__banner"
    >
      Approved by {{ scopeOfWork.ApproverName }} ({{ scopeOfWork.ApproverJobTitle }}) on
      {{ formattedApprovedAt }}
    </v-alert>

    <!-- Pre-approval form -->
    <template v-else>
      <div class="sow-approval-block__fields">
        <label
          for="approver-name"
          class="sow-approval-block__label"
        >
          Full name <span aria-hidden="true" class="sow-approval-block__req">*</span>
        </label>
        <InputText
          id="approver-name"
          data-testid="name-input"
          v-model="localName"
          label="Full name"
          aria-required="true"
          :aria-label="'Approver full name'"
          :disabled="isApproved"
        />

        <label
          for="approver-title"
          class="sow-approval-block__label"
        >
          Job title <span aria-hidden="true" class="sow-approval-block__req">*</span>
        </label>
        <InputText
          id="approver-title"
          data-testid="title-input"
          v-model="localTitle"
          label="Job title"
          aria-required="true"
          :aria-label="'Approver job title'"
          :disabled="isApproved"
        />
      </div>

      <p
        v-if="!scopeOfWork.CsmApproved"
        class="sow-approval-block__waiting"
      >
        Waiting for your onboarding team to activate approval before you can sign off.
      </p>

      <v-alert
        v-if="errorMessage"
        variant="tonal"
        color="error"
        density="compact"
        class="sow-approval-block__error"
      >
        {{ errorMessage }}
      </v-alert>

      <v-btn
        data-testid="approve-btn"
        color="primary"
        variant="flat"
        :disabled="!canApprove"
        :loading="submitting"
        :prepend-icon="mdiCheck"
        class="sow-approval-block__btn"
        @click="handleApprove"
      >
        Approve Scope of Work
      </v-btn>

      <p class="sow-approval-block__hint">
        Approving confirms agreement to the scope above and starts the delivery timeline.
      </p>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { mdiCheck, mdiCheckCircleOutline } from '@mdi/js'
import { InputText } from '@mos/core'
import { useAimOnboarding } from '../composables/useAimOnboarding'
import { onboardingService } from '../services/onboarding'

const { scopeOfWork, patchSoW, advertiserId, sessionId } = useAimOnboarding()

const localName = ref(scopeOfWork.value.ApproverName ?? '')
const localTitle = ref(scopeOfWork.value.ApproverJobTitle ?? '')
const submitting = ref(false)
const errorMessage = ref('')

const isApproved = computed(() => scopeOfWork.value.ApprovedAt !== null)

const canApprove = computed(
  () =>
    scopeOfWork.value.CsmApproved &&
    localName.value.trim().length > 0 &&
    localTitle.value.trim().length > 0
)

const formattedApprovedAt = computed(() => {
  if (!scopeOfWork.value.ApprovedAt) return ''
  return new Intl.DateTimeFormat('en-GB', {
    day: 'numeric',
    month: 'short',
    year: 'numeric',
  }).format(new Date(scopeOfWork.value.ApprovedAt))
})

async function handleApprove() {
  errorMessage.value = ''
  submitting.value = true
  const approvedAt = new Date().toISOString()
  try {
    await onboardingService.sowApproval(advertiserId.value, sessionId.value, {
      ApproverName: localName.value.trim(),
      ApproverJobTitle: localTitle.value.trim(),
      ApprovedAt: approvedAt,
    })
    // B7 returns 204 — patch local state from values sent
    patchSoW({
      ApproverName: localName.value.trim(),
      ApproverJobTitle: localTitle.value.trim(),
      ApprovedAt: approvedAt,
    })
  } catch (err: unknown) {
    const status = (err as { response?: { status?: number } })?.response?.status
    if (status === 409) {
      errorMessage.value =
        'This scope has already been approved. Refresh to see the current state.'
    } else {
      errorMessage.value = 'Approval failed. Please try again.'
    }
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.sow-approval-block {
  border: 1px solid rgb(var(--v-theme-navy-border, 30, 75, 151, 0.18));
  background: rgb(var(--v-theme-navy-soft, 30, 75, 151, 0.07));
  border-radius: 8px;
  padding: 20px 22px;
  margin-top: 22px;
}

.sow-approval-block__fields {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 14px;
}

.sow-approval-block__label {
  display: block;
  font-size: 10.5px;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: rgb(var(--v-theme-text2, 92, 94, 96));
  margin-bottom: 6px;
}

.sow-approval-block__req {
  color: rgb(var(--v-theme-error));
}

.sow-approval-block__waiting {
  font-size: 12.5px;
  color: rgb(var(--v-theme-text2, 92, 94, 96));
  margin: 8px 0 14px;
}

.sow-approval-block__hint {
  font-size: 12.5px;
  color: rgb(var(--v-theme-text2, 92, 94, 96));
  margin: 10px 0 0;
}

.sow-approval-block__btn {
  margin-top: 14px;
}

.sow-approval-block__error {
  margin-top: 10px;
}

.sow-approval-block__banner {
  font-family: Inter, system-ui, sans-serif;
}
</style>
```

- [ ] **Step 2: Run `SowApprovalBlock` tests — expect pass**

```bash
cd /path/to/frontend-mos
npx vitest run packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowApprovalBlock.spec.ts
```

Expected:

```text
 ✓ SowApprovalBlock > renders two labeled inputs — name and job title
 ✓ SowApprovalBlock > approve button is disabled when name and title are empty
 ✓ SowApprovalBlock > approve button stays disabled when csmApproved is false even with name + title
 ✓ SowApprovalBlock > shows "waiting for CSM" caption when csmApproved is false
 ✓ SowApprovalBlock > approve button is enabled when csmApproved is true and both fields filled
 ✓ SowApprovalBlock > calls onboardingService.sowApproval with name, title, and a client-generated ApprovedAt
 ✓ SowApprovalBlock > on success: patches scopeOfWork and shows approved banner with name, title, date
 ✓ SowApprovalBlock > on 409: shows "already approved" error, does not crash
 ✓ SowApprovalBlock > in locked state: shows approval banner and hides the form

Test Files  1 passed (1)
Tests       9 passed (9)
```

If a test fails, the most common cause is the mock composable reference being stale — ensure `mockScopeOfWork` is reset in `beforeEach` and that `patchSoW` is updating `mockScopeOfWork.value` in place, not replacing the ref.

- [ ] **Step 3: Commit**

```bash
git add packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowApprovalBlock.vue
git commit -m "feat(aim-onboarding): add SowApprovalBlock — approval form, terminal lock, csmApproved gate (O2)"
```

---

## Task 6 — TypeScript clean check + full test run

- [ ] **Step 1: Verify no TypeScript errors across the onboarding module**

```bash
cd /path/to/frontend-mos
npx tsc --noEmit --project packages/advertiser/tsconfig.json 2>&1 | grep -i "AimOnboarding\|SowApproval\|SowSection"
```

Expected: no output.

- [ ] **Step 2: Run both test files together**

```bash
cd /path/to/frontend-mos
npx vitest run \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowApprovalBlock.spec.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowSectionNote.spec.ts
```

Expected:

```text
Test Files  2 passed (2)
Tests       14 passed (14)
```

- [ ] **Step 3: Run the full advertiser package test suite to confirm no regressions**

```bash
cd /path/to/frontend-mos
npm run test:ci -- --project packages/advertiser 2>&1 | tail -20
```

Expected: all pre-existing tests pass. If a test outside O2 breaks, diagnose before committing.

- [ ] **Step 4: Final commit**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/interfaces/aimOnboarding.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowApprovalBlock.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/SowSectionNote.vue \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowApprovalBlock.spec.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/output/__tests__/SowSectionNote.spec.ts
git commit -m "feat(aim-onboarding): SoW approval block + inline section notes — O2 complete"
```

---

## Self-review

### Spec coverage

- §4.2 Tab 1 approval block: name field + job title field (both required), captures `approverName`/`approverJobTitle`/`approvedAt` timestamp — covered by `SowApprovalBlock` Tasks 4–5.
- §4.2 each section editable inline notes, always editable, persisted via autosave, included in PDF — covered by `SowSectionNote` Task 4; PDF inclusion is O3's concern (notes are in state).
- §4.2 on approval: "Approved by {name} ({title}) on {date}", answers locked — approved banner + `isApproved` computed lock the form. Wizard answer lock is F1/X3 (status routing).
- §4.2 `onApproved` callback passes `approvedAt` to Timeline for date anchoring — `patchSoW({ ApprovedAt })` sets `scopeOfWork.ApprovedAt`; Timeline reads `anchorDate` from the session. F7/O4 consume it.
- §8a "inline notes remain editable after approval" — `SowSectionNote` never disables the textarea regardless of `approvedAt`.
- §8b real `<label>` not placeholder-as-label — `SowApprovalBlock` uses `InputText` with `label` prop + `aria-required`.
- §8b no focus-halo (remove mock's box-shadow ring) — `InputText` from `@mos/core` has this removed via global SCSS.
- §8b `v-alert variant="tonal"` for banners — used in both success and error states.
- `csmApproved` gate — tested explicitly: approve stays disabled when `csmApproved` is false even with fields filled.
- No Unlock — not implemented; terminal design confirmed.
- Post-approval autosave gap — noted in comment in `SowSectionNote.vue`.
- `ApprovedAt` sent client-side, B7 returns 204 — state patched from values sent, not response.

### Placeholder scan

No TBD / TODO / "similar to" / "fill in details." The post-approval note persistence gap is documented in a code comment (`<!-- TODO: Phase 2 … -->`), which is intentional, not a plan gap.

### Type consistency

- `SowApprovalPayload` (`ApproverName`, `ApproverJobTitle`, `ApprovedAt`) — defined in Task 3, used identically in `SowApprovalBlock` Task 5 `handleApprove()`.
- `ScopeOfWork` fields `CsmApproved`, `ApprovedAt`, `ApproverName`, `ApproverJobTitle`, `SectionNotes` — consistent between test mocks (Tasks 1–2), `SowApprovalBlock` (Task 5), and `SowSectionNote` (Task 4).
- `useAimOnboarding` mock shape — `{ scopeOfWork, patchSoW, advertiserId, sessionId }` — used identically in both spec files.
