---
phase: 08-fix-diff-tools-production-bug
verified: 2026-04-29T04:30:00Z
status: passed
score: 5/5 must-haves verified
re_verification: false
gaps: []
human_verification: []
---

# Phase 08: Fix DIFF Tools Response-Contract Bug — Verification Report

**Phase Goal:** Fix the response-contract bug in differentiators.py where local _show() and _object() helpers pass httpx.Response directly to strip_meta() instead of calling .json() first. Consolidate by deleting local helpers and importing canonical run_show/get_object from _helpers. Update test mocks to match real AOS8Client.request() contract.
**Verified:** 2026-04-29T04:30:00Z
**Status:** PASSED
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| #  | Truth                                                                                                 | Status     | Evidence                                                                                                       |
|----|-------------------------------------------------------------------------------------------------------|------------|----------------------------------------------------------------------------------------------------------------|
| 1  | All 9 AOS8 DIFF tools return parsed JSON dicts (not httpx.Response objects) when invoked in production | VERIFIED  | `differentiators.py` delegates to `run_show`/`get_object` in `_helpers.py`; both call `response.json()` before `strip_meta()` |
| 2  | AOS8Client.request() return value has .json() called on it before strip_meta()                        | VERIFIED  | `_helpers.py` lines 61/81/118 confirm `response = await client.request(...)` then `return strip_meta(response.json())` |
| 3  | 13 DIFF unit tests pass GREEN with mocks that exercise the .json() call site                          | VERIFIED  | `uv run pytest tests/unit/test_aos8_read_differentiators.py -q` → 13 passed in 0.20s                          |
| 4  | Full 764-test unit suite has zero regressions                                                          | VERIFIED  | `uv run pytest tests/unit -q` → 764 passed in 28.28s                                                          |
| 5  | Ruff lint and mypy type-check are clean for the modified files                                         | VERIFIED  | `ruff check .` → All checks passed; `mypy src/ --ignore-missing-imports` → Success: no issues found in 202 source files; `ruff format --check .` → 254 files already formatted |

**Score:** 5/5 truths verified

---

### Required Artifacts

| Artifact                                                                         | Expected                                           | Status     | Details                                                                               |
|----------------------------------------------------------------------------------|----------------------------------------------------|------------|---------------------------------------------------------------------------------------|
| `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` | 9 DIFF tools using run_show/get_object from _helpers | VERIFIED | 307 lines (< 500). Imports `format_aos8_error`, `get_object`, `run_show` from `_helpers`. No local `_show`, `_object`, or `_SHOWCOMMAND_PATH`. |
| `hpe-networking-mcp/tests/unit/test_aos8_read_differentiators.py`                | 13 DIFF tests with httpx.Response-shaped mocks     | VERIFIED  | Contains `_resp()` helper (1 def), `_make_ctx` uses `AsyncMock(return_value=_resp(body))`, both DIFF-09 `_route` closures wrap returns in `_resp()`. `_resp(` count: 13. No bare `AsyncMock(return_value=body)` pattern. |

---

### Key Link Verification

| From                                        | To                               | Via                                                                                  | Status   | Details                                                                                                           |
|---------------------------------------------|----------------------------------|--------------------------------------------------------------------------------------|----------|-------------------------------------------------------------------------------------------------------------------|
| `differentiators.py` tool functions          | `_helpers.run_show / get_object` | import statement at lines 24-28                                                      | WIRED    | `from hpe_networking_mcp.platforms.aos8.tools._helpers import (format_aos8_error, get_object, run_show,)` confirmed |
| `test_aos8_read_differentiators.py` mocks    | httpx.Response contract          | MagicMock with `.json.return_value`                                                  | WIRED    | `r.json.return_value = body` in `_resp()` helper; `_make_ctx` and both `_route` closures use `_resp()`            |

---

### Data-Flow Trace (Level 4)

| Artifact             | Data Variable | Source                             | Produces Real Data | Status   |
|----------------------|---------------|------------------------------------|--------------------|----------|
| `differentiators.py` | response      | `client.request()` → `run_show()`/`get_object()` in `_helpers.py` → `response.json()` → `strip_meta()` | Yes — delegates to canonical helpers that call `.json()` before envelope stripping | FLOWING |

---

### Behavioral Spot-Checks

