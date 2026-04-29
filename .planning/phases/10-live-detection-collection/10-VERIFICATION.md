---
phase: 10-live-detection-collection
verified: 2026-04-29T10:00:00Z
status: human_needed
score: 6/6 must-haves verified (automated)
re_verification: false
human_verification:
  - test: "Scenario A — AOS8 reachable live-mode announcement"
    expected: "AI emits 'AOS8 API mode — live data' (or close paraphrase) before any Stage 0 question when health() returns aos8.status == 'ok'"
    why_human: "Requires a live AOS8 environment to trigger the detection branch. LLM prose interpretation of skill instructions cannot be verified statically. Operator confirmed prose is mechanically correct but cannot exercise Scenario A without an accessible Mobility Conductor."
---

# Phase 10: Live Detection & Collection — Verification Report

**Phase Goal:** When the operator runs the migration-readiness skill against an AOS8/Conductor source, the skill auto-detects connectivity and gathers every Stage 1 data point through live API tools — no CLI paste required.
**Verified:** 2026-04-29
**Status:** human_needed — all automated checks pass; one human scenario deferred
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Skill announces "AOS8 API mode — live data" at session start when AOS8 is reachable, or falls back to paste mode silently when not | ? HUMAN NEEDED | Stage -1 block exists at line 54 with the exact announcement text (line 60) and silent-fallback branch (line 65). Prose is mechanically correct. Runtime behavior unverifiable without live AOS8. Scenarios B (silent fallback) and C (AOS6/IAP paste regression) were operator-confirmed PASSED. |
| 2 | Skill retrieves MD hierarchy and per-node effective config via `aos8_get_md_hierarchy()` + `aos8_get_effective_config()` | ✓ VERIFIED | Both tools present in frontmatter (line 23) and Batch 1 live-mode narration (lines 99-100). Both tools confirmed in REGISTRIES via passing parametrized test `test_skill_aos8_live_detection.py`. |
| 3 | Skill retrieves full AP inventory via `aos8_get_ap_database()` | ✓ VERIFIED | Tool present in frontmatter (line 23) and Batch 2 narration (line 104). `@tool(name="aos8_get_ap_database")` confirmed at `health.py:45`. Parametrized test passes. |
| 4 | Skill retrieves cluster state, running-config, and local-user db via `aos8_get_cluster_state()` + `aos8_show_command()` | ✓ VERIFIED | Both tools in frontmatter (line 23) and Batch 3 narration (lines 107-111). `@tool` decorators confirmed at `differentiators.py:156` and `troubleshooting.py:81`. Parametrized test passes. |
| 5 | Skill retrieves client baseline, BSS/SSID table, active AP RF state, and AP wired ports via `aos8_get_clients()` / `aos8_get_bss_table()` / `aos8_get_active_aps()` / `aos8_get_ap_wired_ports()` | ✓ VERIFIED | All four tools in frontmatter (line 23) and Batch 4 narration (lines 114-117). All four `@tool` decorators confirmed. Parametrized test passes. |
| 6 | AOS6 and IAP paste paths unchanged — no regression | ✓ VERIFIED | `If source = \`aos6\`` at line 148 and `If source = \`iap\`` at line 162. IAP 10-command table intact with `IAP cluster Virtual Controller` rows. `grep` returns 18 hits for the 16-command paste-fallback CLI markers. Full 790-test suite passes with 0 regressions. |

**Score:** 5/6 truths verified automated + 1 human-needed (Scenario A)

---

### Required Artifacts

#### Plan 10-01 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `hpe-networking-mcp/tests/unit/test_skill_tool_references.py` | Updated regex including `aos8` alternation | ✓ VERIFIED | Line 39: `re.compile(r"\b(mist\|central\|greenlake\|clearpass\|apstra\|axis\|aos8)_[a-z_][a-z_0-9]+\b")`. `"aos8"` present in both import loop (line 104) and reload loop (line 130). File is 196 lines, substantive. |
| `hpe-networking-mcp/tests/unit/test_skill_aos8_live_detection.py` | Phase 10 AOS8 tool-presence test, >= 30 lines | ✓ VERIFIED | File exists, 62 lines. Contains `REQUIRED_AOS8_TOOLS = (` tuple with all 9 tool names. Contains `from tests.unit.test_skill_tool_references import _build_full_catalog`. Contains `@pytest.mark.parametrize("tool_name", REQUIRED_AOS8_TOOLS)`. |

