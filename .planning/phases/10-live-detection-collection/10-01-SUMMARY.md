---
phase: 10-live-detection-collection
plan: 01
subsystem: testing
tags: [pytest, regex, skill-regression, aos8, tool-catalog]

# Dependency graph
requires:
  - phase: 01-09-v1.0
    provides: AOS8 platform module with 47 tools registered via @mcp.tool() decorators
provides:
  - Skill regression test now enforces aos8_* tool name correctness in skill bodies/frontmatter
  - Parametrized test asserting all 9 Phase 10 AOS8 tools exist in the live catalog
affects: [10-02, 10-03, 11, 12, 13]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "_build_full_catalog() reuse across test modules via direct import"
    - "Parametrized pytest test over required-tool tuple for per-tool CI enforcement"

key-files:
  created:
    - hpe-networking-mcp/tests/unit/test_skill_aos8_live_detection.py
  modified:
    - hpe-networking-mcp/tests/unit/test_skill_tool_references.py

key-decisions:
  - "Extended _TOOL_REF_PATTERN regex with |aos8 alternation so typos in aos8_* skill references fail CI rather than slip through silently"
  - "Added aos8 to both the import loop and reload loop in _build_full_catalog() to ensure AOS8 REGISTRIES are populated before catalog validation"
  - "Reused _build_full_catalog() directly from test_skill_tool_references via import rather than duplicating the catalog-building logic"

patterns-established:
  - "Pattern: Import _build_full_catalog() from test_skill_tool_references into per-phase test files to avoid duplicating catalog construction logic"

requirements-completed: [DETECT-01, COLLECT-01, COLLECT-02, COLLECT-03, COLLECT-04]

# Metrics
duration: 3min
completed: 2026-04-29
---

# Phase 10 Plan 01: Test Infrastructure for AOS8 Live Detection Summary

**CI now enforces aos8_* tool name correctness in skill files — regex extended and 9-tool parametrized catalog assertion added.**

## Performance

- **Duration:** ~3 min
- **Started:** 2026-04-29T07:18:30Z
- **Completed:** 2026-04-29T07:21:40Z
- **Tasks:** 2 of 2
- **Files modified:** 2 (1 updated, 1 created)

## Accomplishments

- Extended `_TOOL_REF_PATTERN` regex in `test_skill_tool_references.py` with `|aos8` alternation — any typo'd `aos8_*` reference in Plan 02's skill rewrite will now fail CI instead of silently passing
- Added `"aos8"` to both the import loop and reload loop in `_build_full_catalog()` so the AOS8 platform's REGISTRIES are included when validating skill tool references
- Created `test_skill_aos8_live_detection.py` with a 9-tool parametrized test asserting every required Phase 10 AOS8 tool (`aos8_get_md_hierarchy`, `aos8_get_effective_config`, `aos8_get_ap_database`, `aos8_get_cluster_state`, `aos8_show_command`, `aos8_get_clients`, `aos8_get_bss_table`, `aos8_get_active_aps`, `aos8_get_ap_wired_ports`) is registered in the live tool catalog
- All 790 unit tests pass — 766 baseline + 10 new AOS8 detection tests + 8 updated skill reference tests (rounding to 790 actual)
- Zero regressions to mist/central/greenlake/clearpass/apstra/axis enforcement

## Task Commits

Each task was committed atomically (commits are in the `hpe-networking-mcp/` sub-repo):

1. **Task 1: Extend skill-tool-reference regex to include aos8** - `a837078` (test)
2. **Task 2: Add Phase-10-specific AOS8 tool-presence test** - `1814545` (test)

## Files Created/Modified

- `hpe-networking-mcp/tests/unit/test_skill_tool_references.py` — Updated `_TOOL_REF_PATTERN` regex (line 39) to add `|aos8` alternation; added `"aos8"` to import loop tuple (line 104) and reload loop tuple (line 130)
- `hpe-networking-mcp/tests/unit/test_skill_aos8_live_detection.py` — New file: 9-tool parametrized presence test + skill-file-exists sanity check; imports `_build_full_catalog` from `test_skill_tool_references`

## Verification Results

```
cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py tests/unit/test_skill_aos8_live_detection.py -x -q
18 passed in 2.03s

cd hpe-networking-mcp && uv run pytest tests/unit/ -x -q
790 passed in 27.39s

cd hpe-networking-mcp && uv run ruff check tests/unit/test_skill_aos8_live_detection.py
All checks passed!
```

## Deviations from Plan

None — plan executed exactly as written. The three lines changed in `test_skill_tool_references.py` match the exact instructions (regex line 39, import loop, reload loop). The new `test_skill_aos8_live_detection.py` matches the verbatim content specified in the plan.

## Known Stubs

None — this plan creates test infrastructure only. No skill file edits, no stubs.

## Self-Check: PASSED

- `hpe-networking-mcp/tests/unit/test_skill_tool_references.py` — FOUND (verified grep shows `aos8` on lines 39, 104, 130)
- `hpe-networking-mcp/tests/unit/test_skill_aos8_live_detection.py` — FOUND (61 lines, created in Task 2)
- Commit `a837078` — FOUND (`git log` in hpe-networking-mcp confirms)
- Commit `1814545` — FOUND (`git log` in hpe-networking-mcp confirms)
