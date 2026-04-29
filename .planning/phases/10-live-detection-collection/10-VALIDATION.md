---
phase: 10
slug: live-detection-collection
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-29
---

# Phase 10 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.x |
| **Config file** | `pyproject.toml` (`[tool.pytest.ini_options]`) |
| **Quick run command** | `uv run pytest tests/unit/test_skill_tool_references.py -x -q` |
| **Full suite command** | `uv run pytest tests/unit/ -x -q` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `uv run pytest tests/unit/test_skill_tool_references.py -x -q`
- **After every plan wave:** Run `uv run pytest tests/unit/ -x -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 10-01-01 | 01 | 1 | DETECT-01 | unit | `uv run pytest tests/unit/ -k "skill" -x -q` | ✅ | ⬜ pending |
| 10-01-02 | 01 | 1 | COLLECT-01 | unit | `uv run pytest tests/unit/ -k "skill" -x -q` | ✅ | ⬜ pending |
| 10-01-03 | 01 | 1 | COLLECT-02 | unit | `uv run pytest tests/unit/ -k "skill" -x -q` | ✅ | ⬜ pending |
| 10-01-04 | 01 | 1 | COLLECT-03 | unit | `uv run pytest tests/unit/ -k "skill" -x -q` | ✅ | ⬜ pending |
| 10-01-05 | 01 | 1 | COLLECT-04 | unit | `uv run pytest tests/unit/ -k "skill" -x -q` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `tests/unit/test_skill_aos8_live_detection.py` — stubs for DETECT-01, COLLECT-01 through COLLECT-04 (AOS8 live-mode detection + collection)

*Note: Existing skill test infrastructure (`test_skill_tool_references.py`) covers basic tool-name validation but regex excludes `aos8_*` tools — Wave 0 must add coverage or fix regex.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Skill announces "AOS8 API mode — live data" at session start | DETECT-01 | Runtime LLM message content — not statically assertable | Run skill with AOS8 configured; verify session opening banner |
| Graceful fallback to paste mode when AOS8 unreachable | DETECT-01 | Requires live connectivity failure to trigger | Disable AOS8 secrets and run skill; confirm paste prompt appears |
| AOS6 and IAP paste paths execute unchanged | COLLECT-04 (regression) | Requires running the full skill flow with AOS6/IAP source | Run skill with AOS6 or IAP source; confirm paste prompts appear unchanged |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
