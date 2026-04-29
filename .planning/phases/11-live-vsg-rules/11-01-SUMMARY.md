---
phase: 11-live-vsg-rules
plan: "01"
subsystem: skill
tags: [aos8, migration, vsg-rules, live-mode, markdown-only]
dependency_graph:
  requires: [10-02]
  provides: [RULES-01, RULES-02, RULES-03, RULES-04]
  affects: [hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md]
tech_stack:
  added: []
  patterns: [live-mode sub-path conditional structure, per-rule VSG-anchored finding format, Stage 1 batch citation]
key_files:
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md
decisions:
  - D-02 honored: live-mode sub-path inserted before Universal rules, not after
  - D-03 honored: Stage 2 skip clause narrows to "AOS8 data points only" (not "skip Stage 2 entirely")
  - D-04 honored: RULES-03 deferred to Stage 4 A11 — no clearpass_get_local_users() call in Stage 3
  - D-05 honored: RULES-01 references LMS IP field by intent (not pinned JSON key); ip_mode used by name for RULES-04
  - Open Question 1 disposition: LMS IP referenced by intent ("the LMS IP field in the ap_sys_prof response") — JSON key not pinned per research Pitfall 1
metrics:
  duration_minutes: 15
  completed_date: "2026-04-29"
  tasks_completed: 3
  tasks_total: 3
  files_modified: 1
---

# Phase 11 Plan 01: Live VSG Rules — AOS8 Stage 2/3/4 Prose Summary

**One-liner:** Layered AOS8 live-mode rule evaluation (RULES-01..04) into aos-migration-readiness.md so Stage 3 consumes Stage 1 batch data for REGRESSION/DRIFT findings, with RULES-03 cross-check deferred to Stage 4 A11.

## What Was Done

This plan is **markdown-only** — no Python code, no new tools, no frontmatter changes. Three localized edits to one file.

### Edit 1 — Stage 2 AOS8 Live-Mode Skip Clause (line 183)

Inserted a `##### AOS8 live-mode sub-path` block immediately before `#### AOS 8 — what to extract from each artifact` (now at line 187). The skip clause tells the AI to bypass AOS8 paste parsing when Stage -1 announced API mode, while preserving Stage 2 for AOS6/IAP source paths.

**Inserted block (4 lines, after line 182):**
- `##### AOS8 live-mode sub-path — used when Stage -1 announced "AOS8 API mode"` (heading)
- Skip prose: "Skip paste parsing for AOS8 data points and proceed directly to Stage 3."
- Clarification that AOS8 extraction table remains authoritative for paste-fallback/AOS6/IAP

### Edit 2 — Stage 3 AOS8 Live-Mode Sub-Path (lines 230–276)

Inserted a new `#### AOS8 live-mode sub-path — rules evaluated from Stage 1 data` section immediately after the Stage 3 intro paragraph and before `#### Universal rules`. Contains four per-rule sub-sections:

- **RULES-01** (line ~233): VRRP VIP check from Batch 1 `aos8_get_effective_config(object_name='ap_sys_prof', config_path='/md')`. REGRESSION if LMS IP matches individual controller IP. Batch-failure fallback note. Equivalent paste-mode rule: C2.
- **RULES-02** (line ~247): ARM/dot11a/dot11g/reg-domain profile detection from Batch 1 effective-config. DRIFT if any profile type has at least one configured object at `/md` root. Distinguishes "envelope present" vs "objects present". Batch-failure fallback note. Equivalent paste-mode rule: C4.
- **RULES-03** (line ~261): Explicitly deferred to Stage 4 A11. No ClearPass call here. Notes that Batch 3 count is already in context; A11 will do the cross-check.
- **RULES-04** (line ~269): Static AP IP detection from Batch 2 `aos8_get_ap_database()` `ip_mode` field. REGRESSION per AP with non-DHCP ip_mode. Batch-failure fallback note. Equivalent paste-mode rule: U2.

Each rule includes: VSG anchor in `(VSG §####-§####)` form, source tool citation with `(source: tool_name(args), Batch N)`, and batch-failure fallback.

