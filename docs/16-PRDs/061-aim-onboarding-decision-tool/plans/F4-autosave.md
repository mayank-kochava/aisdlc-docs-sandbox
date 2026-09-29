---
id: plan-f4
title: "F4 — Mongo Hybrid Autosave"
---

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add autosave to `useAimOnboarding.ts` — localStorage buffer within a
step, debounced full-doc `PUT` to Mongo on step/tab switch and manual save, a
four-state save-status machine, 409-after-approval terminal lock, load-on-mount,
and an in-flight guard that prevents overlapping PUTs.

**Architecture:** All autosave logic lives inside `useAimOnboarding.ts`; no new
composable. A typed `debounce` helper is scoped to the composable file. The
save-status machine has exactly four states: `idle | saving | saved |
failed | locked`. `failed` exposes a `retrySave()` method. `locked` is terminal
(409 after approval) and surfaces an "approved, answers locked" message. The
localStorage key is scoped by `advertiserId` so switching accounts never
cross-contaminates.

**Tech stack:** Vue 3.5 / TypeScript, Vitest + jsdom, `mmmPortalApi` (axios),
`onboardingService` (`services/onboarding.ts`), localStorage.

---

## Files

| Action | Path |
|--------|------|
| **Modify** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts` |
| **Modify** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts` |
| **Create** | `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding-autosave.spec.ts` |

> **Path anchor:** these paths follow design-consolidated §3
> (`AimOnboardingTab/`) and the F3 plan. If F3 placed the composable at a
> different absolute path, use that path — the file name
> `useAimOnboarding.ts` is fixed.

---

## Dependencies

- **F3** — `useAimOnboarding.ts` must exist with `state` (reactive flat wizard
  doc matching design-consolidated §5), `currentStep` ref, and `advertiserId`
  computed. This plan adds autosave on top.
- **B4** — `PUT /{id}` must be deployed (or mockable) and must return 409 with
  `status: 409` when `approvedAt` is set.

---

## Assumed interfaces (from design-consolidated §5 and §6)

```ts
// services/onboarding.ts — assumed surface consumed by this plan
export interface OnboardingSession {
  id: string;
  advertiserId: string;
  status: 'not_started' | 'in_progress' | 'complete';
  wizard: Record<string, unknown>;   // flat wizard answers (§5)
  tier: { recommended: string; override: string | null; effective: string };
  scopeOfWork: Record<string, unknown>;
  timeline: Record<string, unknown>;
  changeLog: unknown[];
  createdAt: string;
  updatedAt: string;
}

// The two service calls autosave uses:
//   onboardingService.getLatest(advertiserId)
//     → GET /mmm/advertisers/{advertiserId}/onboarding-sessions
//     → Promise<OnboardingSession | null>
//
//   onboardingService.update(advertiserId, sessionId, doc)
//     → PUT /mmm/advertisers/{advertiserId}/onboarding-sessions/{sessionId}
//     → Promise<void>  (throws AxiosError with status 409 when approved)
```

These must already exist in `services/onboarding.ts` (from F2/F3). If they use
different names, update the composable to match — do not rename the service.

---

## Tasks

### Task 1 — Write the failing tests

**Files:**

- Create: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding-autosave.spec.ts`

- [ ] **Step 1.1 — Create the spec file**

```ts
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { setActivePinia, createPinia } from 'pinia';

// ---------------------------------------------------------------------------
// Hoist mocks before any import so vi.mock hoisting works correctly.
// ---------------------------------------------------------------------------
const { mockOnboardingService, mockNotificationsStore } = vi.hoisted(() => ({
  mockOnboardingService: {
    getLatest: vi.fn(),
    update: vi.fn(),
  },
  mockNotificationsStore: {
    addNotification: vi.fn(),
  },
}));

vi.mock('../../services/onboarding', () => ({
  onboardingService: mockOnboardingService,
}));

vi.mock('@mos/core', () => ({
  useNotificationsStore: () => mockNotificationsStore,
}));

// localStorage is available in jsdom; stub navigator.onLine separately per test.
import { useAimOnboarding } from '../useAimOnboarding';