#### Plan 10-02 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` | Updated skill with Stage -1 detection, AOS8 Stage 1 live procedure, frontmatter additions | ✓ VERIFIED | File is 530 lines (at ceiling). Contains `AOS8 API mode` at line 60. All 9 AOS8 tools in `tools:` list at line 23 and in skill body. |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `test_skill_tool_references.py` regex | `aos-migration-readiness.md` aos8_* references | alternation `\|aos8` in `_TOOL_REF_PATTERN` | ✓ WIRED | Line 39 contains `\|aos8)`. Regex test suite passes: 18 tests, 0 failures. |
| `test_skill_aos8_live_detection.py` | REGISTRIES['aos8'] catalog | `_build_full_catalog()` reuse via import | ✓ WIRED | `from tests.unit.test_skill_tool_references import _build_full_catalog` at line 13. 9-tool parametrized test passes — all 9 names resolve in the live catalog. |
| Skill Stage -1 detection block | `health()` tool | session-start probe before Stage 0 | ✓ WIRED | Line 56: `call \`health()\` once and inspect the per-platform status`. `health` already in frontmatter `tools:` list (line 23, position 1). |
| Skill Stage 1 AOS8 Batch 1 | `aos8_get_md_hierarchy` + `aos8_get_effective_config` | live-mode batch narration | ✓ WIRED | Lines 99-100 — both calls present with correct argument signatures. |
| Skill Stage 1 AOS8 Batch 2 | `aos8_get_ap_database` | live-mode batch narration | ✓ WIRED | Line 104 — call present with expected response fields documented. |
| Skill Stage 1 AOS8 Batch 3 | `aos8_get_cluster_state` + `aos8_show_command` | live-mode batch narration | ✓ WIRED | Lines 107-111 — `aos8_get_cluster_state()` at step 1; `aos8_show_command()` called 5 times for all COLLECT-03 commands. |
| Skill Stage 1 AOS8 Batch 4 | `aos8_get_clients` + `aos8_get_bss_table` + `aos8_get_active_aps` + `aos8_get_ap_wired_ports` | live-mode batch narration | ✓ WIRED | Lines 114-117 — all four tools called in sequence. `aos8_get_ap_wired_ports` correctly marked per-AP with iteration instruction. |
| Frontmatter `tools:` list | 9 AOS8 tool names | YAML list addition | ✓ WIRED | Line 23 — all 9 tools appended to existing 19-tool list (28 total). |

---

### Data-Flow Trace (Level 4)

This phase delivers a skill markdown file (not a Python component rendering dynamic data). Level 4 data-flow trace does not apply — the skill is a prose runbook interpreted by the LLM at runtime, not a code path with wired state variables. The LLM-prose-to-tool-call path is the subject of the human verification item (Scenario A).

---

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Skill regression tests pass (Plan 01: regex + 9-tool catalog) | `uv run pytest tests/unit/test_skill_tool_references.py tests/unit/test_skill_aos8_live_detection.py -x -q` | 18 passed in 4.97s | ✓ PASS |
| Full unit suite — no regressions | `uv run pytest tests/unit/ -x -q` | 790 passed in 55.70s | ✓ PASS |
| Skill file within 530-line ceiling | `wc -l src/hpe_networking_mcp/skills/aos-migration-readiness.md` | 530 | ✓ PASS (at ceiling, within target) |
| AOS6 section preserved | `grep -c "If source = \`aos6\`" skill file` | 1 | ✓ PASS |
| IAP section preserved | `grep -c "If source = \`iap\`" skill file` | 1 | ✓ PASS |
| 16-command paste-fallback bundle present | `grep -c "show lc-cluster\|show ap database long\|show configuration node-hierarchy\|show controller-ip"` | 18 hits | ✓ PASS (all 4 unique anchors present) |

---

### Requirements Coverage

