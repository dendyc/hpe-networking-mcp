---
phase: 02-api-client
plan: 03
subsystem: api
tags: [aos8, health-probe, server-lifespan, write-gate, fastmcp]

# Dependency graph
requires:
  - phase: 02-api-client/02-02
    provides: AOS8Client with health_check(), aclose(), server property

provides:
  - _probe_aos8 function in health.py covering ok/unavailable/degraded paths
  - AOS8 entry in _ALL_PLATFORMS tuple and _PROBES dispatch dict
  - AOS8 lifespan init/shutdown blocks in server.py
  - _register_aos8_tools helper in server.py
  - Write-gate Visibility transform for aos8_write/aos8_write_delete tags
  - 3 unit tests for _probe_aos8 (TestProbeAOS8 class)

affects: [03-read-tools, 05-write-tools]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "AOS8 follows the Apstra/Axis lifespan pattern: init in try/except, shutdown in finally with exception swallowing"
    - "_probe_aos8 mirrors _probe_apstra with AOS8-specific hostname+version extraction from health_check()"
    - "Write-gate Visibility transforms all use single-line style for consistency with other platforms"

key-files:
  created: []
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/health.py
    - hpe-networking-mcp/src/hpe_networking_mcp/server.py
    - hpe-networking-mcp/tests/unit/test_health.py

key-decisions:
  - "_probe_aos8 exposes host, hostname, version keys on ok path — matching AOS8Client.health_check() return shape"
  - "server.py line count kept to 495 by collapsing multi-line Visibility call and trimming _register_aos8_tools docstring"
  - "Pre-existing Windows UnicodeDecodeError in server.py fixed by adding encoding='utf-8' to INSTRUCTIONS.md read_text() call"

patterns-established:
  - "AOS8 platform wiring follows identical pattern to Apstra/Axis at all five integration points in server.py"

requirements-completed: [CLIENT-09, CLIENT-10]

# Metrics
duration: 25min
completed: 2026-04-28
---

# Phase 2 Plan 03: AOS8 Server Wiring Summary

**AOS8 platform wired into health probe (ok/unavailable/degraded), server lifespan (init/shutdown), tool registration stub, and write-gate Visibility transform — completing Phase 2**

## Performance

- **Duration:** ~25 min
- **Started:** 2026-04-28
- **Completed:** 2026-04-28
- **Tasks:** 2
- **Files modified:** 3 (health.py, server.py, test_health.py) + 1 pre-existing fix (test_aos8_client.py)

## Accomplishments

- Added `_probe_aos8` to `health.py` with all three status paths (ok/unavailable/degraded) and updated `_ALL_PLATFORMS` + `_PROBES` dispatch dict
- Wired AOS8 into `server.py` at all five integration points: lifespan init, lifespan shutdown, tool registration call, write-gate Visibility transform, and `_register_aos8_tools` helper
- Added `TestProbeAOS8` class with 3 regression tests; all 645 unit tests pass
- Fixed pre-existing Windows UnicodeDecodeError in `server.py` (encoding='utf-8')

## Task Commits

Each task was committed atomically:

1. **Task 1: Add _probe_aos8 to health.py and regression tests** - `3650659` (feat)
2. **Task 2: Wire AOS8 into server.py** - `10731c7` (feat)
3. **Deviation fix: ruff format test_aos8_client.py** - `d1d3ae4` (chore)
4. **Deviation fix: ruff import order test_aos8_client.py** - `1c53692` (chore)

_Note: TDD tasks had single commits (RED + GREEN merged); pre-existing lint fixes committed separately._

## Final Line Counts

- `health.py`: 290 lines
- `server.py`: 495 lines (within 500-line limit)

## Five Wiring Points in server.py Confirmed

