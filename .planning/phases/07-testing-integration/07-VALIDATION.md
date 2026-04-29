---
phase: 7
slug: testing-integration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-28
---

# Phase 7 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4+ with pytest-asyncio (asyncio_mode=auto) |
| **Config file** | `pyproject.toml` `[tool.pytest.ini_options]` |
| **Quick run command** | `uv run pytest tests/unit/test_aos8_client.py tests/unit/platforms/aos8/ -x -q` |
| **Full suite command** | `uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `uv run pytest tests/unit/platforms/aos8/ -x -q`
- **After every plan wave:** Run `uv run pytest tests/unit/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 07-01-01 | 01 | 0 | TEST-01 | unit | `uv run pytest tests/unit/test_aos8_client.py -x -q` | ✅ | ⬜ pending |
| 07-01-02 | 01 | 0 | TEST-02 | unit | `uv run pytest tests/unit/platforms/aos8/ -x -q` | ❌ W0 | ⬜ pending |
| 07-01-03 | 01 | 0 | TEST-03 | unit | `uv run pytest tests/unit/platforms/aos8/test_aos8_write.py -x -q` | ✅ | ⬜ pending |
| 07-02-01 | 02 | 1 | TEST-02 | unit | `uv run pytest tests/unit/platforms/aos8/test_aos8_read_differentiators.py -x -q` | ❌ W0 | ⬜ pending |
| 07-02-02 | 02 | 1 | TEST-04 | unit | `uv run pytest tests/unit/platforms/aos8/test_aos8_security.py -x -q` | ❌ W0 | ⬜ pending |
| 07-03-01 | 03 | 2 | TEST-05 | unit | `uv run pytest tests/unit/ -x -q` | ✅ | ⬜ pending |
| 07-03-02 | 03 | 2 | TEST-06 | unit | `uv run pytest tests/unit/ -q` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `tests/unit/platforms/aos8/test_aos8_read_differentiators.py` — stubs for TEST-02 (DIFF-01..09 tool tests)
- [ ] `tests/unit/platforms/aos8/test_aos8_security.py` — stubs for TEST-04 (token-leak audit)
- [ ] `tests/unit/platforms/aos8/__init__.py` — ensure package init exists if missing

*Existing `tests/unit/test_aos8_client.py`, `test_aos8_write.py`, `test_aos8_config.py` satisfy TEST-01/TEST-03/TEST-05 foundations; stubs only needed for the two new files above.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| CI pipeline passes end-to-end | TEST-06 | Requires GitHub Actions runner | Push to branch, observe `ci.yml` green checks |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
