---
phase: 13
slug: executive-output-quality-gate
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-29
---

# Phase 13 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest >= 8.4.0 |
| **Config file** | `hpe-networking-mcp/pyproject.toml` (`[tool.pytest.ini_options]`, `asyncio_mode = "auto"`) |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -k aos-migration-readiness -v` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` |
| **Estimated runtime** | ~5 seconds (quick) / ~60–90 seconds (full) |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -v`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 5 seconds (quick run)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 13-01-01 | 01 | 1 | QUALITY-02 | manual + smoke | `python -c "import yaml; t = open('src/hpe_networking_mcp/skills/aos-migration-readiness.md', encoding='utf-8').read(); fm = yaml.safe_load(t.split('---', 2)[1]); assert 'aos8' in fm['platforms'], fm['platforms']"` | ✅ | ⬜ pending |
| 13-01-02 | 01 | 1 | OUTPUT-01 | manual-only | visual review of skill diff (exec summary paragraph before `## AOS migration readiness`) | n/a | ⬜ pending |
| 13-01-03 | 01 | 1 | OUTPUT-02 | manual-only | visual review of skill diff (output hygiene subsection with 4 prohibitions) | n/a | ⬜ pending |
| 13-01-04 | 01 | 1 | OUTPUT-01 | manual-only | visual review — PARTIAL row added to decision matrix per D-07 | n/a | ⬜ pending |
| 13-01-05 | 01 | 1 | QUALITY-01, QUALITY-03 | unit | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py::TestSkillToolReferences::test_skill_references_resolve -k aos-migration-readiness -v` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements.* The regression test, `_TOOL_REF_PATTERN`, and catalog builder were all completed in Phase 10. No new test files are needed.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Exec summary paragraph instruction at top of report template (before `## AOS migration readiness —` header) | OUTPUT-01 | AI runtime behavior; content correctness cannot be unit-tested without live model invocation | Read diff: confirm exec summary instruction is inside the code-fence block BEFORE `## AOS migration readiness —` line; confirm it uses angle-bracket placeholders (not hard-coded values); confirm it is 2–4 sentence prose instruction (not a bullet list) |
| Output hygiene rules subsection under "Output formatting" with all 4 prohibitions | OUTPUT-02 | Content review only | Read diff: confirm `### Output hygiene (mandatory)` subsection exists under `## Output formatting`; confirm 4 numbered rules covering: no raw JSON, no tool-call syntax in findings, no stack traces, no ellipsis/truncation |
| PARTIAL row for AOS8 live-mode batch failure in Stage 6 decision matrix | OUTPUT-01 (D-07) | Table content correctness | Read diff: confirm new row added after existing PARTIAL row (line ~505); row covers "AOS8 live mode AND one or more Stage 1 batches failed"; action begins with **PARTIAL**, lists failed checks, includes CLI commands |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 5s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
