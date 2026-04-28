---
phase: 06-guided-prompts-documentation
plan: 01
subsystem: aos8-prompts
tags: [prompts, mcp, aos8, operator-workflows, guided-prompts]
dependency_graph:
  requires:
    - 05-03  # AOS8 TOOLS dict wiring (write tools must be present for pre_change_check reference)
  provides:
    - 9 registered MCP prompts for AOS8 operator workflows (PROMPT-01..09)
  affects:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/prompts.py
tech_stack:
  added:
    - "@mcp.prompt decorator pattern for synchronous prompt registration (mirroring central/tools/prompts.py)"
  patterns:
    - "register(mcp) entry point called from __init__.py inside try/except block"
    - "f-string interpolation for parameterized prompts; plain triple-quoted strings for parameterless ones"
key_files:
  created:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/prompts.py
    - hpe-networking-mcp/tests/unit/test_aos8_prompts.py
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py
decisions:
  - "Followed central/tools/prompts.py registration pattern exactly — register(mcp) plain def, @mcp.prompt decorator inside, one-line docstrings as prompt descriptions"
  - "aos8_pre_change_check explicitly names aos8_write_memory in the prompt body to surface the WRITE-12 contract to operators"
  - "try/except wrapper in __init__.py ensures prompt registration failure cannot break tool registration (per plan requirement)"
metrics:
  duration_seconds: 331
  completed_date: "2026-04-28"
  tasks_completed: 3
  tasks_total: 3
  files_created: 2
  files_modified: 1
---

# Phase 6 Plan 1: AOS8 Guided Prompts Module Summary

**One-liner:** 9 AOS8 operator workflow prompts registered as MCP prompts via register(mcp) pattern, covering triage, health-check, RF analysis, WLAN review, and pre-change safety checks with explicit WRITE-12 contract reminder.

## What Was Built

### New Files

**`hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/prompts.py`** (217 lines)

Contains a single `register(mcp)` entry point that decorates 9 synchronous prompt functions with `@mcp.prompt`. Each function returns a numbered-step string with explicit `aos8_*` tool names and a `Summarize:` section per the D-02 decision. The 9 prompts:

| Prompt | Parameters | PROMPT-ID |
|--------|-----------|-----------|
| `aos8_triage_client` | `mac_address: str` | PROMPT-01 |
| `aos8_triage_ap` | `ap_name: str` | PROMPT-02 |
| `aos8_health_check` | (none) | PROMPT-03 |
| `aos8_audit_change` | (none) | PROMPT-04 |
| `aos8_rf_analysis` | `config_path: str = "/md"` | PROMPT-05 |
| `aos8_wlan_review` | (none) | PROMPT-06 |
| `aos8_client_flood` | `config_path: str = "/md"` | PROMPT-07 |
| `aos8_compare_md_config` | `md_path_1: str, md_path_2: str` | PROMPT-08 |
| `aos8_pre_change_check` | `config_path: str` | PROMPT-09 |

**`hpe-networking-mcp/tests/unit/test_aos8_prompts.py`** (64 lines)

11-test smoke suite:
- `test_register_attaches_all_nine_prompts` — verifies the captured set of decorated functions equals EXPECTED_PROMPTS
- `test_each_prompt_returns_nonempty_summary_string[x9]` — parametrized; each prompt called with placeholder "x" args returns a non-empty str containing "Summarize"
- `test_pre_change_check_references_write_memory` — confirms WRITE-12 contract mention in body

### Modified Files

**`hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py`**

Added 8 lines after the TOOLS importlib for-loop and before `mode = config.tool_mode`:

```python
try:
    from hpe_networking_mcp.platforms.aos8.tools import prompts

    prompts.register(mcp)
    logger.debug("AOS8: loaded prompts")
except Exception as e:
    logger.warning("AOS8: failed to load prompts -- {}", e)
```

Mirrors central/__init__.py pattern exactly. TOOLS dict unmodified (still 38 tools across 6 categories).

## Test Results

| Suite | Before | After | Delta |
|-------|--------|-------|-------|
| Unit tests passing | 756 | 767 | +11 |
| New prompt tests | 0 | 11 | +11 |
| Pre-existing integration failure | 1 | 1 | 0 (unchanged) |

Note: The 1 pre-existing integration test failure (`test_no_visibility_transform_when_write_tools_enabled`) was present before this plan and is unrelated to prompts.

## Verification

- `prompts.register(fake_mcp)` captures exactly 9 functions matching EXPECTED_PROMPTS: PASSED
- `uv run ruff check` on all 3 files: PASSED
- `uv run ruff format --check` on all 3 files: PASSED (no reformatting needed)
- `uv run pytest tests/unit/test_aos8_prompts.py -x -q`: 11 passed
- `uv run pytest tests/ -q`: 767 passed (>= 748 required)
- `aos8_pre_change_check` body contains literal string `aos8_write_memory`: CONFIRMED

## WRITE-12 Contract Confirmation

The `aos8_pre_change_check` prompt body explicitly states:

> "after the change the operator MUST explicitly call `aos8_write_memory` with the affected config_path(s) — `aos8_write_memory` is never called automatically; this is the WRITE-12 contract."

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| Task 1 | `8df1cc4` | feat(06-01): add AOS8 guided prompts module with all 9 operator workflows |
| Task 2 | `7a6196d` | feat(06-01): wire prompts.register(mcp) into aos8/__init__.py via try/except |
| Task 3 | `740bc93` | test(06-01): add smoke tests for all 9 AOS8 guided prompts |

## Deviations from Plan

None — plan executed exactly as written. The pre-existing integration test failure was present before this plan and is out of scope.

## Known Stubs

None — all 9 prompts return substantive numbered-step content with real tool names wired in. No placeholder text present.

## Self-Check: PASSED

- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/prompts.py` — exists, 217 lines
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — contains `prompts.register(mcp)`
- `hpe-networking-mcp/tests/unit/test_aos8_prompts.py` — exists, 11 tests pass
- Commit `8df1cc4` — confirmed in git log
- Commit `7a6196d` — confirmed in git log
- Commit `740bc93` — confirmed in git log
