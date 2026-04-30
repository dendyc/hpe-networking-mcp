---
phase: 10
slug: live-detection-collection
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-29
approved: 2026-04-29
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
| 10-01-01 | 01 | 1 | DETECT-01 / COLLECT-01..04 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x -q` | ✅ | ⬜ pending |
| 10-01-02 | 01 | 1 | DETECT-01 / COLLECT-01..04 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_aos8_live_detection.py -x -q` | ✅ | ⬜ pending |
| 10-02-01 | 02 | 2 | DETECT-01 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py tests/unit/test_skill_aos8_live_detection.py tests/unit/test_skills.py -x -q` | ✅ | ⬜ pending |
| 10-02-02 | 02 | 2 | COLLECT-01..04 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py tests/unit/test_skill_aos8_live_detection.py tests/unit/test_skills.py -x -q` | ✅ | ⬜ pending |
| 10-02-03 | 02 | 2 | DETECT-01 (LLM-prose) | manual | checkpoint:human-verify (3 scenarios) | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [x] `tests/unit/test_skill_aos8_live_detection.py` — stubs for DETECT-01, COLLECT-01 through COLLECT-04 (AOS8 live-mode detection + collection) — addressed by Plan 10-01 Task 2
- [x] Fix `_TOOL_REF_PATTERN` regex in `tests/unit/test_skill_tool_references.py` to include `aos8` alternation — addressed by Plan 10-01 Task 1

*Wave 0 work is delivered by Plan 10-01 (Wave 1). Plan 10-02 (Wave 2) consumes those test guards.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Skill announces "AOS8 API mode — live data" at session start | DETECT-01 | Runtime LLM message content — not statically assertable | Run skill with AOS8 configured; verify session opening banner |
| Graceful fallback to paste mode when AOS8 unreachable | DETECT-01 | Requires live connectivity failure to trigger | Disable AOS8 secrets and run skill; confirm paste prompt appears |
| AOS6 and IAP paste paths execute unchanged | COLLECT-04 (regression) | Requires running the full skill flow with AOS6/IAP source | Run skill with AOS6 or IAP source; confirm paste prompts appear unchanged |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references (delivered by Plan 10-01 Wave 1)
- [x] No watch-mode flags
- [x] Feedback latency < 15s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-04-29 (post-checker revision; Plan 10-01 delivers Wave 0 test guards in Wave 1, consumed by Plan 10-02 in Wave 2)
