---
phase: 7
slug: testing-integration
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-04-28
revised: 2026-04-28
---

# Phase 7 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4+ with pytest-asyncio (asyncio_mode=auto) |
| **Config file** | `pyproject.toml` `[tool.pytest.ini_options]` |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit -k aos8 -x -q` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit -k aos8 -x -q`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

Wave numbering aligns with PLAN frontmatter (`wave: 1` = Plan 07-01, `wave: 2` = Plan 07-02, `wave: 3` = Plan 07-03).

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 07-01-01 | 01 | 1 | TEST-02 | unit (fixtures) | `cd hpe-networking-mcp && python -c "import json, pathlib; [json.loads(p.read_text()) for p in pathlib.Path('tests/unit/fixtures/aos8').glob('*.json')]"` | ❌ W0 | ⬜ pending |
| 07-01-02 | 01 | 1 | TEST-02 | unit (collect-only RED) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py --collect-only -q` | ❌ W0 | ⬜ pending |
| 07-01-03 | 01 | 1 | TEST-04 | unit (collect-only) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_security.py --collect-only -q` | ❌ W0 | ⬜ pending |
| 07-01-04 | 01 | 1 | TEST-01/03/05 | unit (name-presence scan) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_init.py --collect-only -q && uv run pytest tests/unit/test_aos8_client.py tests/unit/test_aos8_write.py tests/unit/test_aos8_config.py --collect-only -q \| tee /tmp/aos8_collect.txt && grep -q "login_success" /tmp/aos8_collect.txt && grep -q "token_reuse" /tmp/aos8_collect.txt && grep -q "401_refresh" /tmp/aos8_collect.txt && grep -q "global_result_error" /tmp/aos8_collect.txt && grep -q "verify_ssl" /tmp/aos8_collect.txt && grep -qi "token" /tmp/aos8_collect.txt && grep -q "config_path" /tmp/aos8_collect.txt && (grep -q "auto_disabled" /tmp/aos8_collect.txt \|\| grep -q "disabled_when_secrets_missing" /tmp/aos8_collect.txt)` | ✅ | ⬜ pending |
| 07-02-01 | 02 | 2 | TEST-02 | unit (DIFF-01..08 GREEN) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py -k "not health_check" -x -q` | ❌ W1 | ⬜ pending |
| 07-02-02 | 02 | 2 | TEST-02/04 | unit (DIFF-09 + security GREEN) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py tests/unit/test_aos8_security.py -x -q` | ❌ W1 | ⬜ pending |
| 07-03-01 | 03 | 3 | TEST-06 | unit (init wired) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_init.py -v` | ✅ | ⬜ pending |
| 07-03-02 | 03 | 3 | TEST-06 | unit (full regression) | `cd hpe-networking-mcp && uv run pytest tests/unit -q` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

*"File Exists" column: ❌ W0 means file is created in Wave 1 (Plan 07-01); ❌ W1 means file is created in Wave 2 (Plan 07-02).*

---

## Wave 0 (Plan 07-01) Requirements

These artifacts MUST exist after Plan 07-01 executes; Plan 07-02 depends on them:

- [ ] `tests/unit/test_aos8_read_differentiators.py` — RED test scaffold for DIFF-01..09 (TEST-02)
- [ ] `tests/unit/test_aos8_security.py` — RED tool-layer token-leak audit (TEST-04)
- [ ] `tests/unit/test_aos8_init.py` — updated EXPECTED_DIFF_TOOLS / EXPECTED_TOTAL=47 / "differentiators":9
- [ ] 9 fixture JSON files under `tests/unit/fixtures/aos8/` (show_switch_hierarchy.json, show_running_config.json, show_pending_config.json, show_ap_arm_neighbors.json, show_lc_cluster_group.json, show_ap_monitor_active.json, show_ap_port_status.json, show_crypto_ipsec_sa.json, show_user_summary.json)

*Existing `tests/unit/test_aos8_client.py`, `tests/unit/test_aos8_write.py`, `tests/unit/test_aos8_config.py` satisfy TEST-01/TEST-03/TEST-05 foundations; Plan 07-01 Task 4 verifies the required test names are present via an automated grep scan that fails the task if any are missing.*

*All test files live directly under `tests/unit/` (flat layout — no `tests/unit/platforms/aos8/` subdirectory). Plan `<files>` paths are authoritative.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| CI pipeline passes end-to-end | TEST-06 | Requires GitHub Actions runner | Push to branch, observe `ci.yml` green checks |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references (paths corrected to `tests/unit/`)
- [x] No watch-mode flags
- [x] Feedback latency < 30s
- [x] Wave numbering aligned with PLAN frontmatter (1/2/3, not 0/1/2)
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending review of revised paths
