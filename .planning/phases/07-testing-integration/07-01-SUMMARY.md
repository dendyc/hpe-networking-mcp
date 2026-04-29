---
phase: 07
plan: 01
subsystem: testing-integration
tags: [tdd, wave-0, red-baseline, aos8, differentiators, security]
requires: []
provides: [test_aos8_read_differentiators.py, test_aos8_security.py, EXPECTED_DIFF_TOOLS, 9 fixture JSON files]
affects: [tests/unit/test_aos8_init.py, tests/unit/test_aos8_config.py]
tech_stack_added: []
patterns: [TDD red-then-green, contract pinning via assert_awaited_once_with, hard-fail name-presence scan]
key_files_created:
  - hpe-networking-mcp/tests/unit/test_aos8_read_differentiators.py
  - hpe-networking-mcp/tests/unit/test_aos8_security.py
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_switch_hierarchy.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_running_config.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_pending_config.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_ap_arm_neighbors.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_lc_cluster_group.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_ap_monitor_active.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_ap_port_status.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_crypto_ipsec_sa.json
  - hpe-networking-mcp/tests/unit/fixtures/aos8/show_user_summary.json
key_files_modified:
  - hpe-networking-mcp/tests/unit/test_aos8_init.py
  - hpe-networking-mcp/tests/unit/test_aos8_config.py
decisions:
  - "DIFF-09 partial-failure contract: failing sub-call returns {'error': ...} in its section, other sections still populated (Pitfall 3)"
  - "Token leak test asserts on token VALUE only, not literal 'UIDARUBA' (Pitfall 1)"
  - "TEST-05 gap-filled with monkeypatch on config.SECRETS_DIR (no fixture dependency)"
requirements: [TEST-01, TEST-02, TEST-03, TEST-04, TEST-05]
metrics:
  duration_minutes: 12
  tasks_completed: 4
  commits: 4
  completed_date: "2026-04-29"
---

# Phase 7 Plan 1: Wave 0 RED Test Scaffold Summary

Wave 0 RED scaffold for Phase 7 — landed all DIFF-01..09 contract tests, the
tool-layer UIDARUBA token-leak audit, and updated wiring expectations
(EXPECTED_TOTAL=47) ahead of any implementation. Verified TEST-01/03/05
coverage via automated name-presence scan; closed the TEST-05 naming gap with
a single-line gap-fill test.

## Tasks Completed

| # | Task | Commit |
| - | ---- | ------ |
| 1 | Add 9 DIFF fixture JSON files | `3ff3a98` |
| 2 | RED tests for DIFF-01..09 (test_aos8_read_differentiators.py) | `9f00d29` |
| 3 | Tool-layer UIDARUBA leak audit (test_aos8_security.py) | `f733124` |
| 4 | Bump EXPECTED_TOTAL to 47 and add TEST-05 gap-fill | `4a63a54` |

## Fixture → DIFF Tool Mapping

| Fixture | DIFF Tool | Show Command |
| ------- | --------- | ------------ |
| show_switch_hierarchy.json | DIFF-01 aos8_get_md_hierarchy | `show switch hierarchy` |
| show_running_config.json | DIFF-02 aos8_get_effective_config (fallback) | (object endpoint, not showcommand) |
| show_pending_config.json | DIFF-03 aos8_get_pending_changes | `show configuration pending` |
| show_ap_arm_neighbors.json | DIFF-04 aos8_get_rf_neighbors | `show ap arm-neighbors ap-name <ap>` |
| show_lc_cluster_group.json | DIFF-05 aos8_get_cluster_state | `show lc-cluster group-membership` |
| show_ap_monitor_active.json | DIFF-06 aos8_get_air_monitors | `show ap monitor active-laser-beams` |
| show_ap_port_status.json | DIFF-07 aos8_get_ap_wired_ports | `show ap port status ap-name <ap>` |
| show_crypto_ipsec_sa.json | DIFF-08 aos8_get_ipsec_tunnels | `show crypto ipsec sa` |
| show_user_summary.json | DIFF-09 aos8_get_md_health_check (client-count source) | `show user summary` (one of 5 sub-calls) |

All fixtures contain `_global_result.status="0"` per Pitfall 2.

## Contract Decisions Per DIFF Tool

| DIFF | Endpoint | Required Params | Default config_path | Returns |
| ---- | -------- | --------------- | ------------------- | ------- |
| 01 | `/v1/configuration/showcommand` | `command="show switch hierarchy"` | n/a | full body sans `_meta`/`_global_result` |
| 02 | `/v1/configuration/object/{object_name}` | `config_path` | `/md` | stripped object body |
| 03 | `/v1/configuration/showcommand` | `command="show configuration pending"` | n/a | stripped showcommand body |
| 04 | `/v1/configuration/showcommand` | `command="show ap arm-neighbors ap-name <ap>"`, `config_path` | (per-call) | stripped body |
| 05 | `/v1/configuration/showcommand` | `command="show lc-cluster group-membership"` | n/a | stripped body |
| 06 | `/v1/configuration/showcommand` | `command="show ap monitor active-laser-beams"` | n/a | stripped body |
| 07 | `/v1/configuration/showcommand` | `command="show ap port status ap-name <ap>"` | n/a | stripped body |
| 08 | `/v1/configuration/showcommand` | `command="show crypto ipsec sa"` | n/a | stripped body |
| 09 | composite of 4+ subcommands | `config_path` REQUIRED (no default) | none — TypeError if missing | dict with keys: `config_path`, `aps`, `clients`, `alarms`, `firmware`; partial-failure → `{"error": ...}` per section (Pitfall 3) |

