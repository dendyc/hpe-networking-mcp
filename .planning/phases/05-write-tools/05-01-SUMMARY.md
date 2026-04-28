---
phase: "05-write-tools"
plan: "01"
subsystem: "aos8-write-tools"
tags: ["tdd", "wave-0", "red-baseline", "aos8", "write-tools"]
dependency_graph:
  requires: []
  provides:
    - "Wave 0 red-baseline tests for 12 AOS8 WRITE tools"
    - "Test contracts for WRITE-01..12 (endpoint, body, tags, elicitation, response shape)"
    - "JSON fixtures for write tool response shapes"
  affects:
    - "hpe-networking-mcp/tests/unit/test_aos8_write.py"
    - "hpe-networking-mcp/tests/unit/fixtures/aos8/"
tech_stack:
  added: []
  patterns:
    - "TDD Wave 0 red baseline (mirrors Phase 3 Plan 01 approach)"
    - "AsyncMock on client.request for AOS8 tool testing"
    - "Parametrized test for response shape contract across all 11 non-write_memory tools"
key_files:
  created:
    - "hpe-networking-mcp/tests/unit/test_aos8_write.py"
    - "hpe-networking-mcp/tests/unit/fixtures/aos8/write_ssid_prof_success.json"
    - "hpe-networking-mcp/tests/unit/fixtures/aos8/write_global_error.json"
  modified: []
decisions:
  - "Tests import writes module symbols inside test bodies (not at module level) so collection succeeds even when writes.py does not exist — red baseline visible at runtime not collect-time"
  - "test_response_shape_contract_writes_01_to_11 uses parametrize for 11 tools to avoid 11 near-identical test functions while still pinning requires_write_memory_for contract on each"
  - "conftest.py confirmed to already have aos8._registry stub from Phase 03-01 — no conftest changes needed"
  - "test_elicitation_middleware_enables_aos8_tags tests the middleware directly without importing writes.py so it provides a contract for Plan 05-02's elicitation.py modification"
metrics:
  duration_minutes: 6
  completed_date: "2026-04-28"
  tasks_completed: 1
  tasks_total: 1
  files_created: 3
  files_modified: 0
---

# Phase 5 Plan 01: AOS8 Write Tools Wave 0 Red Baseline Summary

**One-liner:** TDD Wave 0 red baseline — 44 tests locking the contract for all 12 AOS8 write tools before writes.py implementation.

## What Was Built

Three files establishing the Wave 0 red baseline for the 12 AOS8 WRITE tools (WRITE-01..12):

1. **`tests/unit/test_aos8_write.py`** — 34 test functions generating 44 test cases (11 via parametrize). All fail with `ModuleNotFoundError` on `hpe_networking_mcp.platforms.aos8.tools.writes`.

2. **`tests/unit/fixtures/aos8/write_ssid_prof_success.json`** — Success response fixture used by manage_X tests.

3. **`tests/unit/fixtures/aos8/write_global_error.json`** — Error fixture with `_global_result.status: "1"` for error-path tests.

## Test List (44 tests)

### WRITE-01: `aos8_manage_ssid_profile` (6 tests)
| Test | Contract Pinned |
|------|----------------|
| `test_manage_ssid_profile_create_calls_correct_endpoint` | POST to `/v1/configuration/object`, body has `ssid_prof._action=add`, result has `requires_write_memory_for=["/md"]` |
| `test_manage_ssid_profile_update_uses_modify_action` | `action_type="update"` → `_action: modify` |
| `test_manage_ssid_profile_delete_uses_delete_action` | `action_type="delete"` → `_action: delete` |
| `test_manage_ssid_profile_invalid_action_raises` | `action_type="garbage"` → `ToolError` |
| `test_manage_ssid_profile_missing_profile_name_raises` | Payload without `profile-name` → `ToolError` |
| `test_manage_ssid_profile_global_result_error_returns_error_dict` | `AOS8APIError` → `{result: {error: "AOS8 API error..."}, requires_write_memory_for: []}` |

### WRITE-02..09: Parallel happy-path tests (8 tests)
| Test | Object Key | Identifier |
|------|-----------|-----------|
| `test_manage_virtual_ap_create_posts_virtual_ap_body` | `virtual_ap` | `profile-name` |
| `test_manage_ap_group_create_posts_ap_group_body` | `ap_group` | `profile-name` |
| `test_manage_user_role_create_posts_role_body` | `role` | `rolename` |
| `test_manage_vlan_create_posts_vlan_id_body` | `vlan_id` | `id` |
| `test_manage_aaa_server_radius_uses_rad_server_key` | `rad_server` | `rad_server_name` |
| `test_manage_aaa_server_tacacs_uses_tacacs_server_key` | `tacacs_server` | — |
| `test_manage_aaa_server_ldap_uses_ldap_server_key` | `ldap_server` | — |
| `test_manage_aaa_server_internal_uses_internal_db_server_key` | `internal_db_server` | — |
| `test_manage_aaa_server_group_create_posts_server_group_prof_body` | `server_group_prof` | `sg_name` |
| `test_manage_acl_create_posts_acl_sess_body` | `acl_sess` | `accname` |
| `test_manage_netdestination_create_posts_netdst_body` | `netdst` | `dstname` |

