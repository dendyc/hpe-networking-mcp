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
| **Quick run command** | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py -x -q` |
| **Full suite command** | `uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `uv run pytest tests/unit/platforms/aos8/test_write_tools.py -x -q`
- **After every plan wave:** Run `uv run pytest tests/unit/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 05-01-01 | 01 | 0 | WRITE-01..12 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py -x -q` | ❌ W0 | ⬜ pending |
| 05-02-01 | 02 | 1 | WRITE-01 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_create_ssid_profile -x -q` | ❌ W0 | ⬜ pending |
| 05-02-02 | 02 | 1 | WRITE-02 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_create_virtual_ap -x -q` | ❌ W0 | ⬜ pending |
| 05-02-03 | 02 | 1 | WRITE-03 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_manage_ap_group -x -q` | ❌ W0 | ⬜ pending |
| 05-02-04 | 02 | 1 | WRITE-04 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_manage_user_role -x -q` | ❌ W0 | ⬜ pending |
| 05-02-05 | 02 | 1 | WRITE-05 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_manage_vlan -x -q` | ❌ W0 | ⬜ pending |
| 05-02-06 | 02 | 1 | WRITE-06 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_manage_aaa_server -x -q` | ❌ W0 | ⬜ pending |
| 05-02-07 | 02 | 1 | WRITE-07 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_manage_aaa_server_group -x -q` | ❌ W0 | ⬜ pending |
| 05-02-08 | 02 | 1 | WRITE-08 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_manage_acl -x -q` | ❌ W0 | ⬜ pending |
| 05-02-09 | 02 | 1 | WRITE-09 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_manage_netdestination -x -q` | ❌ W0 | ⬜ pending |
| 05-02-10 | 02 | 1 | WRITE-10 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_disconnect_client -x -q` | ❌ W0 | ⬜ pending |
| 05-02-11 | 02 | 1 | WRITE-11 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_reboot_ap -x -q` | ❌ W0 | ⬜ pending |
| 05-02-12 | 02 | 1 | WRITE-12 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_write_memory -x -q` | ❌ W0 | ⬜ pending |
| 05-02-13 | 02 | 1 | WRITE-01..12 | unit | `uv run pytest tests/unit/platforms/aos8/test_write_tools.py::test_elicitation_middleware -x -q` | ❌ W0 | ⬜ pending |
| 05-03-01 | 03 | 2 | WRITE-01..12 | unit | `uv run pytest tests/unit/ -q` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `tests/unit/platforms/aos8/test_write_tools.py` — stubs for WRITE-01 through WRITE-12 (12 config tools + 2 operational + write_memory + elicitation)
- [ ] `tests/unit/platforms/aos8/conftest.py` — shared `aos8_client` and `mock_responses` fixtures for write operations
- [ ] `tests/unit/platforms/aos8/__init__.py` — package marker if not already created by prior phases

*Wave 0 must create stub tests that import and reference each write tool function before any implementation begins.*

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
