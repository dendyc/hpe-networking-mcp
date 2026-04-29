---
phase: 12-central-enrichment-cutover-validation
plan: "02"
subsystem: skill
tags: [aos8, central, migration, cutover, live-mode, markdown-only]
dependency_graph:
  requires: [12-01]
  provides: [CUTOVER-01, CUTOVER-02, CUTOVER-03]
  affects: [hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md]
tech_stack:
  added: []
  patterns: [Stage-5 live-mode sub-path, fresh show-version call (Pitfall 2 mitigation), Batch-2/Batch-3 reuse, partial-failure paste fallback]
key_files:
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md
decisions:
  - D-09 honored — sub-path inserted BEFORE Phase 0-8 cutover table; existing table untouched
  - D-10 honored — single sub-path covers CUTOVER-01..03
  - D-11 honored — CUTOVER-01 REGRESSION on any cluster state other than L2-connected, citing aos8_get_cluster_state() Batch 3
  - D-12 honored — CUTOVER-02 makes a FRESH aos8_show_command(command='show version') call (NOT Batch 3 reuse); REGRESSION below 8.10.0.12 / 8.12.0.1, VSG §1643-§1649
  - D-13 honored — no tools: frontmatter edit (aos8_show_command already declared at line 23)
  - D-14 honored — CUTOVER-03 INFO baseline from Batch 2 aos8_get_ap_database()
  - Pitfall 2 mitigated — prose explicitly forbids pulling firmware from Batch 3 / show inventory
metrics:
  duration_minutes: 4
  completed_date: "2026-04-29"
  tasks_completed: 1
  tasks_total: 1
  files_modified: 1
---

# Phase 12 Plan 02: Stage 5 AOS8 live-mode cutover prerequisites sub-path Summary

**One-liner:** Inserted an AOS8 live-mode Stage 5 sub-path (CUTOVER-01..03) before the existing Phase 0–8 cutover table in `aos-migration-readiness.md`, automating cluster L2-connected health, controller firmware floor (fresh `show version` call vs 8.10.0.12 / 8.12.0.1), and pre-cutover AP-count baseline — eliminating operator-paste prerequisites for the live AOS8 source path.

## What Was Done

This plan is **markdown-only** — no Python code, no new tools, no `tools:` frontmatter changes. One localized insert into Stage 5 of `aos-migration-readiness.md`.

### Edit — Stage 5 AOS8 live-mode sub-path (CUTOVER-01..03)

Inserted a 36-line block immediately after the Stage 5 heading and intro paragraph, and immediately before the `| Phase | What VSG specifies | Audit check |` Phase 0-8 cutover table header. Block contents:

- **`#### AOS8 live-mode sub-path — cutover prerequisites (CUTOVER-01..03, used when Stage -1 announced "AOS8 API mode")`** — container heading at four hashes, matching Stage 3 / Stage 4 sub-path style.
- **`##### CUTOVER-01 — Live cluster health (REGRESSION on anything other than L2-connected)`** — Reads cluster state from Batch 3 `aos8_get_cluster_state()`; REGRESSION bullet template on non-L2-connected; INFO PASS confirmation template on L2-connected; paste fallback to `show lc-cluster group-membership` if Batch 3 unavailable.
- **`##### CUTOVER-02 — Controller firmware floor (REGRESSION below 8.10.0.12 / 8.12.0.1, VSG §1643-§1649)`** — Explicit "**fresh** call to `aos8_show_command(command='show version')`" with explicit "do NOT pull firmware from Batch 3 / `show inventory`" warning (Pitfall 2 mitigation). Compares against 8.10.0.12 / 8.12.0.1 floor; REGRESSION template + INFO PASS template; paste fallback to pasted `show version` if call fails.
- **`##### CUTOVER-03 — Pre-cutover AP-count baseline (INFO)`** — INFO baseline from Batch 2 `aos8_get_ap_database()` count (`X` APs); paste fallback to `show ap database long`.
- Closing transition: "After this sub-path, continue with the Phase 0–8 cutover table below — those steps apply regardless of source path."

### Inserted Line Range

- Sub-path container heading at line **448**
- New block ends at line **484** (closing transition line)
- Existing Phase 0-8 table header at line **485**, Phase 0 row at line **486**, Phase 8 row at line **494** — all preserved verbatim
- Stage 6 heading at line **496** — unchanged
- Sub-path heading line (448) < Phase 0 row (486) ✓

### Tool Names Referenced (all already in `tools:` frontmatter line 23 — no frontmatter edit)

- `aos8_get_cluster_state` (Batch 3 reuse)
- `aos8_show_command` (fresh `show version` call — D-12)
- `aos8_get_ap_database` (Batch 2 reuse)