### WRITE-10..12: Operational tools (4 tests)
| Test | Contract |
|------|---------|
| `test_disconnect_client_no_config_path` | Body `{aaa_user_delete: {mac: ...}}`, params=None/{}, `requires_write_memory_for=[]` |
| `test_reboot_ap_no_config_path` | Body `{apboot: {ap-name: ...}}`, params=None/{}, `requires_write_memory_for=[]` |
| `test_write_memory_uses_dedicated_endpoint` | POST to `/v1/configuration/object/write_memory`, `json_body={}`, NO `requires_write_memory_for` in result |
| `test_write_memory_empty_body_not_none` | `json_body == {}` and `is not None` (Pitfall 10 guard) |

### Cross-cut tests (15 tests including 11 parametrized)
| Test | Contract |
|------|---------|
| `test_config_path_required_on_manage_ssid_profile` | Missing `config_path` → TypeError/ValidationError |
| `test_config_path_required_on_write_memory` | Same |
| `test_disconnect_client_does_not_accept_config_path` | `config_path` not in signature |
| `test_reboot_ap_does_not_accept_config_path` | Same |
| `test_response_shape_contract_writes_01_to_11` [x11] | Every non-write_memory tool returns `{result, requires_write_memory_for}` |
| `test_no_implicit_write_memory` | Exactly one `client.request` call, path is `/v1/configuration/object` not `write_memory` |
| `test_elicitation_required_when_not_confirmed` | `mode=chat_confirm, confirmed=False` → `{status: confirmation_required}`, `client.request` not called |
| `test_elicitation_disabled_state_auto_accepts` | `mode=disabled, confirmed=False` → tool calls `client.request` |
| `test_all_write_tools_carry_aos8_write_tag` | All 12 tools have `aos8_write` in REGISTRIES tags |
| `test_manage_tools_carry_aos8_write_delete_tag` | WRITE-01..09 have `aos8_write_delete`; WRITE-10..12 do not |
| `test_writes_module_has_write_and_write_delete_constants` | `WRITE.destructiveHint=False`, `WRITE_DELETE.destructiveHint=True`, both `readOnlyHint=False idempotentHint=False openWorldHint=True` |
| `test_elicitation_middleware_enables_aos8_tags` | When `enable_aos8_write_tools=True`, `on_initialize` calls `enable_components(tags={"aos8_write", "aos8_write_delete"})` |
| `test_tools_dict_includes_writes` | `TOOLS["writes"]` == set of 12 expected write tool names |

## Fixture Locations

| File | Content |
|------|---------|
| `tests/unit/fixtures/aos8/write_ssid_prof_success.json` | `{ssid_prof: {_result: {status: 0, status_str: "Success"}}, _global_result: {status: "0", status_str: "Success"}}` |
| `tests/unit/fixtures/aos8/write_global_error.json` | `{_global_result: {status: "1", status_str: "Profile already exists"}}` |

## Red Baseline Error Counts (for Plan 05-02 turning-green checklist)

```
44 failed in 4.65s
42 tests: ModuleNotFoundError — hpe_networking_mcp.platforms.aos8.tools.writes
 2 tests: AssertionError (test_tools_dict_includes_writes, test_elicitation_middleware_enables_aos8_tags)
```

The 2 non-ModuleNotFoundError failures:
- `test_tools_dict_includes_writes`: `AssertionError: TOOLS dict missing 'writes' category` — wired in Plan 05-03
- `test_elicitation_middleware_enables_aos8_tags`: assertion on middleware not yet having AOS8 enable block — wired in Plan 05-02

## Commits

| Commit | Files |
|--------|-------|
| `fa15a56` (hpe-networking-mcp repo) | `tests/unit/test_aos8_write.py`, `tests/unit/fixtures/aos8/write_ssid_prof_success.json`, `tests/unit/fixtures/aos8/write_global_error.json` |

## Deviations from Plan

None — plan executed exactly as written.

The conftest.py already had `hpe_networking_mcp.platforms.aos8._registry` in the registry stubs (confirmed in Phase 03-01), so no conftest modifications were required.

## Self-Check: PASSED

- `tests/unit/test_aos8_write.py` exists: FOUND
- `tests/unit/fixtures/aos8/write_ssid_prof_success.json` exists and contains `_global_result`: FOUND
- `tests/unit/fixtures/aos8/write_global_error.json` exists and contains `"status": "1"`: FOUND
- 44 tests collected, 44 failing with ModuleNotFoundError: CONFIRMED
- 693 existing tests still passing: CONFIRMED
- `ruff check` and `ruff format --check`: CLEAN
- Commit `fa15a56` exists: CONFIRMED
