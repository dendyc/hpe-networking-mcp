---
phase: 03-read-tools
plan: 01
subsystem: testing
tags: [aos8, tdd, pytest, fixtures, scaffolding, fastmcp, tool-annotations]

# Dependency graph
requires:
  - phase: 02-api-client
    provides: AOS8Client, AOS8APIError, AOS8AuthError, _registry.tool() shim
provides:
  - aos8/tools/__init__.py exporting READ_ONLY ToolAnnotations
  - aos8/tools/_helpers.py (strip_meta, run_show, get_object, format_aos8_error)
  - 8 JSON fixtures matching AOS8 showcommand and config-object response shapes
  - 47 failing red tests + 1 skip across 6 files covering all 26 READ requirements
  - conftest._install_registry_stubs extended to include aos8._registry
affects: [03-02-health, 03-03-clients, 03-04-alerts, 03-05-wlan, 03-06-troubleshooting, 03-07-wiring]

# Tech tracking
tech-stack:
  added: []  # no new libs; pytest/AsyncMock/httpx already pinned
  patterns:
    - "Wave 0 red-test scaffold pattern: every category file's failing tests
       written first so Plans 02-06 can flip them green incrementally"
    - "strip_meta/run_show/get_object/format_aos8_error helper module shape
       for the AOS8 platform — single source of truth for showcommand cleanup
       and exception rendering"

key-files:
  created:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py
    - hpe-networking-mcp/tests/unit/test_aos8_read_health.py
    - hpe-networking-mcp/tests/unit/test_aos8_read_clients.py
    - hpe-networking-mcp/tests/unit/test_aos8_read_alerts.py
    - hpe-networking-mcp/tests/unit/test_aos8_read_wlan.py
    - hpe-networking-mcp/tests/unit/test_aos8_read_troubleshooting.py
    - hpe-networking-mcp/tests/unit/test_aos8_init.py
    - hpe-networking-mcp/tests/unit/fixtures/aos8/show_switches.json
    - hpe-networking-mcp/tests/unit/fixtures/aos8/show_ap_database.json
    - hpe-networking-mcp/tests/unit/fixtures/aos8/show_user_table.json
    - hpe-networking-mcp/tests/unit/fixtures/aos8/show_user_table_verbose.json
    - hpe-networking-mcp/tests/unit/fixtures/aos8/show_alarms.json
    - hpe-networking-mcp/tests/unit/fixtures/aos8/show_audit_trail.json
    - hpe-networking-mcp/tests/unit/fixtures/aos8/ssid_prof.json
    - hpe-networking-mcp/tests/unit/fixtures/aos8/show_log_system.json
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/__init__.py
    - hpe-networking-mcp/tests/conftest.py

key-decisions:
  - "Test contracts pin exact endpoint paths and parameter dicts so Plans 02-06
     have zero ambiguity when implementing each tool"
  - "test_all_health_tools_read_only uses pytest.skip when REGISTRIES['aos8']
     is empty — runs as a smoke check after Plan 02 registers the tools"
  - "config_path defaults to '/md' for scope-sensitive tools; tools with no
     scope (version, licenses, audit_trail, ping, traceroute, client_history,
     show_command, get_logs, controller_stats) omit the param entirely"
  - "ssid_prof.json fixture intentionally has no _meta key — it's a config-object
     response, not a showcommand response (per RESEARCH Pitfall 2)"

patterns-established:
  - "AOS8 read-tool test shape: AsyncMock(client.request) + assert_awaited_once_with
     pinning method, path, and params dict; fixture-backed JSON bodies via _load()"
  - "_make_ctx helper duplicated per test file (kept self-contained for clarity
     rather than shared via conftest — each test file is independently runnable)"

requirements-completed: [READ-01, READ-02, READ-03, READ-04, READ-05, READ-06, READ-07, READ-08, READ-09, READ-10, READ-11, READ-12, READ-13, READ-14, READ-15, READ-16, READ-17, READ-18, READ-19, READ-20, READ-21, READ-22, READ-23, READ-24, READ-25, READ-26]

# Metrics
duration: ~12min
completed: 2026-04-28
---

# Phase 3 Plan 01: Wave 0 Read-Tools Scaffold Summary

**Red-test baseline + helper module + 8 JSON fixtures for all 26 AOS8 read tools, with `aos8._registry` stub wired into conftest so subsequent category plans can flip tests green incrementally**

## Performance

