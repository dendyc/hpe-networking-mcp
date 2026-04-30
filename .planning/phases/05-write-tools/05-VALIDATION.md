---
phase: 5
slug: write-tools
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-28
---

# Phase 5 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4+ |
| **Config file** | `pyproject.toml` (`[tool.pytest.ini_options]`) |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py -x -q` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py -x -q`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 05-01-01 | 01 | 0 | WRITE-01..12 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py -x -q` | ❌ W0 | ⬜ pending |
| 05-02-01 | 02 | 1 | WRITE-01..12 | unit | `cd hpe-networking-mcp && uv run python -c "from hpe_networking_mcp.platforms.aos8.tools._helpers import post_object"` | ❌ W0 | ⬜ pending |
| 05-02-02 | 02 | 1 | WRITE-01 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_ssid_profile_create_calls_correct_endpoint -x -q` | ❌ W0 | ⬜ pending |
| 05-02-03 | 02 | 1 | WRITE-02 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_virtual_ap_create_posts_virtual_ap_body -x -q` | ❌ W0 | ⬜ pending |
| 05-02-04 | 02 | 1 | WRITE-03 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_ap_group_create_posts_ap_group_body -x -q` | ❌ W0 | ⬜ pending |
| 05-02-05 | 02 | 1 | WRITE-04 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_user_role_create_posts_role_body -x -q` | ❌ W0 | ⬜ pending |
| 05-02-06 | 02 | 1 | WRITE-05 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_vlan_create_posts_vlan_id_body -x -q` | ❌ W0 | ⬜ pending |
| 05-02-07 | 02 | 1 | WRITE-06 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py -k manage_aaa_server -x -q` | ❌ W0 | ⬜ pending |
| 05-02-08 | 02 | 1 | WRITE-07 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_aaa_server_group_create_posts_server_group_prof_body -x -q` | ❌ W0 | ⬜ pending |
| 05-02-09 | 02 | 1 | WRITE-08 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_acl_create_posts_acl_sess_body -x -q` | ❌ W0 | ⬜ pending |
| 05-02-10 | 02 | 1 | WRITE-09 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_manage_netdestination_create_posts_netdst_body -x -q` | ❌ W0 | ⬜ pending |
| 05-02-11 | 02 | 1 | WRITE-10 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_disconnect_client_no_config_path -x -q` | ❌ W0 | ⬜ pending |
| 05-02-12 | 02 | 1 | WRITE-11 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_reboot_ap_no_config_path -x -q` | ❌ W0 | ⬜ pending |
| 05-02-13 | 02 | 1 | WRITE-12 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_write_memory_uses_dedicated_endpoint -x -q` | ❌ W0 | ⬜ pending |
| 05-02-14 | 02 | 1 | WRITE-01..12 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py::test_elicitation_middleware_enables_aos8_tags -x -q` | ❌ W0 | ⬜ pending |
| 05-03-01 | 03 | 2 | WRITE-01..12 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_write.py tests/unit/test_aos8_init.py -q` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `hpe-networking-mcp/tests/unit/test_aos8_write.py` — stubs for WRITE-01 through WRITE-12 (9 manage_X config tools + 2 operational + write_memory + cross-cut tests for tags, response shape, config_path requirement, no-implicit-write_memory, elicitation gate, TOOLS dict wiring, middleware extension)
- [ ] `hpe-networking-mcp/tests/unit/fixtures/aos8/write_ssid_prof_success.json` — success-shape fixture used by manage_X create/update/delete tests
- [ ] `hpe-networking-mcp/tests/unit/fixtures/aos8/write_global_error.json` — failure fixture with non-zero `_global_result.status` for error-shape tests

*Wave 0 must create stub tests that import and reference each write tool function before any implementation begins. The existing AOS8 test pattern uses module-level helpers in `tests/unit/test_aos8_*.py` plus a shared `tests/conftest.py` registry-stub fixture — no per-platform `conftest.py` or package marker is required. JSON fixtures live alongside the existing AOS8 read fixtures in `tests/unit/fixtures/aos8/`.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Elicitation confirmation dialog shown to user | WRITE-01..12 | Requires live MCP session with client supporting elicitation protocol | Connect Claude Desktop, enable `ENABLE_AOS8_WRITE_TOOLS=true`, invoke any write tool, verify confirmation dialog appears |
| `write_memory` only persists — no implicit calls | WRITE-12 | Requires live AOS8 device | Invoke a write tool, disconnect/reconnect, verify config reverts without explicit `write_memory` call |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
</content>
</invoke>