1. Lifespan init: `context["aos8_client"] = AOS8Client(config.aos8)` with try/except
2. Lifespan shutdown: `await aos8.aclose()` with exception swallowing in finally block
3. Tool registration: `if config.aos8: _register_aos8_tools(mcp, config)`
4. Write-gate: `Visibility(False, tags={"aos8_write", "aos8_write_delete"}, components={"tool"})`
5. Helper function: `_register_aos8_tools(mcp, config)` alongside other platform helpers

## Final pytest Summary

- **645 passed**, 0 failed, 0 skipped
- All TestProbeAOS8 tests green
- All pre-existing tests unaffected

## Files Created/Modified

- `src/hpe_networking_mcp/platforms/health.py` - Added `_probe_aos8`, updated `_ALL_PLATFORMS` and `_PROBES`
- `src/hpe_networking_mcp/server.py` - Added AOS8 lifespan blocks, registration call, write-gate, helper; fixed encoding bug
- `tests/unit/test_health.py` - Added `_make_ctx` helper + `TestProbeAOS8` class (3 tests)
- `tests/unit/test_aos8_client.py` - Pre-existing ruff format/import-order fix applied

## Decisions Made

- `_probe_aos8` exposes `host`, `hostname`, `version` keys on the ok path to match AOS8Client.health_check() return shape (includes hostname from show version)
- `server.py` line count kept at 495 by using single-line Visibility call style (matching other platforms) and a concise docstring for `_register_aos8_tools`
- Pre-existing UnicodeDecodeError on Windows fixed inline (Rule 1 - Bug): `read_text(encoding="utf-8")` added to INSTRUCTIONS.md read

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed UnicodeDecodeError in server.py INSTRUCTIONS.md read**
- **Found during:** Task 2 acceptance criteria smoke test (`uv run python -c "from hpe_networking_mcp.server import create_server; print('ok')"`)
- **Issue:** `Path.read_text()` uses system locale encoding (cp1252 on Windows) which cannot decode the byte 0x9d in INSTRUCTIONS.md
- **Fix:** Added `encoding="utf-8"` to `read_text()` call on line 11 of server.py
- **Files modified:** `src/hpe_networking_mcp/server.py`
- **Verification:** Import succeeds; all 645 tests pass (5 test_code_mode.py tests that were previously failing due to this issue now pass)
- **Committed in:** `10731c7` (Task 2 commit)

**2. [Rule 1 - Bug] Fixed pre-existing ruff format/import-order violations in test_aos8_client.py**
- **Found during:** End-of-phase gate (`ruff format --check src/ tests/` + `ruff check src/ tests/`)
- **Issue:** test_aos8_client.py had a formatting issue and an unsorted import block (I001)
- **Fix:** Applied `ruff format` then `ruff check --fix`
- **Files modified:** `tests/unit/test_aos8_client.py`
- **Verification:** `ruff check src/ tests/` and `ruff format --check src/ tests/` both exit 0
- **Committed in:** `d1d3ae4`, `1c53692`

---

**Total deviations:** 2 auto-fixed (2 Rule 1 - pre-existing bugs)
**Impact on plan:** The encoding fix unblocked the server import smoke test and also resolved 5 previously-failing unit tests. The ruff fixes were required for the end-of-phase gate.

## Issues Encountered

None beyond the auto-fixed deviations above.

## Note for Phase 3

- `platforms/aos8/tools/` directory is empty — Phase 3 will populate it
- `_register_aos8_tools` is already wired in server.py and will pick up new tools without further server.py changes
- Meta-tools (`aos8_list_tools`, `aos8_get_tool_schema`, `aos8_invoke_tool`) are deferred until the first tool ships in Phase 3
- The health tool now reports AOS8 status alongside the other 6 platforms

## Next Phase Readiness

- Phase 3 (Read Tools) can begin immediately — all server wiring is complete
- No further `server.py` or `health.py` changes needed for Phase 3 tool development
- Write-gate transform is in place for Phase 5 write tools

---
*Phase: 02-api-client*
*Completed: 2026-04-28*
