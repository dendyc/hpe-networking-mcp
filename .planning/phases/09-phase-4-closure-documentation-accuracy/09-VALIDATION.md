---
phase: 9
slug: phase-4-closure-documentation-accuracy
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-28
---

# Phase 9 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4+ with `pytest-asyncio` (asyncio_mode=auto) |
| **Config file** | `hpe-networking-mcp/pyproject.toml` `[tool.pytest.ini_options]` |
| **Quick run command** | `cd hpe-networking-mcp && uv run pytest tests/unit -k "aos8 or server" -q` |
| **Full suite command** | `cd hpe-networking-mcp && uv run pytest tests/unit -q` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_init.py tests/unit/test_aos8_read_differentiators.py tests/unit/test_server_code_mode.py -q`
- **After every plan wave:** Run `cd hpe-networking-mcp && uv run pytest tests/unit -q && uv run ruff check . && uv run ruff format --check . && uv run mypy src/ --ignore-missing-imports`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 9-01-01 | 01 | 0 | server fix | unit | `uv run pytest tests/unit/test_server_code_mode.py -v` | ❌ Wave 0 | ⬜ pending |
| 9-01-02 | 01 | 1 | server fix | unit | `uv run pytest tests/unit/test_server_code_mode.py::test_execute_description_lists_aos8_prefix -v` | ❌ Wave 0 | ⬜ pending |
| 9-01-03 | 01 | 1 | DIFF-01..09 | unit | `uv run pytest tests/unit/test_aos8_init.py -v` | ✅ | ⬜ pending |
| 9-01-04 | 01 | 1 | DIFF-01..09 | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py -v` | ✅ | ⬜ pending |
| 9-01-05 | 01 | 2 | DOCS-02 | manual+grep | `grep -nE "aos8.{0,40}38\|38.{0,40}aos8" hpe-networking-mcp/README.md` returns 0 | manual | ⬜ pending |
| 9-01-06 | 01 | 2 | DOCS-04 | manual+grep | `grep -nE "aos8_get_md_hierarchy" hpe-networking-mcp/docs/TOOLS.md` returns ≥1 | manual | ⬜ pending |
| 9-01-07 | 01 | 2 | DOCS-03 | manual | Visual review of CHANGELOG.md 2.4.0.1 entry | manual-only | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `hpe-networking-mcp/tests/unit/test_server_code_mode.py` — new test asserting `execute_description` lists all 7 platform prefixes including `aos8_`. Required because the literal-string drift pattern has no other automated guard.
- [ ] (Optional) `hpe-networking-mcp/tests/unit/test_aos8_docs.py` — doc-completeness assertion greping `docs/TOOLS.md` for 9 DIFF tool names. Recommended but noisy in CI; skip if team prefers manual gate.
- [ ] No framework install needed — pytest already configured.

*Existing test infrastructure covers DIFF-01..09 (test_aos8_init.py + test_aos8_read_differentiators.py).*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| CHANGELOG.md has a new [2.4.0.1] entry | DOCS-03 | No automated test for changelog content presence | Visual review: `head -40 hpe-networking-mcp/CHANGELOG.md` — confirm `## [2.4.0.1]` section exists above `## [2.4.0.0]` |
| 04-VERIFICATION.md cross-references Phase 7 | planning integrity | Planning artifact, not in test suite | Visual review: confirm `04-VERIFICATION.md` exists in `.planning/phases/04-differentiator-tools/` and references `07-VERIFICATION.md` Observable Truth #1 |
| REQUIREMENTS.md DIFF rows show [x] | DIFF-01..09 | Text-file edit, not executable | `grep -c "\[x\]" .planning/REQUIREMENTS.md` — count should increase by 9 compared to before |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
