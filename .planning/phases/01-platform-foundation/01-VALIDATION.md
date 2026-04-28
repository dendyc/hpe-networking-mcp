---
phase: 1
slug: platform-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-27
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4.0+ with pytest-asyncio 1.2.0+ (`asyncio_mode = "auto"`) |
| **Config file** | `hpe-networking-mcp/pyproject.toml` (`[tool.pytest.ini_options]`) |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_config.py tests/unit/test_tool_registry.py tests/unit/test_config.py -x -q` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/ -q` |
| **Estimated runtime** | ~5 seconds (quick), ~60 seconds (full) |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_config.py tests/unit/test_config.py tests/unit/test_tool_registry.py -x -q`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit -q`
- **Before `/gsd:verify-work`:** Full suite must be green plus `ruff check . && ruff format --check . && mypy src/ --ignore-missing-imports`
- **Max feedback latency:** 5 seconds (quick), 60 seconds (wave)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01 | 01 | 1 | FOUND-01 | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_secrets_when_all_required_present -x` | ❌ W0 | ⬜ pending |
| 1-02 | 01 | 1 | FOUND-01 | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_port_defaults_to_4343_when_missing -x` | ❌ W0 | ⬜ pending |
| 1-03 | 01 | 1 | FOUND-01 | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_verify_ssl_defaults_true_when_missing -x` | ❌ W0 | ⬜ pending |
| 1-04 | 01 | 1 | FOUND-02 | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_none_when_host_missing -x` | ❌ W0 | ⬜ pending |
| 1-05 | 01 | 1 | FOUND-02 | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_none_when_username_missing -x` | ❌ W0 | ⬜ pending |
| 1-06 | 01 | 1 | FOUND-02 | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_none_when_password_missing -x` | ❌ W0 | ⬜ pending |
| 1-07 | 01 | 1 | FOUND-02 | unit | `uv run pytest tests/unit/test_config.py::TestLoadConfig::test_returns_config_with_only_mist_enabled -x` | ✅ (extend) | ⬜ pending |
| 1-08 | 01 | 1 | FOUND-03 | unit | extend `test_config.py::TestLoadConfig::test_reads_env_overrides` with `ENABLE_AOS8_WRITE_TOOLS=true` | ✅ (extend) | ⬜ pending |
| 1-09 | 01 | 1 | FOUND-03 | unit | `uv run pytest tests/unit/test_tool_registry.py::TestIsToolEnabled::test_aos8_write_gate -x` | ❌ W0 (extend file) | ⬜ pending |
| 1-10 | 01 | 1 | FOUND-03 | unit | `uv run pytest tests/unit/test_tool_registry.py::TestIsToolEnabled::test_aos8_write_delete_gate -x` | ❌ W0 (extend file) | ⬜ pending |
| 1-11 | 01 | 1 | FOUND-04 | unit | `uv run pytest tests/unit/test_tool_registry.py::TestRecordTool::test_records_into_the_right_platform -x` (extend with aos8 case) | ✅ (extend) | ⬜ pending |
| 1-12 | 01 | 1 | FOUND-05 | smoke | `python -c "import yaml; c=yaml.safe_load(open('docker-compose.yml')); assert 'aos8_host' in c['secrets']"` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `hpe-networking-mcp/tests/unit/test_aos8_config.py` — covers FOUND-01, FOUND-02 (10+ test methods, mirroring `test_apstra_config.py`)
- [ ] Extend `hpe-networking-mcp/tests/unit/test_tool_registry.py` — add `test_aos8_write_gate` and `test_aos8_write_delete_gate` (FOUND-03, FOUND-04)
- [ ] Extend `hpe-networking-mcp/tests/conftest.py` `secrets_dir` fixture — add 5 AOS8 keys (`aos8_host`, `aos8_username`, `aos8_password`, `aos8_port`, `aos8_verify_ssl`)
- [ ] Extend `hpe-networking-mcp/tests/unit/test_config.py::TestLoadConfig::test_returns_config_with_all_platforms_enabled` — add `aos8` assertion (FOUND-01 regression)
- [ ] Extend `hpe-networking-mcp/tests/unit/test_config.py::TestLoadConfig::test_returns_config_with_only_mist_enabled` — add 5 AOS8 files to unlink list (FOUND-02 regression)
- [ ] Extend `hpe-networking-mcp/tests/unit/test_config.py::TestLoadConfig::test_reads_env_overrides` — add `ENABLE_AOS8_WRITE_TOOLS` env check (FOUND-03)

*No framework install needed — pytest infrastructure already in place at `hpe-networking-mcp/tests/`.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| All 5 `.example` files exist with non-empty content | FOUND-05 | File existence check, not logic | `for f in aos8_host aos8_username aos8_password aos8_port aos8_verify_ssl; do test -s hpe-networking-mcp/secrets/$f.example || echo "MISSING: $f.example"; done` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 5s (quick), < 60s (wave)
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
