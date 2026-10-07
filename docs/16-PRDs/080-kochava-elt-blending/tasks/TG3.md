# TG3: Run tests & validate build

## In short

Run the full build and tests for Kochava/ko-infrastructure with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG3 |
| Repo | Kochava/ko-infrastructure |
| Phase | Phase 2 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/ko-infrastructure with every task above merged. Do not edit source.

This task is part of Kochava Elt Blending.

### ko-infrastructure + ko-k8s-apps (Infra)
**Technology**: Terraform (`terraform-google-modules/kubernetes-engine` v41.0.1), Helm/FluxCD
**Key Files**:
- `service/qa/platformmesh/greedy_goat.tf` — `s-n4-std-8-airbyte` node pool definition
  (unchanged) and the new dedicated pool's definition (added)
- `airbyte/app/qa/release.yaml` — existing job-pod `nodeSelector`/`tolerations` pattern
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| New dedicated node pool (e.g. `s-n4-std-8-blend-combine`) with `local_ssd_ephemeral_storage_count = N` and its own `blend-combine-workload` taint — `s-n4-std-8-airbyte` itself untouched | New | M | **SRE-owned task, own GitHub issue (Phase 6)** — see saved project memory. No fallback pool exists with local SSD if this new pool hits the same scale-from-zero bug |
| `MAX_SYNC_WORKERS` Helm default bump | Modify | L | Confirmed blend fan-out consumes N worker slots per run; exact new number left for real-usage sizing |
| Combine-Job `nodeSelector`/`toleration` wiring for the new `blend-combine-workload` taint, if chart-level config is needed beyond the pod spec itself | Modify | L | Mirrors the existing `global.jobs.kube` pattern, new taint name |

### Source Code Changes
```text
airbyte-platform-2.0.0/                                    [BACKEND + INFRA]
├── airbyte-db/db-lib/.../instance/kochava/migrations/
│   └── V1_0_0_018__CreateBlendTables.kt                    [NEW]  (+ ~5 more, see Affected Systems)
├── airbyte-kochava-data/.../repositories/
│   ├── entities/Blend.kt, BlendSource.kt, ...               [NEW]
│   └── BlendRepository.kt, ...                              [NEW]
├── airbyte-server/.../apis/controllers/kochava/
│   ├── BlendController.kt                                   [NEW]
│   ├── CanonicalMappingController.kt                        [NEW]
│   └── BlendOutputController.kt                             [NEW]  (`@Controller` for the combine-output download endpoint)
├── airbyte-server/.../handlers/kochava/
│   ├── BlendHandler.kt, BlendModels.kt                       [NEW]
│   ├── CanonicalMappingHandler.kt                            [NEW]
│   ├── CompositionGraphValidator.kt                          [NEW]
│   ├── BlendScheduler.kt                                     [NEW]  (also owns the output-leg: create/reuse source-file Source + Connection, trigger its sync post-combine)
│   ├── BlendOutputHandler.kt                                 [NEW]  (serves combine output from GCS over HTTPS — CustomFileSource's serving pattern, GCS-backed not Postgres-blob-backed)
│   └── AlertConditionEvaluator.kt                            [MODIFY]  (add `run_degraded`)
├── airbyte-blend-combiner/ (new Gradle module)                [NEW]
│   ├── build.gradle.kts (DuckDB via JDBC)
│   └── src/.../BlendCombineJob.kt
├── airbyte-workload-launcher/.../pods/factories/
│   └── BlendCombineJobPodFactory.kt                          [NEW]
├── airbyte-tests/src/test-acceptance/kotlin/.../
│   └── BlendAcceptanceTest.kt                                [NEW]
└── charts/v2/airbyte/values.yaml                             [MODIFY]  (MAX_SYNC_WORKERS)
...

## Implementation Guide

1. Implement: Run tests & validate build
2. Add or update the tests listed under Files to Modify
3. Run `kustomize build {dirs} && yamllint {files}`

## Files to Modify

| File | Create/Modify/Test | Note |
|---|---|---|
| (none captured, check the plan) | - | - |

## Acceptance Criteria

- Build and existing tests still pass.
- Existing tests pass
- New functionality has test coverage
- Code follows repository conventions (see CLAUDE.md)

## Dependencies

**Blocked by:**
- T201 - [SRE-owned, own issue — do not fold into an application-code task]
- T202 - Confirm the blend-staging/ GCS prefix convention with whoever owns the shared storage bucket's key-prefix scheme…
- T203 - Add nodeSelector/toleration wiring for the combine-Job pod in ko-k8s-apps/airbyte/app/qa/release.yaml, if…

**Blocks:**
- none

## Testing Notes

```bash
kustomize build {dirs} && yamllint {files}
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/ko-infrastructure` tasks in one PR on branch `impl/prd08026760-infra`.

---

**Parent Task:** Kochava Elt Blending — Kochava/ko-infrastructure (4 tasks · 1 PR)
**PRD:** docs/16-PRDs/080-kochava-elt-blending