// ---------------------------------------------------------------------------
// Shared fixture: a minimal valid session returned by getLatest
// ---------------------------------------------------------------------------
const SESSION = {
  id: 'session-abc',
  advertiserId: 'adv-001',
  status: 'in_progress' as const,
  wizard: { companyName: 'Acme' },
  tier: { recommended: 'aim_x', override: null, effective: 'aim_x' },
  scopeOfWork: {},
  timeline: {},
  changeLog: [],
  createdAt: '2026-01-01T00:00:00Z',
  updatedAt: '2026-01-01T00:00:00Z',
};

describe('useAimOnboarding — autosave', () => {
  beforeEach(() => {
    setActivePinia(createPinia());
    vi.clearAllMocks();
    vi.useFakeTimers();
    localStorage.clear();

    // Default: online
    Object.defineProperty(navigator, 'onLine', { value: true, configurable: true, writable: true });

    // Default: no existing session
    mockOnboardingService.getLatest.mockResolvedValue(null);
    mockOnboardingService.update.mockResolvedValue(undefined);
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  // -------------------------------------------------------------------------
  // 1. load-on-mount
  // -------------------------------------------------------------------------
  describe('loadOnMount', () => {
    it('calls getLatest and populates state when a session exists', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);

      const { loadOnMount, sessionId, saveStatus } = useAimOnboarding('adv-001');
      await loadOnMount();

      expect(mockOnboardingService.getLatest).toHaveBeenCalledWith('adv-001');
      expect(sessionId.value).toBe('session-abc');
      expect(saveStatus.value).toBe('idle');
    });

    it('restores wizard draft from localStorage when no server session', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(null);
      localStorage.setItem(
        'aim-onboarding-draft-adv-001',
        JSON.stringify({ companyName: 'Draft Co' }),
      );

      const { loadOnMount, state } = useAimOnboarding('adv-001');
      await loadOnMount();

      expect(state.wizard.companyName).toBe('Draft Co');
    });

    it('prefers server session over localStorage draft', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);
      localStorage.setItem(
        'aim-onboarding-draft-adv-001',
        JSON.stringify({ companyName: 'Stale Draft' }),
      );

      const { loadOnMount, state } = useAimOnboarding('adv-001');
      await loadOnMount();

      expect(state.wizard.companyName).toBe('Acme');
    });
  });

  // -------------------------------------------------------------------------
  // 2. localStorage buffer (within-step edits)
  // -------------------------------------------------------------------------
  describe('bufferToLocalStorage', () => {
    it('writes wizard draft to localStorage keyed by advertiserId', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);
      const { loadOnMount, bufferToLocalStorage, state } = useAimOnboarding('adv-001');
      await loadOnMount();

      state.wizard.companyName = 'New Name';
      bufferToLocalStorage();

      const stored = JSON.parse(localStorage.getItem('aim-onboarding-draft-adv-001') ?? '{}');
      expect(stored.companyName).toBe('New Name');
    });
  });

  // -------------------------------------------------------------------------
  // 3. save-status machine: happy path
  // -------------------------------------------------------------------------
  describe('saveToMongo — happy path', () => {
    it('transitions saving → saved and clears localStorage draft on success', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);
      const { loadOnMount, saveToMongo, saveStatus } = useAimOnboarding('adv-001');
      await loadOnMount();

      localStorage.setItem('aim-onboarding-draft-adv-001', JSON.stringify({ companyName: 'X' }));

      const promise = saveToMongo();
      expect(saveStatus.value).toBe('saving');

      await promise;
      expect(saveStatus.value).toBe('saved');
      expect(localStorage.getItem('aim-onboarding-draft-adv-001')).toBeNull();
      expect(mockOnboardingService.update).toHaveBeenCalledWith(
        'adv-001',
        'session-abc',
        expect.objectContaining({ wizard: expect.any(Object) }),
      );
    });
  });

  // -------------------------------------------------------------------------
  // 4. save-status machine: failed path + retry
  // -------------------------------------------------------------------------
  describe('saveToMongo — failure + retry', () => {
    it('transitions to failed on network error and re-saves on retrySave()', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);
      mockOnboardingService.update
        .mockRejectedValueOnce(new Error('Network Error'))
        .mockResolvedValue(undefined);

      const { loadOnMount, saveToMongo, saveStatus, retrySave } = useAimOnboarding('adv-001');
      await loadOnMount();

      await saveToMongo();
      expect(saveStatus.value).toBe('failed');

      await retrySave();
      expect(saveStatus.value).toBe('saved');
      expect(mockOnboardingService.update).toHaveBeenCalledTimes(2);
    });
  });

  // -------------------------------------------------------------------------
  // 5. save-status machine: offline path
  // -------------------------------------------------------------------------
  describe('saveToMongo — offline', () => {
    it('sets status to offline when navigator.onLine is false', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);
      Object.defineProperty(navigator, 'onLine', { value: false, configurable: true });

      const { loadOnMount, saveToMongo, saveStatus } = useAimOnboarding('adv-001');
      await loadOnMount();

      await saveToMongo();
      expect(saveStatus.value).toBe('offline');
      expect(mockOnboardingService.update).not.toHaveBeenCalled();
    });
  });

  // -------------------------------------------------------------------------
  // 6. save-status machine: 409 terminal lock
  // -------------------------------------------------------------------------
  describe('saveToMongo — 409 approval terminal lock', () => {
    it('sets status to locked and does NOT schedule a retry', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);

      const conflict = Object.assign(new Error('Conflict'), {
        response: { status: 409 },
      });
      mockOnboardingService.update.mockRejectedValue(conflict);

      const { loadOnMount, saveToMongo, saveStatus, isLocked } = useAimOnboarding('adv-001');
      await loadOnMount();

      await saveToMongo();
      expect(saveStatus.value).toBe('locked');
      expect(isLocked.value).toBe(true);

      // Advance timers to prove no retry fires
      vi.advanceTimersByTime(10_000);
      expect(mockOnboardingService.update).toHaveBeenCalledTimes(1);
    });
  });

  // -------------------------------------------------------------------------
  // 7. debounced save (step/tab switch triggers after 800 ms)
  // -------------------------------------------------------------------------
  describe('debouncedSave', () => {
    it('does not fire before the debounce window', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);
      const { loadOnMount, debouncedSave } = useAimOnboarding('adv-001');
      await loadOnMount();

      debouncedSave();
      debouncedSave();
      debouncedSave();

      vi.advanceTimersByTime(400);
      expect(mockOnboardingService.update).not.toHaveBeenCalled();
    });

    it('fires exactly once after the debounce window', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);
      const { loadOnMount, debouncedSave } = useAimOnboarding('adv-001');
      await loadOnMount();

      debouncedSave();
      debouncedSave();

      vi.advanceTimersByTime(800);
      await vi.runAllTimersAsync();

      expect(mockOnboardingService.update).toHaveBeenCalledTimes(1);
    });
  });

  // -------------------------------------------------------------------------
  // 8. in-flight guard (last-write-wins)
  // -------------------------------------------------------------------------
  describe('in-flight guard', () => {
    it('queues a trailing save when a PUT is already in flight', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);

      // First call resolves slowly; second arrives mid-flight
      let resolveFirst!: () => void;
      mockOnboardingService.update
        .mockImplementationOnce(
          () => new Promise<void>((resolve) => { resolveFirst = resolve; }),
        )
        .mockResolvedValue(undefined);

      const { loadOnMount, saveToMongo } = useAimOnboarding('adv-001');
      await loadOnMount();

      // Fire first save — still in-flight
      const p1 = saveToMongo();

      // Fire second save while first is in-flight
      const p2 = saveToMongo();

      // Resolve first
      resolveFirst();
      await Promise.all([p1, p2]);

      // Two calls total: first + trailing
      expect(mockOnboardingService.update).toHaveBeenCalledTimes(2);
    });

    it('does not fire more than one trailing save regardless of queuing frequency', async () => {
      mockOnboardingService.getLatest.mockResolvedValue(SESSION);

      let resolveFirst!: () => void;
      mockOnboardingService.update
        .mockImplementationOnce(
          () => new Promise<void>((resolve) => { resolveFirst = resolve; }),
        )
        .mockResolvedValue(undefined);

      const { loadOnMount, saveToMongo } = useAimOnboarding('adv-001');
      await loadOnMount();

      const p1 = saveToMongo();
      // Queue multiple trailing saves — only one should fire
      void saveToMongo();
      void saveToMongo();
      void saveToMongo();

      resolveFirst();
      await p1;
      await vi.runAllTimersAsync();

      expect(mockOnboardingService.update).toHaveBeenCalledTimes(2);
    });
  });
});
```

- [ ] **Step 1.2 — Run the spec to confirm it fails**

```bash
cd /path/to/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding-autosave.spec.ts 2>&1 | tail -30
```

Expected: multiple failures — `useAimOnboarding` does not yet export
`loadOnMount`, `bufferToLocalStorage`, `saveToMongo`, `saveStatus`,
`retrySave`, `isLocked`, `debouncedSave`, `sessionId`.

---

### Task 2 — Add the `onboardingService` methods (if not present from F2)

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts`

