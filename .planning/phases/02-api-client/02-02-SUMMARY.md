---
phase: 02-api-client
plan: "02"
subsystem: platforms/aos8
tags: [api-client, aos8, httpx, auth, tdd]
dependency_graph:
  requires: ["02-01"]
  provides: ["AOS8Client", "AOS8AuthError", "AOS8APIError", "aos8-platform-skeleton"]
  affects: ["03-server-wiring", "health-probe"]
tech_stack:
  added: []
  patterns: ["httpx-async-client", "asyncio-lock-double-check", "lazy-login", "url-sanitizer", "global-result-check"]
key_files:
  created:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/_registry.py
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/__init__.py
  modified: []
decisions:
  - "UIDARUBA injected as query param (not cookie jar); httpx AsyncClient sends it in URL params on every request — mirrors test assertions where request.url.query contains UIDARUBA"
  - "aclose() uses POST /v1/api/logout per RESEARCH.md; errors caught in try/except Exception per Bandit B110 exception for shutdown paths"
  - "_check_global_result skips None status (str(None) == 'None') to allow responses without _global_result from passing"
metrics:
  duration: "7m"
  completed_date: "2026-04-28"
  tasks_completed: 2
  files_created: 4
  files_modified: 0
---

# Phase 02 Plan 02: AOS8 Platform Module Skeleton + AOS8Client Summary

**One-liner:** AOS8Client with lazy UIDARUBA auth, asyncio.Lock serialization, GET/POST-only whitelist, _global_result error detection, and URL sanitizer; all 14 Wave 0 tests GREEN.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Platform module skeleton | 2fcf5e4 | `__init__.py`, `_registry.py`, `tools/__init__.py` |
| 2 | AOS8Client implementation | ed91788 | `client.py` |
| - | Format fix | 5498bef | `__init__.py` (ruff format) |

## Verification Results

- `pytest tests/unit/test_aos8_client.py -x -q`: **14 passed** (exit 0)
- `pytest tests/unit/ -q`: **637 passed**, 5 pre-existing failures (unrelated: `test_code_mode.py` Unicode codec issue on Windows, `test_platform_template.py`)
- `ruff check src/ tests/`: exit 0 (all checks passed)
- `ruff format --check src/hpe_networking_mcp/platforms/aos8/`: exit 0 (4 files formatted)
- `mypy src/ --ignore-missing-imports`: exit 0 (193 source files clean)
- `wc -l client.py`: **318 lines** (< 500 max)
- `grep -c assert client.py`: **0** (no assert statements)
- `grep -c 4343 client.py`: **0** (no hardcoded port)
- Class definitions: **3** (AOS8Client, AOS8AuthError, AOS8APIError)
- `_sanitize_url_for_log|mask_secret` usages: **4** (≥ 3 required)
- `asyncio.Lock|self._lock` usages: **5** (≥ 2 required)
- Endpoint references: **5** (login, logout, showcommand × multiple)

## Requirements Satisfied

All 10 CLIENT-NN requirements satisfied:

| ID | Requirement | Satisfied By |
|----|-------------|-------------|
| CLIENT-01 | Login once, reuse UIDARUBA | `_ensure_token()` returns cached `self._token`; second call no-op |
| CLIENT-02 | Concurrent logins serialized | `asyncio.Lock` + double-checked pattern in `_ensure_token()` |
| CLIENT-03 | Constructor no HTTP I/O | `__init__` only sets attrs and creates httpx client; verified by `test_init_does_not_login` |
| CLIENT-04 | 401 → exactly one refresh | `request()` retries once after `_refresh_token()`; second 401 → `raise_for_status()` |
| CLIENT-05 | `_global_result.status != 0` → AOS8APIError | `_check_global_result()` called on every 2xx response |
| CLIENT-06 | UIDARUBA never in logs | `mask_secret()` on token logs; `_sanitize_url_for_log()` on URL logs |
| CLIENT-07 | `verify_ssl=False` emits WARNING | Constructor checks `config.verify_ssl is False`; `logger.warning(...)` |
| CLIENT-08 | Port from config, not hardcoded | `base_url=f"https://{config.host}:{config.port}"` |
| CLIENT-09 | `health_check()` returns hostname+version | GET `/v1/configuration/showcommand?command=show+version`; returns `{hostname, version, raw}` |
| CLIENT-10 | `aclose()` logs out + closes httpx | POST `/v1/api/logout` in try/except; `self._http.aclose()` in finally path |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Format] ruff format applied to __init__.py after initial commit**
- **Found during:** Post-task-2 end-of-plan verification
- **Issue:** `ruff format --check` reported `__init__.py` would be reformatted (multi-line logger.info collapsed to single line)
- **Fix:** Applied `uv run ruff format src/hpe_networking_mcp/platforms/aos8/__init__.py`
- **Files modified:** `__init__.py`
- **Commit:** 5498bef

### Design Decisions

- **_check_global_result status "None" guard:** `str(gr.get("status"))` returns `"None"` when the key is absent (Python dict.get() returns None → str(None) == "None"). The implementation explicitly excludes `"None"` from the error path so responses without a `_global_result.status` field don't raise spurious errors.

- **UIDARUBA as query param:** The tests assert on `request.url.path` for login/logout/showcommand paths and the handler logic uses path matching. The UIDARUBA token is passed as a query param (`params={"UIDARUBA": token, ...}`) which matches AOS8 API documentation (showcommand endpoint uses UIDARUBA in query string).

- **aclose() token guard:** `aclose()` only attempts logout if `self._token is not None`, preventing a no-op logout POST on clean shutdown before first login. The `_token = None` reset happens in the `finally` block to ensure it's cleared regardless of logout success.

## Known Stubs

- `TOOLS: dict[str, list[str]] = {}` in `__init__.py` — intentional; populated in Phase 3 when tool modules are created
- `register_tools()` returns 0 and logs "0 tools registered" — intentional Phase 2 stub; Phase 3 will call `build_meta_tools` and populate TOOLS

## Self-Check: PASSED

Files created verified:
- `/c/Dev/adams_mcp/hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — FOUND
- `/c/Dev/adams_mcp/hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/_registry.py` — FOUND
- `/c/Dev/adams_mcp/hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` — FOUND
- `/c/Dev/adams_mcp/hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/__init__.py` — FOUND

Commits verified:
- 2fcf5e4 (Task 1 skeleton) — FOUND
- ed91788 (Task 2 client) — FOUND
- 5498bef (format fix) — FOUND
