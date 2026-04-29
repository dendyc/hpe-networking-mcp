---
phase: 11
slug: live-vsg-rules
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-29
---

# Phase 11 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4.0+ with pytest-asyncio |
| **Config file** | `hpe-networking-mcp/pyproject.toml` (`[tool.pytest.ini_options]`) |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit -q` |
| **Estimated runtime** | ~15 seconds (quick) / ~60 seconds (full) |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 11-01-01 | 01 | 1 | RULES-01, RULES-02, RULES-03, RULES-04 | unit (regression) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. `tests/unit/test_skill_tool_references.py` already exists and was extended in Phase 10 with `|aos8` regex coverage. No new test files, fixtures, or framework setup needed.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Live-mode sub-path executes correctly against a real AOS8 controller | RULES-01..04 | No live AOS8 environment available (CLAUDE.md constraint; Scenario A deferred from Phase 10) | Requires operator to run the skill against a live AOS8/Mobility Conductor and confirm REGRESSION/DRIFT findings are produced for each rule |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