The tests mock `onboardingService.getLatest` and `onboardingService.update`.
Verify that `services/onboarding.ts` exports those two methods on a named
`onboardingService` object. If they exist with these signatures, skip this task.

- [ ] **Step 2.1 — Check existing service methods**

Open `services/onboarding.ts`. Confirm:

1. `getLatest(advertiserId: string): Promise<OnboardingSession | null>` — calls
   `GET /mmm/advertisers/{advertiserId}/onboarding-sessions`
2. `update(advertiserId: string, sessionId: string, doc: Partial<OnboardingSession>): Promise<void>`
   — calls `PUT /mmm/advertisers/{advertiserId}/onboarding-sessions/{sessionId}`

If either is missing, add the following to the file:

```ts
// In services/onboarding.ts — add to the existing onboardingService object.
// Do NOT create a second object; extend the existing one.

import { mmmPortalApi } from '../../../services/mmm';
import type { OnboardingSession } from '../interfaces/aimOnboarding';

const BASE = (advertiserId: string) =>
  `/mmm/advertisers/${advertiserId}/onboarding-sessions`;

export const onboardingService = {
  async getLatest(advertiserId: string): Promise<OnboardingSession | null> {
    const res = await mmmPortalApi.get<OnboardingSession | null>(BASE(advertiserId));
    return res.data ?? null;
  },

  async update(
    advertiserId: string,
    sessionId: string,
    doc: Partial<OnboardingSession>,
  ): Promise<void> {
    await mmmPortalApi.put(`${BASE(advertiserId)}/${sessionId}`, doc);
    // Throws AxiosError with response.status === 409 when approvedAt is set.
  },
};
```

