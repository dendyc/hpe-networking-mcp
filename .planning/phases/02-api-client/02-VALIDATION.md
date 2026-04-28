---
phase: 2
slug: api-client
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-28
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.x + pytest-asyncio (asyncio_mode = "auto") |
| **Config file** | `pyproject.toml` (`[tool.pytest.ini_options]`) |
| **Quick run command** | `uv run pytest tests/unit/platforms/aos8/ -x -q` |
| **Full suite command** | `uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~15 seconds (full unit suite) |

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
| 2-01-01 | 01 | 0 | CLIENT-01 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py -x -q` | ❌ W0 | ⬜ pending |
| 2-01-02 | 01 | 1 | CLIENT-01 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_login -x -q` | ❌ W0 | ⬜ pending |
| 2-01-03 | 01 | 1 | CLIENT-02 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_token_reuse -x -q` | ❌ W0 | ⬜ pending |
| 2-01-04 | 01 | 1 | CLIENT-03 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_401_refresh -x -q` | ❌ W0 | ⬜ pending |
| 2-01-05 | 01 | 1 | CLIENT-04 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_concurrent_login -x -q` | ❌ W0 | ⬜ pending |
| 2-01-06 | 01 | 1 | CLIENT-05 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_lazy_login -x -q` | ❌ W0 | ⬜ pending |
| 2-01-07 | 01 | 1 | CLIENT-06 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_no_token_in_logs -x -q` | ❌ W0 | ⬜ pending |
| 2-01-08 | 01 | 1 | CLIENT-07 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_global_result_error -x -q` | ❌ W0 | ⬜ pending |
| 2-01-09 | 01 | 1 | CLIENT-08 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_ssl_verify -x -q` | ❌ W0 | ⬜ pending |
| 2-02-01 | 02 | 2 | CLIENT-09 | unit | `uv run pytest tests/unit/platforms/aos8/test_client.py::test_get_method -x -q` | ❌ W0 | ⬜ pending |
| 2-03-01 | 03 | 2 | CLIENT-10 | unit | `uv run pytest tests/unit/test_health.py -k aos8 -x -q` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `tests/unit/platforms/aos8/__init__.py` — test package init
- [ ] `tests/unit/platforms/aos8/test_client.py` — test stubs for CLIENT-01 through CLIENT-09
- [ ] `tests/unit/platforms/aos8/conftest.py` — MockTransport fixtures (httpx.MockTransport pattern from `test_apstra_client.py`)

*Existing `pytest-asyncio`, `responses`, and `httpx` infrastructure covers all framework requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `_global_result.status` string vs int normalization across AOS8 firmware versions | CLIENT-07 | No live firmware available; mock fixtures cover known shapes only | If live system available: call a known-good endpoint, inspect raw JSON shape, verify `str(result) != "0"` works for both int and string variants |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
