---
phase: 3
slug: read-tools
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-28
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4+ with pytest-asyncio (asyncio_mode=auto) |
| **Config file** | `pyproject.toml` (`[tool.pytest.ini_options]`) |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_tools.py -x -q` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_tools.py -x -q`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 3-01-01 | 01 | 0 | READ-01..26 | unit | `uv run pytest tests/unit/test_aos8_read_tools.py -x -q` | ❌ W0 | ⬜ pending |
| 3-02-01 | 02 | 1 | READ-01..06 | unit | `uv run pytest tests/unit/test_aos8_read_tools.py::test_health -x -q` | ✅ | ⬜ pending |
| 3-03-01 | 03 | 1 | READ-07..12 | unit | `uv run pytest tests/unit/test_aos8_read_tools.py::test_clients -x -q` | ✅ | ⬜ pending |
| 3-04-01 | 04 | 1 | READ-13..17 | unit | `uv run pytest tests/unit/test_aos8_read_tools.py::test_alerts -x -q` | ✅ | ⬜ pending |
| 3-05-01 | 05 | 1 | READ-18..22 | unit | `uv run pytest tests/unit/test_aos8_read_tools.py::test_wlan -x -q` | ✅ | ⬜ pending |
| 3-06-01 | 06 | 1 | READ-23..26 | unit | `uv run pytest tests/unit/test_aos8_read_tools.py::test_troubleshooting -x -q` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `hpe-networking-mcp/tests/unit/test_aos8_read_tools.py` — stubs for all READ-01..26 tools
- [ ] `hpe-networking-mcp/tests/conftest.py` — add `aos8._registry` stub to `_install_registry_stubs()`
- [ ] `hpe-networking-mcp/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` — shared helper module (`strip_meta`, `run_show`, `get_object`, `format_aos8_error`)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `_meta` field actually stripped from live AOS8 response | READ-25 | No live AOS8 system in CI | Verify against live device or mock that returns `_meta` key |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