## TEST-01 / TEST-03 / TEST-05 Verification Status

Hard-fail name-presence scan run via `pytest --collect-only` against
`test_aos8_client.py`, `test_aos8_write.py`, `test_aos8_config.py`:

| Token (plan grep) | Status | Coverage location |
| ----------------- | ------ | ----------------- |
| `login_success` | PASS | `TestAOS8ClientLogin::test_login_success_caches_token` |
| `token_reuse` | NAMING-GAP | Covered by `test_login_success_caches_token` (asserts single login despite multiple `_ensure_token()` calls) and `test_login_serialized_under_lock` |
| `401_refresh` | NAMING-GAP | Covered by `test_401_triggers_refresh_and_retry_once` and `test_double_401_raises` |
| `global_result_error` | PASS | `test_global_result_error_raises`, `test_global_result_status_int_normalization` |
| `verify_ssl` | PASS | `test_verify_ssl_*` parametrized tests |
| `token` | PASS | `TestAOS8ClientLogging::test_uidaruba_never_in_logs` |
| `config_path` | PASS | multiple in test_aos8_write.py |
| `auto_disabled`/`disabled_when_secrets_missing` | PASS (after gap-fill) | new `test_aos8_auto_disabled_when_secrets_missing` |

Two naming gaps (`token_reuse`, `401_refresh`) — actual coverage is intact;
the plan's grep terms are stricter than the test names. Per plan
instruction ("Do NOT add tests for TEST-01 or TEST-03"), no new tests added —
documented here for the verifier.

TEST-05 closed by a one-line `monkeypatch` test in test_aos8_config.py.

## Final Test State (RED baseline established)

| Suite | Result | Notes |
| ----- | ------ | ----- |
| `test_aos8_read_differentiators.py` | RED 13/13 fail | ModuleNotFoundError for `differentiators` — expected until plan 07-02 |
| `test_aos8_security.py` | RED 2/2 fail | Trips redaction-line guard on legitimate log message `"AOS8: requesting new UIDARUBA token from <host>"` — surfaces a real masking gap to fix in 07-02 |
| `test_aos8_init.py` | RED 3/3 fail | EXPECTED_TOTAL=47 vs registered 38 — expected until plan 07-03 wires differentiators |
| `test_aos8_config.py::test_aos8_auto_disabled_when_secrets_missing` | GREEN 1/1 | Gap-fill test passes immediately |
| ruff check on new/modified files | GREEN | All checks passed |

## writes.py Signature Confirmation Log (Task 3 STEP 0)

Confirmed signature of `aos8_manage_ssid_profile`:

```python
async def aos8_manage_ssid_profile(
    ctx: Context,
    config_path: _ConfigPath,        # required, str
    action_type: _ActionType,        # required, "create" | "update" | "delete"
    payload: dict,                   # required, must contain 'profile-name'
    confirmed: _Confirmed = False,
)
```

Signature matched plan author's pre-verified spec exactly. No drift detected.
Test invocation in `test_aos8_security.py::test_uidaruba_value_not_logged_during_write_tool`
uses these exact parameter names.

## Token Leak Test Design Decision (Pitfall 1)

Asserts the token VALUE (`tok-supersecret-leak-bait-99`) never appears in any
captured log line, AND any line referencing the literal `UIDARUBA` must also
include `<redacted>`. This catches both:

1. Raw token leakage (the obvious bug)
2. URL-encoded token leakage where redaction was forgotten

The current code emits `INFO ... AOS8: requesting new UIDARUBA token from <host>`
which trips the redaction-line guard. This is a real surface to be fixed in
07-02 (either reword the log to not say "UIDARUBA" or add `<redacted>` marker).

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] N806 ruff lint violation in test_aos8_init.py**
- **Found during:** Task 4 verification
- **Issue:** Local set named `_READ_CATEGORIES` failed ruff N806 (uppercase var inside function)
- **Fix:** Renamed to `read_categories`
- **Commit:** rolled into `4a63a54`

### Naming-gap notes (no test added)

`token_reuse` / `401_refresh` grep tokens not present as literal substrings
in test names, but coverage is fully present (see TEST-01/03/05 table above).
Plan explicitly forbids adding TEST-01/03 tests for naming reasons. Documented
here so the verifier can confirm coverage on the existing names.

## Self-Check: PASSED

- All 9 fixture files exist and parse as valid JSON
- `test_aos8_read_differentiators.py` exists, 13 tests collected, all RED
- `test_aos8_security.py` exists, 2 tests collected, all RED
- `test_aos8_init.py` updated: EXPECTED_DIFF_TOOLS, EXPECTED_TOTAL=47, "differentiators": 9
- `test_aos8_config.py::test_aos8_auto_disabled_when_secrets_missing` PASSED
- ruff clean on all new/modified test files
- 4 commits in hpe-networking-mcp sub-repo: 3ff3a98, 9f00d29, f733124, 4a63a54