- [ ] **Step 2.2 — Verify the file lints**

```bash
cd /path/to/frontend-mos/packages/advertiser
npx eslint src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts
```

Expected: no errors.

---

### Task 3 — Implement autosave in `useAimOnboarding.ts`

**Files:**

- Modify: `packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts`

- [ ] **Step 3.1 — Add the save-status type and debounce helper**

At the top of `useAimOnboarding.ts`, after existing imports, add:

```ts
import { ref, computed } from 'vue';
import type { OnboardingSession } from '../interfaces/aimOnboarding';
import { onboardingService } from '../services/onboarding';

// ---------------------------------------------------------------------------
// Save-status machine
// ---------------------------------------------------------------------------
export type SaveStatus = 'idle' | 'saving' | 'saved' | 'failed' | 'offline' | 'locked';

// ---------------------------------------------------------------------------
// Typed debounce — produces a trailing-edge debounced wrapper.
// Using a local helper rather than debounceFunc from @mos/core because that
// helper only forwards the first argument (callback(args[0])).
// ---------------------------------------------------------------------------
function debounce<T extends () => void>(fn: T, ms: number): T {
  let timer: ReturnType<typeof setTimeout> | undefined;
  return (() => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(), ms);
  }) as T;
}
```

- [ ] **Step 3.2 — Add autosave state and methods to the composable**

Locate the `return` statement of `useAimOnboarding`. Add the following reactive
refs and methods **before** the `return`, then expose them in the returned
object.

