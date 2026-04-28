---
phase: 03-read-tools
plan: 05
subsystem: aos8-read-tools
tags: [aos8, wlan, config-object, read-tools, tdd]

# Dependency graph
requires:
  - phase: 03-read-tools
    plan: 01
    provides: aos8.tools.READ_ONLY, _helpers.get_object, _helpers.format_aos8_error
provides:
  - aos8/tools/wlan.py with 4 config-object read tools
  - aos8_get_ssid_profiles, aos8_get_virtual_aps, aos8_get_ap_groups, aos8_get_user_roles
affects: [03-07-wiring]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Config-object read tool pattern: fetch via get_object(client, '<obj>', config_path=...)
       — distinct from showcommand-based read tools that use run_show"
    - "Default config_path '/md' with override per CONTEXT D-04 — covers Mobility Conductor
       hierarchy by default while allowing per-MD/AP-group targeting"

key-files:
  created:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py
  modified: []

key-decisions:
  - "Tool function signature is async def (ctx, config_path='/md') -> dict[str, Any] | str —
     return type union covers both the parsed body and format_aos8_error error string."
  - "All 4 tools share an identical body shape (try get_object → except → format_aos8_error)
     differing only in object_path and verb — keeps file under 120 lines and review-friendly."

# Metrics
metrics:
  duration: ~3 min
  completed: 2026-04-28
  tasks_completed: 1
  files_created: 1
  files_modified: 0
  tests_added: 0  # tests pre-landed in 03-01 (Wave 0 red tests now green)
  tests_passing: 8  # all wlan red tests now GREEN
---

# Phase 3 Plan 05: AOS8 WLAN Read Tools Summary

Implemented READ-16..19 in `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py`: 4 AOS8 configuration-object read tools that route through `get_object()` (config-object endpoint) rather than `run_show` — exposing SSID profiles, virtual-AP profiles, AP-groups, and user-roles to the AI client.

## What Changed

- **Created** `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` (~114 lines)
  - 4 `@tool(name="aos8_*", annotations=READ_ONLY)`-decorated async functions
  - Each accepts `(ctx, config_path: str = "/md")` and returns `dict[str, Any] | str`
  - Each delegates to `_helpers.get_object(client, "<object_path>", config_path=...)`
  - Each catches `Exception` and renders via `format_aos8_error(exc, "<verb>")`
  - Object paths: `ssid_prof`, `virtual_ap`, `ap_group`, `role`
  - Action verbs: "fetch SSID profiles", "fetch virtual APs", "fetch AP groups", "fetch user roles"

## Verification

- `uv run pytest tests/unit/test_aos8_read_wlan.py -x -q` — 8 passed (all WLAN red tests flipped GREEN)
- `uv run ruff check src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — All checks passed
- `uv run mypy src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — Success: no issues found
- `uv run pytest tests/unit/ -q` — 690 passed, 3 failed (test_aos8_init.py wiring tests — out of scope; landed in 03-07)

Done-criteria checks:
- `grep -c '@tool(name=' wlan.py` → 4 ✓
- `grep -c 'get_object(client' wlan.py` → 4 ✓
- `grep -c 'run_show' wlan.py` → 0 ✓
- File under 500 lines (114) ✓

## Deviations from Plan

None — plan executed exactly as written. Object paths, default `config_path`, verbs, and decorator wiring all match `<behavior>` and `<action>` exactly.

## Commits

- `8b2fdd5` — feat(03-05): implement AOS8 WLAN read tools (READ-16..19)

## Self-Check: PASSED

- FOUND: hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py
- FOUND: commit 8b2fdd5 in hpe-networking-mcp sub-repo
