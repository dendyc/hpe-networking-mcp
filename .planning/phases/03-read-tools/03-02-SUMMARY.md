---
phase: 03-read-tools
plan: 02
subsystem: aos8-platform
tags: [aos8, read-tools, health, inventory, fastmcp, tdd-green]

# Dependency graph
requires:
  - phase: 03-read-tools
    plan: 01
    provides: _helpers.run_show, _helpers.format_aos8_error, READ_ONLY annotation, red tests in test_aos8_read_health.py
  - phase: 02-api-client
    provides: AOS8Client (lifespan_context["aos8_client"])
provides:
  - aos8_get_controllers (READ-01)
  - aos8_get_ap_database (READ-02)
  - aos8_get_active_aps (READ-03)
  - aos8_get_ap_detail (READ-04)
  - aos8_get_bss_table (READ-05)
  - aos8_get_radio_summary (READ-06)
  - aos8_get_version (READ-07)
  - aos8_get_licenses (READ-08)
affects: [03-07-wiring]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Per-category tool module pattern: one file per functional area (health.py)
       containing all tools that share the same domain — mirrors central/tools layout"
    - "Conductor-root vs config_path-scoped tool dichotomy: tools whose underlying
       show command runs only at the Mobility Conductor root (show switches, show
       version, show license) omit config_path entirely; managed-device-scoped
       tools accept config_path with default '/md'"
    - "Mutually-exclusive selector pattern: aos8_get_ap_detail uses
       (ap_name is None) == (ap_mac is None) to enforce 'exactly one of' without
       Pydantic validators — returns plain error string before any HTTP call"

key-files:
  created:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py
  modified: []

key-decisions:
  - "aos8_get_ap_detail returns the literal string 'Must provide exactly one of ap_name or ap_mac' when both selectors are None or both are set, matching the test contract from Plan 01 and avoiding Pydantic-style validation errors that surface as unreadable middleware output"
  - "All 8 tools use uniform try/except pattern with format_aos8_error and a verb-phrase action string — keeps error rendering consistent with other AOS8 tools landing in Plans 03-04..07"
  - "config_path is a positional-or-keyword parameter (not keyword-only) on the 5 scope-sensitive tools — tests call both ways (positional config_path='/md/site1' as kwarg and default), so the simpler signature wins"
  - "Type annotation dict[str, Any] | str matches what run_show returns (dict after strip_meta) and what format_aos8_error returns (str) — accurately reflects the success/failure union without over-specifying the success shape"

metrics:
  duration: ~10 min
  tasks_completed: 1
  files_created: 1
  files_modified: 0
  loc_added: 191
  tests_added: 0
  tests_flipped_green: 12
---

# Phase 3 Plan 02: AOS8 Health/Inventory Read Tools Summary

Implemented all 8 AOS8 health/inventory read tools (READ-01..08) in a single new `tools/health.py` module, flipping every test in `test_aos8_read_health.py` from red to green while satisfying the per-tool API contracts pinned by Plan 01.

## What Shipped

A single new file `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py` (191 lines) containing 8 async tool functions, each decorated `@tool(name="aos8_<n>", annotations=READ_ONLY)`. Every tool routes its single show command through the shared `run_show` helper from Plan 01 and renders exceptions through `format_aos8_error` with a tool-specific verb phrase.

## Per-Tool Implementation

| Tool                     | Command                                  | config_path | Action verb               |
| ------------------------ | ---------------------------------------- | ----------- | ------------------------- |
| `aos8_get_controllers`   | `show switches`                          | none        | "list controllers"        |
| `aos8_get_ap_database`   | `show ap database`                       | `"/md"`     | "fetch AP database"       |
| `aos8_get_active_aps`    | `show ap active`                         | `"/md"`     | "list active APs"         |
| `aos8_get_ap_detail`     | `show ap details ap-name <n>` or `ap-mac <m>` | `"/md"` | "fetch AP details"        |
| `aos8_get_bss_table`     | `show ap bss-table`                      | `"/md"`     | "fetch BSS table"         |
| `aos8_get_radio_summary` | `show ap radio-summary`                  | `"/md"`     | "fetch radio summary"     |
| `aos8_get_version`       | `show version`                           | none        | "fetch software version"  |
| `aos8_get_licenses`      | `show license`                           | none        | "fetch licenses"          |

## Verification

- `uv run pytest tests/unit/test_aos8_read_health.py -x -q` → **12 passed in 0.61s**
- `uv run ruff check src/hpe_networking_mcp/platforms/aos8/tools/health.py` → all checks passed
- `uv run ruff format --check src/.../health.py` → no changes needed
- `uv run mypy src/.../health.py` → no issues
- `uv run pytest tests/unit -q` → **690 passed, 3 failed** — the 3 failures are pre-existing red tests in `test_aos8_init.py::test_tools_dict_complete`, `test_register_tools_dynamic_mode`, `test_register_tools_static_mode` that depend on Plan 03-07 wiring (TOOLS dict + `register_tools()`); they were red before this plan and remain red as planned. No regressions in the 645 pre-existing tests.
- `grep -c "@tool(name=\"aos8_" health.py` → **8** ✓
- File line count: **191** (well under 500 limit)

## Deviations from Plan

None — the plan executed exactly as written. The single Task 1 implementation matched the behavior contract verbatim. No deviation rules triggered: no bugs found, no missing functionality discovered, no blockers, no architectural questions.

## Self-Check: PASSED

- ✓ FOUND: hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py (committed at fc4efb1 in inner repo)
- ✓ FOUND: 8 `@tool(name="aos8_..."` decorations in health.py
- ✓ FOUND: 12/12 tests in test_aos8_read_health.py passing
- ✓ FOUND: ruff + mypy clean