```ts
// ---------------------------------------------------------------------------
// Autosave state (module-level so all callers share the same instance,
// matching the pattern in useValidateOnboarding.ts)
// ---------------------------------------------------------------------------
const sessionId    = ref<string | null>(null);
const saveStatus   = ref<SaveStatus>('idle');
const _inFlight    = ref(false);
const _pendingSave = ref(false);

const isLocked = computed(() => saveStatus.value === 'locked');

// ---------------------------------------------------------------------------
// localStorage key scoped by advertiserId (mirrors useSnapshotStore pattern)
// ---------------------------------------------------------------------------
function _draftKey(advertiserId: string): string {
  return `aim-onboarding-draft-${advertiserId}`;
}

// ---------------------------------------------------------------------------
// bufferToLocalStorage — called on every within-step field change.
// Writes wizard answers only; server doc is the source of truth for metadata.
// ---------------------------------------------------------------------------
function bufferToLocalStorage(): void {
  if (!advertiserId.value) return;
  try {
    localStorage.setItem(_draftKey(advertiserId.value), JSON.stringify(state.wizard));
  } catch {
    // Quota exceeded — silently skip; Mongo PUT is the authoritative save.
  }
}

// ---------------------------------------------------------------------------
// saveToMongo — full-doc PUT.  Implements the state machine:
//
//   idle/saved/failed
//     → (offline check) → offline  (no PUT)
//     → saving           → saved   (PUT resolves)
//                        → locked  (PUT 409 — terminal, no retry)
//                        → failed  (any other error — retrySave() available)
//
//   If a PUT is already in flight, set _pendingSave and return early.
//   On completion, fire one trailing save if _pendingSave is set.
// ---------------------------------------------------------------------------
async function saveToMongo(): Promise<void> {
  if (_inFlight.value) {
    _pendingSave.value = true;
    return;
  }

  if (!navigator.onLine) {
    saveStatus.value = 'offline';
    return;
  }

  if (!sessionId.value || !advertiserId.value) return;

  _inFlight.value = true;
  saveStatus.value = 'saving';

  try {
    await onboardingService.update(advertiserId.value, sessionId.value, {
      wizard:      state.wizard,
      tier:        state.tier,
      scopeOfWork: state.scopeOfWork,
      timeline:    state.timeline,
    });
    saveStatus.value = 'saved';
    // Clear draft on successful server write
    try { localStorage.removeItem(_draftKey(advertiserId.value)); } catch { /* ignore */ }
  } catch (err: unknown) {
    const status = (err as { response?: { status?: number } })?.response?.status;
    if (status === 409) {
      // Approval is terminal — answers are locked. Never retry.
      saveStatus.value = 'locked';
    } else {
      saveStatus.value = 'failed';
    }
  } finally {
    _inFlight.value = false;
    if (_pendingSave.value && saveStatus.value !== 'locked') {
      _pendingSave.value = false;
      void saveToMongo();  // one trailing save
    } else {
      _pendingSave.value = false;
    }
  }
}

// ---------------------------------------------------------------------------
// retrySave — only meaningful in 'failed' or 'offline' state.
// ---------------------------------------------------------------------------
async function retrySave(): Promise<void> {
  if (saveStatus.value === 'failed' || saveStatus.value === 'offline') {
    await saveToMongo();
  }
}

// ---------------------------------------------------------------------------
// debouncedSave — call on step/tab switch.  800 ms window coalesces rapid
// navigation (e.g. clicking through the rail quickly).
// ---------------------------------------------------------------------------
const debouncedSave = debounce(() => { void saveToMongo(); }, 800);

// ---------------------------------------------------------------------------
// loadOnMount — called from onMounted() in the root tab component.
// Order of precedence: server session > localStorage draft > empty state.
// ---------------------------------------------------------------------------
async function loadOnMount(): Promise<void> {
  if (!advertiserId.value) return;
  try {
    const session = await onboardingService.getLatest(advertiserId.value);
    if (session) {
      sessionId.value = session.id;
      // Merge all server fields into state
      Object.assign(state.wizard,      session.wizard      ?? {});
      Object.assign(state.tier,        session.tier        ?? {});
      Object.assign(state.scopeOfWork, session.scopeOfWork ?? {});
      Object.assign(state.timeline,    session.timeline    ?? {});
      saveStatus.value = 'idle';
      return;
    }
  } catch {
    // Network error on mount — fall through to localStorage
  }

  // No server session — try localStorage draft
  try {
    const raw = localStorage.getItem(_draftKey(advertiserId.value));
    if (raw) {
      const draft = JSON.parse(raw) as Record<string, unknown>;
      Object.assign(state.wizard, draft);
    }
  } catch {
    // Malformed JSON — ignore
  }
}
```

- [ ] **Step 3.3 — Expose new symbols in the composable's return statement**