### Severity Decisions Applied

- **CUTOVER-01:** REGRESSION on any state other than L2-connected (D-11)
- **CUTOVER-02:** REGRESSION below 8.10.0.12 / 8.12.0.1 floor with VSG §1643-§1649 cite (D-12)
- **CUTOVER-03:** INFO pre-cutover AP baseline (D-14)

### Pitfall 2 (RESEARCH.md) — controller firmware MUST be from fresh `show version`

The CUTOVER-02 prose contains an explicit guard: "Make a **fresh** call to `aos8_show_command(command='show version')` here in Stage 5 — do NOT pull firmware from Batch 3 / `show inventory`, which does not reliably include the running OS firmware string." The string `aos8_show_command(command='show version')` appears 4 times in the new block (heading mention, fresh-call instruction, REGRESSION finding template, INFO PASS template, fallback). The substring `Batch 3` appears only inside the CUTOVER-01 cluster-health prose, never near "firmware".

## Test Results

- **Skill tool-reference regression test:** `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -q` → **8 passed in 2.07s**
- **Full unit suite:** `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` → **790 passed in 29.14s**

## Acceptance Criteria Verification

| Criterion | Result |
|---|---|
| `AOS8 live-mode sub-path — cutover prerequisites` appears exactly once | ✓ |
| `CUTOVER-01 — Live cluster health` appears exactly once | ✓ |
| `CUTOVER-02 — Controller firmware floor` appears exactly once | ✓ |
| `CUTOVER-03 — Pre-cutover AP-count baseline` appears exactly once | ✓ |
| `aos8_show_command(command='show version')` literal appears ≥ 2 times | ✓ (4 occurrences) |
| `8.10.0.12`, `8.12.0.1`, `VSG §1643-§1649` all present in sub-path | ✓ |
| `Pre-cutover AP baseline:` present | ✓ |
| `Cluster not L2-connected:` present | ✓ |
| Sub-path heading line < Phase 0 row line | ✓ (448 < 486) |
| `tools:` frontmatter line 23 unchanged | ✓ |
| Stage 6, Decision matrix, Output formatting untouched | ✓ |
| CUTOVER-02 prose does NOT pull firmware from Batch 3 | ✓ (explicit warning included) |
| Skill regression pytest exits 0 | ✓ |
| Full unit suite green | ✓ (790/790) |

## Deviations from Plan

None — plan executed exactly as written. The verbatim block from the plan's `<action>` section was inserted between the Stage 5 intro paragraph and the Phase 0-8 cutover table header.

## Deferred Manual Verification (Scenario A)

Live AOS8 + Central paired-environment end-to-end execution remains deferred per Phase 10 / 11 / 12-01 precedent — no live AOS8/Mobility Conductor environment is available (CLAUDE.md constraint). Mechanical correctness verified by:

- Block placement above Phase 0-8 table inside Stage 5 boundary (lines 444 / 448 / 485 / 496 confirm scope)
- All three AOS8 tools referenced are present in `tools:` frontmatter line 23 (no frontmatter edit)
- Severity locks honored (D-11 REGRESSION, D-12 REGRESSION + VSG cite, D-14 INFO)
- D-12 fresh-call constraint enforced by explicit "**fresh** call" + "do NOT pull firmware from Batch 3" prose (Pitfall 2 mitigation)
- pytest skill-reference test green (catches any tool-name typo or unauthorized frontmatter drift)
- Full unit suite (790/790) green — no regression in any other platform module

## Commits

| Task | Name | Commit (hpe-networking-mcp repo) | Files |
|------|------|--------|-------|
| 1 | Stage 5 AOS8 live-mode cutover prerequisites sub-path | 91d208c | aos-migration-readiness.md (+36 lines) |

## Self-Check: PASSED

- [x] aos-migration-readiness.md modified and committed (hpe-networking-mcp@91d208c)
- [x] Sub-path container heading present once at line 448
- [x] CUTOVER-01..03 sections all present, one each (lines 454, 464, 474)
- [x] Sub-path block ends before Phase 0-8 table header at line 485
- [x] Phase 0-8 cutover table preserved unchanged (Phase 0 at 486, Phase 8 at 494)
- [x] `tools:` frontmatter line 23 unchanged
- [x] Stage 6 heading at line 496 unchanged; Decision matrix and Output formatting unchanged
- [x] CUTOVER-02 uses fresh `aos8_show_command(command='show version')` (4 occurrences in block); Pitfall 2 mitigation prose included
- [x] Skill regression test passes (8/8)
- [x] Full unit suite passes (790/790)
