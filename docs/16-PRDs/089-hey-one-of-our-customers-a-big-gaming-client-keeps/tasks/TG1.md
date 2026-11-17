# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/k4a-mcp-server with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/k4a-mcp-server |
| Phase | Phase 3 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/k4a-mcp-server with every task above merged. Do not edit source.

This task is part of hey, one of our customers (a big gaming client) keeps asking why the dates in th.

### k4a-mcp-server (confirmed-minimum fix site)
**Technology**: Python 3.13 / FastMCP 2.12.4 / toon-format 0.9.0b1 (investigation-verified)
**Key Files**:
- `tools/campaign_tools.py:42-48` — campaign-stats serialization (`json.dumps`), the FR-001 named path
- `tools/k4a_api_tools.py:186-194` — `mcp_wrapper`, single choke point for ~19 decorated tools
- `tools/k4a_api_tools.py:895-941` — `convert_date_with_time`, the ISO pattern to mirror (read-only reference)
- `utils/kochava_agency_client.py:152-155` — client forward for campaign stats
- `tools/k4a_report_tools.py:466-468` — TOON encode path
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Add `date_format.py` recursive normalizer | New | M | Pure fn; handles US strings, epoch ints, already-ISO; FR-002 zero-pad; FR-007 fallback + anomaly |
| Modify `campaign_tools.py` (stats path) | Modify | L | Confirmed-minimum FR-001; invoke normalizer before `json.dumps` |
| Modify `k4a_api_tools.py` choke point | Modify | L | One edit covers all ~19 decorated tools (full scope) |
| Modify 5 other inline-serialize tool files | Modify | M | Full "all dates" scope (FR-003) — pending authoritative scope confirmation; prefer shared `serialize_result()` wrapper over per-site edits |
| Add unit + integration tests | New | M | pytest; both serialization styles produce identical `YYYY-MM-DD` (FR-008) |

### kochava-k4a-mcp-server (documentation surface only)
**Technology**: Registry listing — 3 files, no code (investigation-verified)
**Changes**:
| Change | Type | Complexity | Notes |
|--------|------|------------|-------|
| Optional README note on ISO output | Modify | L | Skippable; documentation-only, not an implementation touch |

## Summary
Render all in-scope dates in K4A MCP tool text results as ISO 8601 (`YYYY-MM-DD`) instead of US format (`MM/DD/YYYY`). The Read-verified investigation reframes this from a format-string swap into a **small per-site code change gated by a discovery pass**: no repo in this checkout currently emits `MM/DD/YYYY`, the MCP tools pass upstream JSON through verbatim, there is no single serialization choke point, and dates arrive in at least three shapes (epoch ints, US strings, already-ISO). The confirmed-minimum fix site is the Python/FastMCP `k4a-mcp-server`'s campaign-stats path, implemented as a pure recursive date-normalizing helper mirroring the existing `convert_date_with_time` stdlib pattern.

## Technical Context
**Language/Version**: Python 3.13 (`k4a-mcp-server`) — the confirmed fix site. (Go repos in the surrounding picture — `reporting-api` Go 1.24.9, `koch-query-api` — were investigated and ruled off the fix path.)
**Primary Dependencies**: FastMCP 2.12.4, `toon-format` 0.9.0b1, `aiohttp`, `tzdata` (already present); new helper uses stdlib `datetime`/`strftime` only — **no new dependency** (investigation-verified: `requirements.txt` already carries all needed libs).
**Storage**: None. Stateless request/response proxy; no DB, no ORM, no migrations (investigation-verified).
**Testing**: pytest with `asyncio_mode=auto` (`pytest.ini:9`), `testpaths = unit-tests`; pattern `patch("tools.<mod>.aiohttp.ClientSession")` + `AsyncMock`; decorated tools tested via `._internal(...)` to bypass TOON encoding (investigation-verified).
**Target Platform**: Hosted remote MCP server behind `https://analytics.mcp.kochava.com/mcp` (KG evidence from `server.json:11-24`; confirmation that the Python repo backs this remote is an open question — see Register).
**Performance Goals**: NFR-001 — p95 serialization within 5% of pre-change, no new network call on the result path. NFR-002 — result volume/structure unchanged.
**Constraints**: BR-4 — change must land at serialization, not a UI layer; NFR-004 — no new access surface; NFR-003 — no PII in diagnostics.

### Go Style Guide
- [ ] N/A for the confirmed fix site — `k4a-mcp-server` is Python, not Go. (If discovery relocates the fix to a Go repo such as `apis/koch-query`, Go style applies then.)

### Documentation
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
- T002 - Normalize campaign tool results before json.dumps
- T003 - Normalize dates in mcp_wrapper before TOON encode

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

**Parent Task:** hey, one of our customers (a big gaming client) keeps asking why the dates in th — Kochava/k4a-mcp-server (5 tasks · 1 PR)
**PRD:** 089-hey-one-of-our-customers-a-big-gaming-client-keeps