# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/k4a-mcp-server with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/k4a-mcp-server |
| Phase | Phase 6 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/k4a-mcp-server with every task above merged. Do not edit source.

This task is part of K4A MCP Equal Priority Override.

### k4a-mcp-server (sole affected repo)
**Technology**: Python ≥3.11, FastMCP 2.12.4, aiohttp (verified — investigation + KG, confidence: high)
**Key Files**:
- `tools/reconciliation_tools.py` — hosts `_validate_settings`, `_set_reconciliation_settings`, and the `kochava_set_attribution_windows` tool/docstring (verified — CA-1/CA-3/CA-7)
- `unit-tests/test_reconciliation_tools.py` — unit tests + the schema regression guard class `TestAttributionWindowsSettingsSchema` (verified — CA-9, `:525-572`)
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| `_validate_settings`: bespoke `equal_priority_window` range branch (`-1`/`0`/`1/60≤h≤24`), inserted before the generic numeric `else` | Modify | L | Mirror float `0.016666666666666666`; error names valid values + `-1`=app default (FR-1, AC-3) |
| `_validate_settings`: frozenset `{0,1}` check for `reconciliation_appstore_search_exception` | Modify | L | Reuse `_ENUM_SETTING_VALID_TOKENS`-style idiom (FR-2, AC-4) |
| `_set_reconciliation_settings` + tool return: capture `result.get("warnings")` and attach onto read-back dict, guarded by `not isinstance(result, str)` | Modify | M | Two coordinated edits; mirror smartlink idiom (CA-8); gated by change-source question (FR-4, AC-5) |
| `kochava_set_attribution_windows` docstring: document 3 keys + BR-3 note | Modify | L | Published as tool `description`; keep `settings` schema `anyOf[object,string]` with no null branch (FR-3, AC-6) |
| Unit tests for AC-1…AC-5 | New | M | Accept/reject paths, no-HTTP assertions, warnings assertion |
| Carve out existing negative-lookback parametrized test so `-1` is valid for `equal_priority_window` | Modify | L | Easy to miss; required for AC-7 "existing suite passes" |

## Summary
This change makes the K4A MCP tool `kochava_set_attribution_windows` able to remove a partner's CTV Equal Priority override (via `equal_priority_window = -1`) and to validate CTV-related keys correctly client-side, mirroring dash-api's contract exactly. The work is fully self-contained in `Kochava/k4a-mcp-server` (Python 3.11+, FastMCP): two bespoke validation branches, a two-part warnings capture-and-attach, a docstring extension, and unit tests. No dash-api, schema, or deploy changes.

## Specification Validation
*Summary of findings from eng-research.md (verified against live `k4a-mcp-server` and `dash-api` checkouts)*

### Confirmed Assumptions
| Spec Statement | Verified By | Notes |
|----------------|-------------|-------|
| Fix lives in `_validate_settings` (FR-1/FR-2) | CA-1 (k4a-mcp-server, verified) | Two-branch design confirmed at `:79-113` |
| Tool rejects `-1` before sending | CA-1, line 108 (`numeric_value < 0`) (verified) | New logic must exempt `-1` for `equal_priority_window` |
| `25` triggers ~4-min retry loop | CA-4 (verified) | 400 retriable, `MAX_RETRIES=10`, backoff ≈255s |
| Warnings discarded (FR-4) | CA-7, lines 158/161 + 341 (verified) | Two-part capture+attach needed |
| Three keys undocumented (FR-3) | CA-6 grep, CA-3 docstring (verified) | Pure docstring addition + BR-3 note |
| BR-1: `-1`/`0`/`1/60 ≤ h ≤ 24` | CA-2 (dash-api, verified) | Mirror float `0.016666666666666666` |
| BR-2: appstore exception ∈ `{0,1}` | CA-6 (dash-api, verified) | Validated unconditionally server-side |
| Warnings path = top-level `warnings` | CA-7 (dash-api, verified, `base_response.go:9`) | Resolves backend Q2; use `result.get("warnings")` |
| Input schema structure unchanged | Investigation §2a (verified) | Signature frozen; only docstring text changes |

### Source Code Changes
```text
k4a-mcp-server/
├── tools/
│   └── reconciliation_tools.py       [MODIFY]
└── unit-tests/
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
- T003 - Document the three CTV keys in the tool docstring
- T005 - Accept-path, carve-out, and full-suite green

**Blocks:**
- none

## Testing Notes

```bash
pytest
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/k4a-mcp-server` tasks in one PR on branch `impl/prd090-backend`.

---

**Parent Task:** K4A MCP Equal Priority Override — Kochava/k4a-mcp-server (6 tasks · 1 PR)
**PRD:** docs/16-PRDs/090-k4a-mcp-equal-priority-override