| Requirement | Source Plan(s) | Description | Status | Evidence |
|-------------|----------------|-------------|--------|----------|
| DETECT-01 | 10-01, 10-02 | Skill detects AOS8 connectivity at session start and announces API mode vs paste-mode fallback | ? PARTIAL — automated prose correct; runtime behavior human-pending | Stage -1 block at line 54; announcement text at line 60; silent-fallback at line 65. Scenarios B+C passed; Scenario A deferred. |
| COLLECT-01 | 10-01, 10-02 | `aos8_get_md_hierarchy()` + `aos8_get_effective_config()` replace CLI paste | ✓ SATISFIED | Both tools in frontmatter and Batch 1 narration. Tools confirmed in REGISTRIES. Test passes. |
| COLLECT-02 | 10-01, 10-02 | `aos8_get_ap_database()` replaces `show ap database long` | ✓ SATISFIED | Tool in frontmatter and Batch 2 narration. Tool confirmed in REGISTRIES. Test passes. |
| COLLECT-03 | 10-01, 10-02 | `aos8_get_cluster_state()` + `aos8_show_command()` replace 6 CLI commands | ✓ SATISFIED | Both tools in frontmatter and Batch 3 narration (5 `aos8_show_command` calls covering all COLLECT-03 targets). Tools confirmed in REGISTRIES. Test passes. |
| COLLECT-04 | 10-01, 10-02 | `aos8_get_clients()` / `aos8_get_bss_table()` / `aos8_get_active_aps()` / `aos8_get_ap_wired_ports()` replace 5 CLI commands | ✓ SATISFIED | All four tools in frontmatter and Batch 4 narration. Tools confirmed in REGISTRIES. Test passes. |

**Orphaned requirement check:** REQUIREMENTS.md traceability table maps DETECT-01 and COLLECT-01..04 exclusively to Phase 10. All 5 are claimed by Plans 10-01 and 10-02. No orphaned requirements found for this phase.

---

### Anti-Patterns Found

| File | Pattern | Severity | Assessment |
|------|---------|----------|------------|
| `aos-migration-readiness.md` line 120 | Post-batch summary paragraph was identified as a trim candidate and removed, but a shortened version remains: `After all four batches, present a compiled Stage 1 inventory table...` | Info | Not a stub — this is operator-facing instruction, not a code placeholder. Present at line 120 (1 line vs the 3-line paragraph in the plan's un-trimmed block). Trim #4 was applied partially — one-sentence form retained. Acceptable per plan's trim priority guidance. |
| `test_skill_tool_references.py` line 146 | `for platform in ("mist", "central", "greenlake", "clearpass", "apstra", "axis"):` — meta-tool loop does NOT include `"aos8"` | Info | This loop adds dynamic meta-tools (`<platform>_list_tools`, etc.) to the catalog. `aos8` is absent because the AOS8 platform does not expose dynamic mode meta-tools (intentional — AOS8 uses static tool mode). Not a defect; the 9 required AOS8 tools are in REGISTRIES and resolve correctly without meta-tool entries. |

No blocker anti-patterns found.

---

### Human Verification Required

#### 1. Scenario A — AOS8 Reachable Live-Mode Announcement

**Test:** Configure AOS8 secrets (`aos8_host`, `aos8_username`, `aos8_password`) in the `secrets/` directory. Start the server with `SECRETS_DIR=./secrets uv run python -m hpe_networking_mcp` from `hpe-networking-mcp/`. From an MCP client, call `skills_load(name="aos-migration-readiness")` and prompt: "I want to do an AOS8 to AOS10 migration readiness audit."

**Expected:** The AI's first response includes the announcement "AOS8 API mode — live data. Stage 1 collection will run via API; no CLI paste required for the AOS8 source path." (or a close paraphrase containing "AOS8 API mode" and "live data") BEFORE any Stage 0 interview question.

**Why human:** Requires a reachable AOS8 Mobility Conductor at session start so `health()` returns `aos8.status == "ok"`. The detection branch and announcement behavior are LLM prose interpretation of skill instructions — cannot be unit-tested statically. The operator confirmed during Plan 10-02 Task 3 checkpoint that a live AOS8 environment is not currently available. Scenarios B (silent fallback when AOS8 unreachable) and C (AOS6/IAP paste table unchanged) were PASSED by the operator; only Scenario A is deferred.

---

### Gaps Summary

No automated gaps. All test infrastructure artifacts exist, are substantive, and are wired. All 9 AOS8 tool names resolve in the live catalog. All 4 COLLECT-* requirements are mechanically satisfied in the skill body. The AOS6 and IAP paste paths are confirmed unchanged. The full 790-test suite passes with no regressions.

The one open item is DETECT-01's runtime behavioral guarantee — the prose is mechanically correct and follows the skill instruction clarity standard established in prior phases, but the announcement behavior (Scenario A) requires an accessible Mobility Conductor to confirm empirically. This is classified as human_verification, not a gap, per the task instruction provided.

---

_Verified: 2026-04-29_
_Verifier: Claude (gsd-verifier)_
