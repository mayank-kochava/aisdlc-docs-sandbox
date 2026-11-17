# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/k4a-mcp-server with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/k4a-mcp-server |
| Phase | Phase 4 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/k4a-mcp-server with every task above merged. Do not edit source.

This task is part of K4A MCP Equal Priority Override Removal.

### k4a-mcp-server (IMPLEMENTATION TARGET)
**Technology**: Python / FastMCP (verified)
**Key Files**:
- `tools/reconciliation_tools.py` — hosts the tool, validation, read/merge/post flow, and the warning-discard site
- `unit-tests/test_reconciliation_tools.py` — existing validation + schema tests
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| `_set_reconciliation_settings` stop discarding the downstream warning field (`:172-178`) | Modify | M | Warning currently arrives in-process (`handle_api_response` returns full body, `k4a_api_tools.py:355-361`) but is dropped at `:175`. Capture it off the write leg. |
| `kochava_set_attribution_windows` thread warning from `write_result` (`:362`) into final return (`:369`) | Modify | M | **The one non-trivial control-flow edit.** Final return is a fresh read-back, not the write response — so capture from the write leg and merge into the read-back dict. Use `success_with_warnings` + `warnings[]` (precedent `faa_create_acc.py:316-323`). Guard dict-vs-string result (`smartlink_tools.py` `not isinstance(result, str)`). |
| Add `equal_priority_enabled` to docstring (`:279-283`) — BR-9 | Modify | L | `equal_priority_window` and `reconciliation_appstore_search_exception` already documented. Pending OQ on whether it also needs validation (see Register). |
| New BR-7 unit test + BR-1…BR-6 regression-guards | New | M | Patch `_set_reconciliation_settings` to return a warning payload; assert `status == "success_with_warnings"` + populated `warnings` (mirror `test_faa_create_acc.py:178-193`). Ensure return-shape change doesn't break `TestAttributionWindowsSettingsSchema` (`:567-613`). |

### kochava-k4a-mcp-server, mcp, clog, constants, docs (NOT AFFECTED — ruled out by investigation)
- `kochava-k4a-mcp-server`: verified registry/marketing stub (3 files, zero code).
- `mcp` (`github.com/kochava/mcp`): verified distinct Go vendor-connector gateway; none of the override logic.
- `clog`, `constants`: verified unrelated generic libraries/services surfaced by name/KG false-positive.
- `docs`: verified static Docusaurus site; its `dash-api.yaml` OpenAPI mirror under-models `warnings` (has `messages`, not `warnings`) — a doc-hygiene drift noted for the dash-api owner, not a change site here.

## Summary
Surface downstream `dash-api` warnings from the `kochava_set_attribution_windows` MCP tool as a non-plain-success result (BR-7) and document `equal_priority_enabled` in the tool docstring (BR-9). Investigation (Read-verified across all candidate repos) confirmed the feature is ~85% already shipped: validation (BR-1…BR-6), the `-1` override-removal sentinel, and validate-before-dispatch already exist and are tested in `k4a-mcp-server`; the `-1` removal semantics, range/enum validation, and the "network without CTV support" warning already exist in `dash-api`. The real, narrow work lands entirely in `k4a-mcp-server` (Python/FastMCP), reusing the existing in-repo `success_with_warnings` convention.

## Technical Context
**Language/Version**: Python (FastMCP server); `k4a-mcp-server` is the sole implementation target (verified)
**Primary Dependencies**: FastMCP; `pytest` + `pytest-asyncio` + `unittest.mock` for tests
**Storage**: None in `k4a-mcp-server` — stateless, in-process MCP tool server; all persistence lives in `dash-api` (external) via `POST /apps/{app_id}/configure_reconciliation` (verified: `reconciliation_tools.py:158-178`)
**Testing**: pytest/pytest-asyncio with mock-patched legs; call tools via `._internal(...)` to assert raw dicts (verified idiom: `test_reconciliation_tools.py:26-45`)
**Target Platform**: Deploy topology external to repo — no k8s/Helm/env manifests in `k4a-mcp-server` (verified absent)
**Performance Goals**: NFR-001 — no measurable p95 regression for a valid request; NFR-002 — synchronous rejection already satisfied (validation runs before first network call, `reconciliation_tools.py:343-347`)
**Constraints**: BR-8/FR-011/NFR-006/NFR-010 — read/merge/post flow, retry policy, and all other settings keys must remain unchanged; forward `equal_priority_window` values **literally** (no pre-rounding) because `dash-api` uses `big.Rat` exact arithmetic to defeat rounding attacks (verified: `equal_priority.go:46-85`)

### Go Style Guide
- [ ] N/A — implementation target is Python (`k4a-mcp-server`). No Go code is edited (dash-api requires no change).

### Corrected Assumptions
| Spec Statement | Actual Reality | Impact |
|----------------|----------------|--------|
| The change is net-new validation + flow work | ~85% already implemented across `k4a-mcp-server` and `dash-api` | Reframe FR-001…FR-008 as "assert existing behavior." Primary new work is BR-7 + BR-9. |
| Downstream returns "a status that is not a plain success" | `dash-api` returns `200 + success:true + warnings[]` even on drop (verified: `configure-reconciliation.go:338-343`, `Success` unconditionally `true`) | "Not a plain success" must be **synthesized MCP-side** from a non-empty `warnings` array. |
| A new result-status convention is needed | Two in-repo precedents exist (`success_with_warnings` + `warnings` list) | Reuse `success_with_warnings`; do not invent vocabulary (verified: `faa_create_acc.py:316-323`, `smartlink_tools.py:1205-1212`). |
...

## Implementation Guide

1. Implement: Run tests & validate build
2. Add or update the tests listed under Files to Modify
3. Run `pytest`

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
- T003 - Document equal_priority_window, equal_priority_enabled, appstore search exception and warning status in tool docstring

**Blocks:**
- none

## Testing Notes

```bash
pytest
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/k4a-mcp-server` tasks in one PR on branch `impl/prd089-backend`.

---

**Parent Task:** K4A MCP Equal Priority Override Removal — Kochava/k4a-mcp-server (4 tasks · 1 PR)
**PRD:** 089-k4a-mcp-equal-priority-override-removal