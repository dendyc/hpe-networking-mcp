---
phase: 09-phase-4-closure-documentation-accuracy
plan: "01"
subsystem: server/code-mode
tags: [bug-fix, test, code-mode, execute-description, aos8]
dependency_graph:
  requires: []
  provides: [code-mode-aos8-prefix-fix, execute-description-regression-test]
  affects: [server.py, code-mode-sandbox]
tech_stack:
  added: []
  patterns: [TDD-red-green, regex-source-inspection]
key_files:
  created:
    - hpe-networking-mcp/tests/unit/test_server_code_mode.py
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/server.py
decisions:
  - "Read source file via Path(srv.__file__) for regex inspection — avoids any import-time side effects; tests remain pure unit tests"
  - "Single-line edit to execute_description literal — preserve surrounding string split; only insert aos8_ between axis_ and em-dash"
metrics:
  duration: "~3 minutes"
  completed_date: "2026-04-29"
  tasks_completed: 2
  files_touched: 2
---

# Phase 9 Plan 01: Fix aos8_ Missing from Code-Mode execute_description Summary

**One-liner:** Added `aos8_` to the code-mode `execute_description` string in server.py and added a 2-test regression suite to prevent future platform-prefix drift.

## What Was Done

### Problem

When `MCP_TOOL_MODE=code`, the in-sandbox LLM reads `execute_description` to learn which `call_tool(name, ...)` prefixes are valid. The literal listed 6 platforms (`mist_`, `central_`, `greenlake_`, `clearpass_`, `apstra_`, `axis_`) but omitted `aos8_`. This meant `await call_tool("aos8_get_md_hierarchy", ...)` would produce `Unknown tool` errors despite the tool being registered.

### Fix (Task 2)

**Before (line 374):**
```python
"`axis_` — plus the cross-platform `health` tool.\n\n"
```

**After (line 374):**
```python
"`axis_`, `aos8_` — plus the cross-platform `health` tool.\n\n"
```

That is the only change to `server.py`. The surrounding split across two string lines was preserved.

### Regression Test (Task 1)

New file: `hpe-networking-mcp/tests/unit/test_server_code_mode.py`

Two tests added:
- `test_execute_description_lists_all_platform_prefixes` — asserts all 7 platform prefixes appear as backticked tokens in the `execute_description` block
- `test_execute_description_lists_aos8_prefix` — specific guard for `aos8_` (named in 09-VALIDATION.md task 9-01-02)

Both tests were **RED** after Task 1 commit and **GREEN** after Task 2 commit.

## Test Count Delta

- Before: 764 unit tests
- After: **766 unit tests** (+2 new regression tests)
- All 766 passed; zero regressions across all 6 existing platform test suites

## Lint / Mypy / Suite Status

- `ruff check src/hpe_networking_mcp/server.py tests/unit/test_server_code_mode.py` — All checks passed
- `ruff format --check src/hpe_networking_mcp/server.py tests/unit/test_server_code_mode.py` — 2 files already formatted
- `mypy src/hpe_networking_mcp/server.py --ignore-missing-imports` — Success: no issues found
- Full suite: `uv run pytest tests/unit -q` — **766 passed**

## Commits

| Task | Hash | Message |
|------|------|---------|
| Task 1 (RED) | e58526f | test(09-01): add failing tests for execute_description aos8_ prefix |
| Task 2 (GREEN) | fa6eca1 | fix(09-01): add aos8_ to code-mode execute_description literal |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None.

## Self-Check: PASSED
