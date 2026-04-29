---
phase: 12
slug: central-enrichment-cutover-validation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-29
---

# Phase 12 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest >= 8.4.0 |
| **Config file** | `hpe-networking-mcp/pyproject.toml` |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -q` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~10 seconds (skill reference test only); ~60 seconds (full unit suite) |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -q`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 12-01-01 | 01 | 1 | ENRICH-01, ENRICH-02, ENRICH-03, ENRICH-04 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -q` | ✅ | ⬜ pending |
| 12-02-01 | 02 | 2 | CUTOVER-01, CUTOVER-02, CUTOVER-03 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -q` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements.

- `tests/unit/test_skill_tool_references.py` — validates every tool name in skill body against catalog + `tools:` frontmatter (extended in Phase 10 to cover `aos8` prefixed tools)
- No new test files or fixtures needed — markdown-only edits; tool references validated by existing regex

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Stage 4 sub-path inserts above A1–A13 table (not below) | ENRICH-01..04 | Positional check not expressible as a unit test | Read `aos-migration-readiness.md` Stage 4 section; confirm `##### AOS8 live-mode sub-path — Central enrichment` appears before the first A1 table row |
| Stage 5 sub-path inserts before Phase 0–8 cutover table | CUTOVER-01..03 | Positional check not expressible as a unit test | Read `aos-migration-readiness.md` Stage 5 section; confirm `##### AOS8 live-mode sub-path — cutover prerequisites` appears before the Phase 0 row |
| ENRICH-02 uses single `central_recommend_firmware()` call (no per-model loop) | ENRICH-02 | Structural/architectural prose check | Confirm sub-path prose calls `central_recommend_firmware()` once with no filter; no iteration instruction present |
| CUTOVER-02 uses fresh `aos8_show_command(command='show version')` not Batch 3 | CUTOVER-02 | Intent check — tool reuse vs. fresh call | Confirm Stage 5 sub-path explicitly calls `aos8_show_command(command='show version')` rather than referencing Batch 3 context |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