### Edit 3 — Stage 4 A11 RULES-03 Cross-Check Expansion (line 376)

Replaced the A11 row in-place. The new row:
- Performs the AOS8-vs-ClearPass local-user count cross-check using Stage 1 Batch 3 data already in context
- Emits DRIFT when both counts are non-zero, with explicit finding format
- Cites VSG §1134-§1136 (no Internal Auth Server in AOS 10)
- Notes fallback message when AOS8 Batch 3 was unavailable
- Tool column still contains `clearpass_get_local_users()` (required for regression test)
- Row stays on a single line (table parses correctly)

## Paste-Mode Rules Unchanged

The following rows are byte-identical to their pre-plan state:

- **C2** (line ~295): `| C2 | **AP discovery configured properly** ... | VSG §1651-§1657 |`
- **C4** (line ~297): `| C4 | **ARM Profiles / Dot11a/g Radio Profiles...** | VSG §412-§418, §1163-§1169 |`
- **U2** (line ~279): `| U2 | DHCP available for AP IP assignment ... | VSG §1232-§1234, §475-§477 |`

These remain authoritative for the paste-fallback path (AOS6, IAP, and unreachable-AOS8 sessions).

## Frontmatter Verification

No `tools:` or `platforms:` frontmatter changes were needed. All tools referenced in the new prose (`aos8_get_effective_config`, `aos8_get_ap_database`, `aos8_show_command`, `aos8_get_md_hierarchy`, `clearpass_get_local_users`) were already listed in the skill's `tools:` frontmatter from Phase 10.

## Open Question 1 Disposition

LMS IP field name in `ap_sys_prof` API response: referenced by intent ("the LMS IP field in the `ap_sys_prof` response") rather than a pinned JSON key path. This is the correct approach per Research Pitfall 1 — no live AOS8 box is available to verify the exact JSON key (`lms_ip`, `lms-ip`, etc.). The AI client introspects the response structure at runtime.

## Test Results

- **Skill regression test (after each task):** `tests/unit/test_skill_tool_references.py` — 8/8 passed
- **Full unit suite (after Task 3):** 790/790 passed in 46.65s

## Deviations from Plan

None — plan executed exactly as written. All three insertions used the verbatim action text from the plan's `<action>` sections.

## Deferred Manual Verification (Scenario A)

Live AOS8 end-to-end execution is not possible — no live AOS8/Mobility Conductor environment is available (CLAUDE.md constraint; Phase 10 Scenario A deferral precedent). Mechanical correctness verified:

- Stage 2 has skip clause narrowed to AOS8 data points
- Stage 3 has live-mode sub-path before Universal rules with all four RULES-0X sections
- RULES-03 in Stage 3 defers to A11; A11 performs the cross-check
- Every new tool reference uses the exact catalog name
- Every new finding cites its VSG anchor and source tool
- C2/C4/U2 paste-mode rules are unchanged

## Commits

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Stage 2 AOS8 live-mode skip clause | beb0487 | aos-migration-readiness.md (+4 lines) |
| 2 | Stage 3 AOS8 live-mode sub-path block | 5af916b | aos-migration-readiness.md (+44 lines) |
| 3 | Stage 4 A11 RULES-03 cross-check expansion | 7f6a534 | aos-migration-readiness.md (+1/-1 lines) |

## Self-Check: PASSED

- [x] aos-migration-readiness.md modified and committed
- [x] All three commits exist in hpe-networking-mcp repo
- [x] test_skill_tool_references.py passes (8/8)
- [x] Full unit suite passes (790/790)
- [x] Stage 2 skip clause present at line 183 (2 occurrences of Stage -1 heading = Stage 1 + Stage 2)
- [x] Stage 3 live-mode sub-path present at line 230, before Universal rules at line 278
- [x] Stage 4 A11 expanded in place at line 376
- [x] No tools:/platforms: frontmatter changes
- [x] C2/C4/U2 paste-mode rules unchanged
