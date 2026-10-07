# Task Packet: TG2 — Run tests & validate build

**Matrix:** `wi_0c97d96967969a0244e1-r1` · **Repo:** `Kochava/ko-k8s-apps` · **Stack:** `unknown` · **Owner:** `test-change-gate` (confidence 0.90)

## Objective

Run tests & validate build

## Dependencies

- Depends on `T201` — Add an Oathkeeper access rule for the onboarding-sessions route (GET/POST/PUT, api-key au…

## Allowed Files

Touch only these paths. Anything else is out of scope for this task.

- (none captured from source -- verify against the plan before editing)

## Context

**Technology (ko-k8s-apps):** Oathkeeper access rules (YAML), Kustomize

From eng-plan:

> ### Kochava/ko-k8s-apps
> **Technology**: Oathkeeper access rules (YAML)
> **Key Files**:
> - `oathkeeper/proxy/qa/portal/portal-mmm-kochava-com-api.yaml`, `oathkeeper/proxy/prod/portal/portal-mmm-kochava-com-api.yaml`
> **Changes**:
> | Change | Type | Complexity | Notes |
> |--------|------|------------|-------|
> | Route rule for `.../onboarding-sessions<.*>` | Modify | L | Mirror `mmm-portal-advertiser-saved-views` |
> ---
>

## Acceptance Criteria

- [ ] Build and existing tests still pass.

## Definition of Done

- Tests pass for every file listed above under Test.
- No file outside Allowed Files is modified.
- Commit uses the conventional-commits format; PR title references `TG2`.

## Escalation

Stop and record an open question instead of guessing if: the files listed above do not match the current checkout, an interface this task consumes does not exist yet, or completing the task requires touching a file outside Allowed Files.
