---
phase: 8
slug: fix-diff-tools-production-bug
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-28
---

# Phase 8 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4.0 + pytest-asyncio (asyncio_mode="auto") |
| **Config file** | `hpe-networking-mcp/pyproject.toml` |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py -q` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit -q` |
| **Estimated runtime** | ~15 seconds (full unit suite) |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py -q`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 8-01-01 | 01 | 1 | DIFF-01..09 | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py -q` | ✅ | ⬜ pending |
| 8-01-02 | 01 | 1 | DIFF-01..09 | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py -q` | ✅ | ⬜ pending |
| 8-01-03 | 01 | 1 | DIFF-01..09 | lint | `uv run ruff check src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py && uv run mypy src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` | ✅ | ⬜ pending |
| 8-01-04 | 01 | 2 | all | regression | `uv run pytest tests/unit -q` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements.* No new test files or framework installs needed — `test_aos8_read_differentiators.py` already exists with 13 tests; this phase corrects the mocks within that file.

---

## Manual-Only Verifications

*All phase behaviors have automated verification.*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
