---
phase: 02-api-client
plan: "01"
subsystem: aos8-client
tags:
  - tdd
  - testing
  - aos8
  - wave-0
dependency_graph:
  requires:
    - "01-04: AOS8Secrets dataclass in config.py"
  provides:
    - "02-02: AOS8Client test contract (CLIENT-01..CLIENT-10)"
  affects:
    - "tests/unit/ test suite (new file, no existing tests changed)"
tech_stack:
  added:
    - "tests/unit/fixtures/ directory (new)"
  patterns:
    - "MockTransport pattern mirroring test_apstra_client.py"
    - "loguru_capture fixture for token-redaction assertions"
key_files:
  created:
    - hpe-networking-mcp/tests/unit/test_aos8_client.py
    - hpe-networking-mcp/tests/unit/fixtures/aos8_show_version.json
  modified:
    - hpe-networking-mcp/tests/conftest.py
decisions:
  - "loguru_capture fixture added new (no pre-existing equivalent in conftest.py)"
  - "Import order follows ruff isort rules: platforms.aos8.client before config (ruff auto-detected ordering)"
metrics:
  duration: "~10 minutes"
  completed: "2026-04-28"
  tasks_completed: 2
  files_changed: 3
---

# Phase 02 Plan 01: Wave 0 TDD Scaffold Summary

Wave 0 TDD test scaffold for AOS8Client — 13 failing tests covering CLIENT-01..CLIENT-10, plus loguru_capture fixture and show_version JSON fixture.

## What Was Built

### Task 1: loguru_capture fixture + show version fixture

- Added `loguru_capture` pytest fixture to `tests/conftest.py` (new — no prior loguru capture existed)
  - Installs a loguru sink at TRACE level, captures all formatted log lines into a list per-test
  - Cleans up sink via `try/finally` to prevent bleed between tests
- Created `tests/unit/fixtures/aos8_show_version.json` with synthetic but plausible AOS8 `show version` response
  - Contains `_global_result.status == "0"`, `Hostname: conductor-lab-01`, `Version: ArubaOS (MODEL: ArubaMC-VA), Version 8.10.0.7`

**Commit:** f59c782

### Task 2: test_aos8_client.py (RED state)

Created `tests/unit/test_aos8_client.py` with 13 test functions across 6 test classes covering all CLIENT-01..CLIENT-10 requirements.

**Current state:** Tests fail with `ModuleNotFoundError: No module named 'hpe_networking_mcp.platforms.aos8'` — correct RED state for Wave 0.

**Commit:** 41d4a74

## Test Functions Inventory

| Test Class | Test Function | Requirement |
|---|---|---|
| TestAOS8ClientLogin | test_login_success_caches_token | CLIENT-01 |
| TestAOS8ClientLogin | test_login_serialized_under_lock | CLIENT-02 |
| TestAOS8ClientLogin | test_login_rejected_global_result_nonzero | CLIENT-01, CLIENT-05 |
| TestAOS8ClientLazy | test_init_does_not_login | CLIENT-03 |
| TestAOS8ClientRequest | test_401_triggers_refresh_and_retry_once | CLIENT-04 |
| TestAOS8ClientRequest | test_double_401_raises | CLIENT-04 |
| TestAOS8ClientRequest | test_global_result_error_raises | CLIENT-05 |
| TestAOS8ClientRequest | test_global_result_status_int_normalization | CLIENT-05 |
| TestAOS8ClientLogging | test_uidaruba_never_in_logs | CLIENT-06 |
| TestAOS8ClientMisc | test_verify_ssl_false_emits_warning | CLIENT-07 |
| TestAOS8ClientMisc | test_custom_port_in_base_url | CLIENT-08 |
| TestAOS8ClientHealth | test_health_check_returns_hostname_and_version | CLIENT-09 |
| TestAOS8ClientShutdown | test_aclose_calls_logout_endpoint | CLIENT-10 |
| TestAOS8ClientShutdown | test_aclose_swallows_logout_error | CLIENT-10, D-04 |

Total: 14 test functions (13 async + 1 sync `test_custom_port_in_base_url`).

## loguru_capture Fixture Details

- **Status:** Added new — no pre-existing loguru capture fixture existed in `tests/conftest.py`
- **Name:** `loguru_capture`
- **Scope:** function (default)
- **Yields:** `list[str]` — fully-formatted log lines
- **Usage:** `async def test_uidaruba_never_in_logs(self, loguru_capture):`

## RED State Confirmation

```
ModuleNotFoundError: No module named 'hpe_networking_mcp.platforms.aos8'
```

Tests fail only because `client.py` does not yet exist. This is the correct Wave 0 RED state. Plan 02 will create `AOS8Client` to make these tests pass.

## Verification Results

- `uv run python -c "import json; json.load(open('tests/unit/fixtures/aos8_show_version.json'))"` — PASSED
- `uv run ruff check tests/unit/test_aos8_client.py tests/conftest.py` — PASSED (0 errors)
- `uv run pytest tests/unit/ --ignore=tests/unit/test_aos8_client.py` — 623 passed, 5 pre-existing failures (test_code_mode.py, test_platform_template.py — unrelated to this plan)
- `uv run pytest tests/unit/test_aos8_client.py --collect-only` — Fails with ImportError (expected RED state)

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check: PASSED

- [x] `hpe-networking-mcp/tests/unit/test_aos8_client.py` exists (41d4a74)
- [x] `hpe-networking-mcp/tests/unit/fixtures/aos8_show_version.json` exists (f59c782)
- [x] `hpe-networking-mcp/tests/conftest.py` modified with loguru_capture (f59c782)
- [x] Commits f59c782 and 41d4a74 exist in hpe-networking-mcp repo