Find the `return { ... }` at the bottom of `useAimOnboarding`. Add to it:

```ts
// Autosave
sessionId,
saveStatus,
isLocked,
loadOnMount,
bufferToLocalStorage,
saveToMongo,
retrySave,
debouncedSave,
```

Do not remove anything already in the return statement.

- [ ] **Step 3.4 — Ensure `advertiserId` is in scope**

The composable receives `advertiserId` as a parameter or derives it from a
store. Verify it is a `ComputedRef<string>` or `Ref<string>` accessible inside
the function body. If `useAimOnboarding` takes an `advertiserId: string`
argument, wrap it in `ref`:

```ts
// At the top of useAimOnboarding, if not already present:
const advertiserId = ref(advertiserId_param);  // rename param to avoid shadowing
```

Follow whatever pattern F3 established for this — do not change the function
signature.

- [ ] **Step 3.5 — Lint the composable**

```bash
cd /path/to/frontend-mos/packages/advertiser
npx eslint src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts
```

Expected: no errors.

---

### Task 4 — Run the tests (expect pass)

- [ ] **Step 4.1 — Run only the autosave spec**

```bash
cd /path/to/frontend-mos/packages/advertiser
npx vitest run src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding-autosave.spec.ts 2>&1 | tail -40
```

Expected output: all 11 tests pass.

```text
✓ load-on-mount > calls getLatest and populates state when a session exists
✓ load-on-mount > restores wizard draft from localStorage when no server session
✓ load-on-mount > prefers server session over localStorage draft
✓ bufferToLocalStorage > writes wizard draft to localStorage keyed by advertiserId
✓ saveToMongo — happy path > transitions saving → saved and clears localStorage draft on success
✓ saveToMongo — failure + retry > transitions to failed on network error and re-saves on retrySave()
✓ saveToMongo — offline > sets status to offline when navigator.onLine is false
✓ saveToMongo — 409 approval terminal lock > sets status to locked and does NOT schedule a retry
✓ debouncedSave > does not fire before the debounce window
✓ debouncedSave > fires exactly once after the debounce window
✓ in-flight guard > queues a trailing save when a PUT is already in flight
✓ in-flight guard > does not fire more than one trailing save regardless of queuing frequency
```

- [ ] **Step 4.2 — If any test fails, diagnose before re-running**

Common failure modes:

- **`advertiserId` undefined in tests** — the composable takes an argument but
  tests call `useAimOnboarding('adv-001')`. Ensure the composable accepts a
  string arg and exposes it as a ref internally.
- **Module-level state leaks** — if `sessionId` / `saveStatus` are module-level
  (outside the function), they survive between tests. Add a `reset()` helper or
  move them inside the function. Prefer inside unless F3 explicitly made them
  module-level for the same reason `useValidateOnboarding` did.
- **`_pendingSave` clearing too early** — the trailing-save logic in `finally`
  must check `saveStatus !== 'locked'` before firing.
- **Debounce timer not advancing** — confirm `vi.useFakeTimers()` runs in
  `beforeEach` and `vi.useRealTimers()` in `afterEach`.

- [ ] **Step 4.3 — Run the full advertiser test suite**

```bash
cd /path/to/frontend-mos/packages/advertiser
npx vitest run 2>&1 | tail -20
```

Expected: no regressions. Fix any that arise (do not use `--reporter=silent` to
hide them).

---

### Task 5 — Lint-check the markdown plan

- [ ] **Step 5.1 — Lint this plan file**

```bash
npx markdownlint-cli2 \
  "/path/to/aim-onboarding-tool/docs/docs/16-PRDs/061-aim-onboarding-decision-tool/plans/F4-autosave.md"
```

Expected: zero errors.

---

### Task 6 — Commit

- [ ] **Step 6.1 — Commit the implementation and test**

```bash
git add \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/useAimOnboarding.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/composables/__tests__/useAimOnboarding-autosave.spec.ts \
  packages/advertiser/src/views/Analytics/MmmInsightsConfiguration/AimOnboardingTab/services/onboarding.ts

git commit -m "feat(onboarding): add autosave to useAimOnboarding — localStorage buffer, debounced PUT, 4-state machine, 409 lock (F4)"
```

Only include files that were actually modified.