| Behavior                                       | Command                                                                  | Result         | Status |
|------------------------------------------------|--------------------------------------------------------------------------|----------------|--------|
| 13 DIFF tests pass with .json() mocks          | `uv run pytest tests/unit/test_aos8_read_differentiators.py -q`          | 13 passed      | PASS   |
| Full suite has zero regressions (764 tests)    | `uv run pytest tests/unit -q`                                            | 764 passed     | PASS   |
| Ruff lint clean (all files)                    | `uv run ruff check .`                                                    | All checks passed | PASS |
| Ruff format clean                              | `uv run ruff format --check .`                                           | 254 files already formatted | PASS |
| Mypy clean (202 source files)                  | `uv run mypy src/ --ignore-missing-imports`                              | Success: no issues found in 202 source files | PASS |

---

### Requirements Coverage

| Requirement | Source Plan | Description                                                                          | Status    | Evidence                                                                                                    |
|-------------|-------------|--------------------------------------------------------------------------------------|-----------|-------------------------------------------------------------------------------------------------------------|
| DIFF-01     | 08-01-PLAN  | `aos8_get_md_hierarchy` — Conductor-to-Managed-Device hierarchy                      | SATISFIED | Tool at line 49 uses `run_show(client, "show switch hierarchy")`; test passes                               |
| DIFF-02     | 08-01-PLAN  | `aos8_get_effective_config` — resolved configuration after inheritance                | SATISFIED | Tool at line 72 uses `get_object(client, object_name, config_path=config_path)`; test asserts params match  |
| DIFF-03     | 08-01-PLAN  | `aos8_get_pending_changes` — staged config not yet persisted via write_memory         | SATISFIED | Tool at line 102 uses `run_show(client, "show configuration pending")`; test passes                         |
| DIFF-04     | 08-01-PLAN  | `aos8_get_rf_neighbors` — ARM neighbor graph for an AP                                | SATISFIED | Tool at line 124 uses `run_show(client, f"show ap arm-neighbors ap-name {ap_name}", config_path=config_path)` |
| DIFF-05     | 08-01-PLAN  | `aos8_get_cluster_state` — AP cluster membership and master/standby state             | SATISFIED | Tool at line 156 uses `run_show(client, "show lc-cluster group-membership")`                                |
| DIFF-06     | 08-01-PLAN  | `aos8_get_air_monitors` — APs operating in air-monitor mode                           | SATISFIED | Tool at line 178 uses `run_show(client, "show ap monitor active-laser-beams")`                              |
| DIFF-07     | 08-01-PLAN  | `aos8_get_ap_wired_ports` — wired port config and state for APs                       | SATISFIED | Tool at line 200 uses `run_show(client, f"show ap port status ap-name {ap_name}")`                         |
| DIFF-08     | 08-01-PLAN  | `aos8_get_ipsec_tunnels` — site-to-site IPsec and Remote AP tunnel state              | SATISFIED | Tool at line 226 uses `run_show(client, "show crypto ipsec sa")`                                            |
| DIFF-09     | 08-01-PLAN  | `aos8_get_md_health_check` — unified health report for MD scope                       | SATISFIED | Tool at line 248 uses 5x `run_show(...)` calls inside `asyncio.gather(return_exceptions=True)`; partial-failure test passes |

All 9 requirement IDs from the PLAN frontmatter are present in REQUIREMENTS.md and mapped to Phase 8 (lines 178-186). No orphaned requirements detected.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None | — | — | — | — |

No stub patterns, placeholder comments, empty handlers, or suspicious hardcoded-empty data were found in the two modified files.

**Smoke check — no `await client.request` outside `_helpers.py` passes the result directly to `strip_meta()`:**
- `troubleshooting.py:106` — assigns to `response`, then line 114 calls `body = response.json()` → clean
- `writes.py:411` — assigns to `response`, then immediately calls `strip_meta(response.json())` → clean
- `client.py:67` — appears inside a docstring usage example, not executable code → clean
- `_helpers.py:61,81,118` — canonical helpers; all call `response.json()` before `strip_meta()` → clean

No tool file calls `await client.request(...)` and passes the result anywhere except to `.json()`.

---

### Human Verification Required

None — all acceptance criteria are programmatically verifiable and have been verified.

---

### Gaps Summary

No gaps. All 5 observable truths are verified, both required artifacts pass all four levels (exists, substantive, wired, data-flowing), all 9 requirement IDs are satisfied, the quality gate (ruff lint, ruff format, mypy) is clean, and the test suite stands at 764 passed with zero regressions.

The production bug — local `_show()`/`_object()` helpers passing `httpx.Response` directly to `strip_meta()` without calling `.json()` first — is confirmed eliminated. The 13 DIFF tests now exercise the real `.json()` call site via `MagicMock` with `.json.return_value`, eliminating the prior masking effect.

---

_Verified: 2026-04-29T04:30:00Z_
_Verifier: Claude (gsd-verifier)_
