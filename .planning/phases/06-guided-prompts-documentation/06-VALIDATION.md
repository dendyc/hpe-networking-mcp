---
phase: 6
slug: guided-prompts-documentation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-28
---

# Phase 6 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 8.4.0+ with pytest-asyncio 1.2.0+ (`asyncio_mode = "auto"`) |
| **Config file** | `pyproject.toml` `[tool.pytest.ini_options]` |
| **Quick run command** | `uv run pytest tests/unit/test_aos8_prompts.py -x` |
| **Full suite command** | `uv run pytest tests/ -q` |
| **Estimated runtime** | ~2 seconds (quick) / ~30 seconds (full suite) |

---

## Sampling Rate

- **After every task commit:** Run `uv run pytest tests/unit/test_aos8_prompts.py -x`
- **After every plan wave:** Run `uv run pytest tests/ -q`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 2 seconds (quick), 30 seconds (full)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 6-01-01 | 01 | 1 | PROMPT-01..09 | unit (smoke) | `uv run pytest tests/unit/test_aos8_prompts.py -x` | ❌ Wave 0 creates it | ⬜ pending |
| 6-01-02 | 01 | 1 | PROMPT-01 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k triage_client` | ❌ Plan 06-01 creates it | ⬜ pending |
| 6-01-03 | 01 | 1 | PROMPT-02 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k triage_ap` | ❌ Plan 06-01 | ⬜ pending |
| 6-01-04 | 01 | 1 | PROMPT-03 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k health_check` | ❌ Plan 06-01 | ⬜ pending |
| 6-01-05 | 01 | 1 | PROMPT-04 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k audit_change` | ❌ Plan 06-01 | ⬜ pending |
| 6-01-06 | 01 | 1 | PROMPT-05 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k rf_analysis` | ❌ Plan 06-01 | ⬜ pending |
| 6-01-07 | 01 | 1 | PROMPT-06 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k wlan_review` | ❌ Plan 06-01 | ⬜ pending |
| 6-01-08 | 01 | 1 | PROMPT-07 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k client_flood` | ❌ Plan 06-01 | ⬜ pending |
| 6-01-09 | 01 | 1 | PROMPT-08 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k compare_md_config` | ❌ Plan 06-01 | ⬜ pending |
| 6-01-10 | 01 | 1 | PROMPT-09 | unit | `uv run pytest tests/unit/test_aos8_prompts.py -x -k pre_change_check` | ❌ Plan 06-01 | ⬜ pending |
| 6-02-01 | 02 | 2 | DOCS-01..05 | manual | reviewer eyeball pass on each doc diff | ✅ existing files; INSTRUCTIONS.md is new | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `tests/unit/test_aos8_prompts.py` — smoke tests covering PROMPT-01..09 (created in Plan 06-01)

*(No framework install needed — pytest infrastructure has been operational since Phase 1.)*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `INSTRUCTIONS.md` covers config_path semantics, write_memory contract, show command, Conductor vs standalone | DOCS-01 | doc completeness is human judgement | Read file, verify all 4 topics have dedicated sections |
| `README.md` has AOS8 row in capability table, secrets section, auto-disable example, updated tool counts | DOCS-02..03 | layout/accuracy requires visual review | Read README sections, verify AOS8 row present and 3 tool-count locations updated |
| `docs/TOOLS.md` lists all 38 AOS8 tools with descriptions and tags | DOCS-04 | tool list completeness requires review | Count AOS8 section entries, verify descriptions non-empty |
| `CHANGELOG.md` has 2.4.0.0 entry; `pyproject.toml` version is 2.4.0.0 | DOCS-05 | changelog accuracy is human review | Read CHANGELOG top entry; grep pyproject.toml for version |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