- **Duration:** ~12 min
- **Started:** 2026-04-28T13:27:00Z
- **Completed:** 2026-04-28T13:40:00Z
- **Tasks:** 3
- **Files modified:** 17 (15 created, 2 modified)

## Accomplishments

- conftest `_install_registry_stubs()` now stubs `aos8._registry.mcp` so tool
  modules import cleanly during test collection (Pitfall 5 closed)
- `aos8/tools/__init__.py` exports `READ_ONLY = ToolAnnotations(...)` matching
  the central platform pattern verbatim
- `aos8/tools/_helpers.py` provides four helpers used by every category plan:
  `strip_meta`, `run_show`, `get_object`, `format_aos8_error`
- 8 JSON fixtures capture representative AOS8 showcommand and config-object
  response shapes — each fixture parses cleanly via `json.loads`
- 6 pytest files (47 tests + 1 skipped) cover READ-01..26 plus TOOLS-dict and
  `register_tools()` wiring; all fail with `ModuleNotFoundError` (red baseline)
- 645 pre-existing unit tests still pass; ruff and mypy clean on new modules

## Task Commits

Each task was committed atomically (sub-repo `hpe-networking-mcp`, `--no-verify`
flag for parallel-executor hook contention avoidance):

1. **Task 1: Extend conftest registry stubs + create tools/__init__.py + _helpers.py** — `5d259f5` (feat)
2. **Task 2: Create JSON fixtures for representative AOS8 responses** — `3b22328` (test)
3. **Task 3: Write all 26 red tests + init wiring tests** — `c221ace` (test, TDD red baseline)

## Files Created/Modified

### Created
- `src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` — strip_meta + run_show + get_object + format_aos8_error helpers
- `tests/unit/test_aos8_read_health.py` — READ-01..08 red tests (12 tests)
- `tests/unit/test_aos8_read_clients.py` — READ-09..12 red tests (7 tests)
- `tests/unit/test_aos8_read_alerts.py` — READ-13..15 red tests (3 tests)
- `tests/unit/test_aos8_read_wlan.py` — READ-16..19 red tests (8 tests)
- `tests/unit/test_aos8_read_troubleshooting.py` — READ-20..26 red tests (15 tests)
- `tests/unit/test_aos8_init.py` — TOOLS dict + register_tools wiring (3 tests)
- 8 JSON fixtures under `tests/unit/fixtures/aos8/`

### Modified
- `src/hpe_networking_mcp/platforms/aos8/tools/__init__.py` — exports `READ_ONLY` ToolAnnotations
- `tests/conftest.py` — add `aos8._registry` to `_install_registry_stubs()` tuple (alphabetical position before apstra)

## Decisions Made

None beyond the locked decisions in 03-CONTEXT.md and 03-RESEARCH.md.
Plan executed exactly as specified; helper signatures, fixture shapes, and
test contracts all match the plan's literal direction.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- **Repo layout discovery:** the parallel-executor harness initially tried to
  commit at `C:/Dev/adams_mcp` (parent), but `hpe-networking-mcp/` is a nested
  git repository. Resolved by running git commands inside the sub-repo
  (`cd hpe-networking-mcp && git commit ...`). All Task 1-3 commit hashes
  recorded above are from the sub-repo's git log.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- **Plans 03-02..03-06 (category implementations):** can begin in parallel — each
  category plan flips its own red tests green by creating one tool module under
  `aos8/tools/<category>.py`. Helpers (`run_show`, `get_object`, `strip_meta`,
  `format_aos8_error`) are already in place; tool authors should `from
  hpe_networking_mcp.platforms.aos8.tools._helpers import ...` and never
  call `client.request()` directly except inside helpers.
- **Plan 03-07 (wiring):** depends on all 5 category plans landing — flips the
  three `test_aos8_init.py` tests green by populating `TOOLS` dict and calling
  `build_meta_tools` in `register_tools()`.
- **No blockers.**

## Self-Check: PASSED

- All 17 expected files present on disk
- All 3 task commit hashes (`5d259f5`, `3b22328`, `c221ace`) present in `hpe-networking-mcp` sub-repo git log
- `pytest --collect-only` collects 48 new tests
- 645 pre-existing tests pass; 47 new fail with `ModuleNotFoundError`; 1 skipped (read-only smoke check awaiting Plan 02)
- `ruff check` clean on all new files (test files + tools/)
- `mypy` clean on `src/hpe_networking_mcp/platforms/aos8/tools/`

---
*Phase: 03-read-tools*
*Completed: 2026-04-28*
