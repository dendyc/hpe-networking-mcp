---
phase: 12-central-enrichment-cutover-validation
plan: "01"
subsystem: skill
tags: [aos8, central, migration, enrichment, live-mode, markdown-only]
dependency_graph:
  requires: [11-01]
  provides: [ENRICH-01, ENRICH-02, ENRICH-03, ENRICH-04]
  affects: [hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md]
tech_stack:
  added: []
  patterns: [Stage-4 live-mode sub-path, per-conflict REGRESSION enumeration, fleet-wide central_recommend_firmware single-call, partial-failure fallback to A-row paste path]
key_files:
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md
decisions:
  - D-01 honored — sub-path inserted ABOVE A1-A13 table; existing table untouched
  - D-02 honored — sub-path covers ENRICH-01..04 in a single block
  - D-03 honored — prose explicitly notes "no re-fetching"; cites Batch 1 / Batch 2 / Batch 4 already in context
  - D-04 honored — ENRICH-01 INFO finding template "Source AP count: X. Central onboarded: Y. Gap: Z not yet onboarded."
  - D-05 honored — `central_recommend_firmware()` invoked exactly once with no filter; explicit "do NOT iterate per model"
  - D-06 honored — three conflict surfaces (SSID, role, named-VLAN) cross-referenced against Batch 4 / Batch 1
  - D-07 honored — naming conflicts emitted as REGRESSION (not INFO); 21 REGRESSION occurrences in new block
  - D-08 honored — "one finding per conflicting item"; explicit instruction "do not collapse multiple conflicts into a single bullet"
metrics:
  duration_minutes: 5
  completed_date: "2026-04-29"
  tasks_completed: 1
  tasks_total: 1
  files_modified: 1
---

# Phase 12 Plan 01: Stage 4 AOS8 live-mode Central enrichment sub-path Summary

**One-liner:** Layered an AOS8 live-mode Central enrichment sub-path (ENRICH-01..04) into Stage 4 of `aos-migration-readiness.md`, surfacing AP count gap, per-model AOS10 firmware recommendations, and SSID / role / named-VLAN naming conflicts as REGRESSION findings without operator paste.

## What Was Done

This plan is **markdown-only** — no Python code, no new tools, no `tools:` frontmatter changes. One localized insert into Stage 4 of `aos-migration-readiness.md`.

### Edit — Stage 4 AOS8 live-mode sub-path (ENRICH-01..04)

Inserted a 64-line block immediately after the Stage 4 intro paragraph and immediately before the `| # | Check | Tool | Severity if failing |` A-table header. Block contents:

- **`#### AOS8 live-mode sub-path — Central enrichment (ENRICH-01..04, used when Stage -1 announced "AOS8 API mode")`** — container heading at four hashes, matching Stage 3 sub-path style.
- **`##### ENRICH-01 — AP count gap (INFO)`** — Batch 2 `aos8_get_ap_database()` total `X` vs `central_get_aps()` total `Y`; emit `INFO — Source AP count: X. Central onboarded: Y. Gap: Z not yet onboarded.` Fallback to A4 if `central_get_aps()` fails; paste fallback to `show ap database long` if Batch 2 unavailable.
- **`##### ENRICH-02 — Per-model AOS10 firmware recommendation (INFO)`** — single fleet-wide `central_recommend_firmware()` call with no filter; cross-reference distinct AP `model` values from Batch 2; emit two-column markdown table inline; explicit note "do NOT iterate per model" (D-05). Restricted to AP models — controller firmware deferred to CUTOVER-02 in Stage 5. Fallback to A13 if call fails; paste fallback if Batch 2 unavailable.
- **`##### ENRICH-03 — SSID conflict detection (REGRESSION, one finding per conflict)`** — Batch 4 `aos8_get_bss_table()` SSIDs vs `central_get_wlan_profiles()` (no `ssid` filter); one REGRESSION bullet per conflicting SSID, single INFO bullet for clean case. Fallback to A7 if call fails; paste fallback to `show ap essid` if Batch 4 unavailable.
- **`##### ENRICH-04 — Role and VLAN conflict detection (REGRESSION, one finding per conflict)`** — Batch 1 `aos8_get_effective_config()` user-role names + named VLAN IDs vs `central_get_roles()` + `central_get_named_vlans()` (no filters); one REGRESSION bullet per conflicting role and per conflicting VLAN ID, INFO bullet on clean case. Fallback to A8 / A9 individually; paste fallback to `show configuration effective detail` if Batch 1 unavailable.
- Closing line: "After this sub-path, continue with the A1–A13 table below for any checks not superseded by ENRICH-01..04."

### Inserted Line Range

- Sub-path container heading at line **364**
- New block ends at line **427** (closing transition line)
- Existing A-table header at line **428**, A1 row at line 430, A13 row at line 442 — all preserved verbatim

### Tool Names Introduced into Stage 4 (all already in `tools:` frontmatter line 23 — no frontmatter edit)

- `aos8_get_ap_database` (Batch 2)
- `aos8_get_bss_table` (Batch 4)
- `aos8_get_effective_config` (Batch 1)
- `central_get_aps`
- `central_recommend_firmware`
- `central_get_wlan_profiles`
- `central_get_roles`
- `central_get_named_vlans`

### Severity Decisions Applied

- **ENRICH-01:** INFO (D-04) — gap is migration-scope info, not a blocker.
- **ENRICH-02:** INFO (D-05 implicit) — per-model firmware recommendation is operational reference.
- **ENRICH-03:** REGRESSION per conflict (D-07 + D-08) — upgrades from A7's INFO baseline.
- **ENRICH-04:** REGRESSION per conflict (D-07 + D-08) — upgrades from A8/A9's INFO baseline.

The 21 occurrences of `**REGRESSION** —` inside the new block reflect the conflict-finding templates plus the `**REGRESSION**` strings used in fallback prose narration. The existing A1-A13 table severities are unchanged.

## Test Results

- **Skill tool-reference regression test:** `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -q` → **8 passed in 2.07s**
- **Full unit suite:** `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` → **790 passed in 25.37s**

## Acceptance Criteria Verification

| Criterion | Result |
|---|---|
| Sub-path heading appears exactly once | ✓ (1 occurrence) |
| ENRICH-01..04 headers each appear once | ✓ (1/1/1/1) |
| Sub-path appears BEFORE the A-table header in Stage 4 | ✓ (364 < 428) |
| `central_recommend_firmware()` (no-arg) literal substring present | ✓ |
| `central_recommend_firmware(serial_number=` not present | ✓ |
| `for each model, call central_recommend_firmware` not present | ✓ |
| `central_get_wlan_profiles()` / `central_get_roles()` / `central_get_named_vlans()` present | ✓ |
| `**REGRESSION** —` inside new block (≥3) | ✓ (21 occurrences) |
| `**INFO** — Source AP count:` template present | ✓ |
| 13 A-rows preserved | ✓ (A1..A13) |
| Line 23 `tools:` frontmatter unchanged | ✓ |
| Skill regression pytest exits 0 | ✓ |
| Full unit suite green | ✓ (790/790) |

## Deviations from Plan

None — plan executed exactly as written. The verbatim block from the plan's `<action>` section was inserted; only the surrounding anchor (the existing intro paragraph + A-table header) was used as the Edit anchor, no other prose was touched.

## Deferred Manual Verification (Scenario A)

Live AOS8 + Central paired-environment end-to-end execution remains deferred per Phase 10 / 11 precedent — no live AOS8/Mobility Conductor environment is available (CLAUDE.md constraint). Mechanical correctness verified by:

- Block placement above A-table inside Stage 4 boundary (lines 360 / 364 / 428 / 444 confirm scope)
- All eight tool-name references already present in `tools:` frontmatter line 23 (no frontmatter edit)
- Severity locks honored (D-04, D-07) — REGRESSION on every conflict-finding template, INFO on enrichment-summary templates
- D-05 single-call constraint enforced by explicit "do NOT iterate per model" prose
- D-08 one-finding-per-conflict constraint enforced by explicit "do not collapse multiple conflicts into a single bullet" prose
- pytest skill-reference test green (catches any tool-name typo or unauthorized frontmatter drift)
- Full unit suite (790/790) green — no regression in any other platform module

## Commits

| Task | Name | Commit (hpe-networking-mcp repo) | Files |
|------|------|--------|-------|
| 1 | Stage 4 AOS8 live-mode sub-path (ENRICH-01..04) | f89b9f5 | aos-migration-readiness.md (+64 lines) |

## Self-Check: PASSED

- [x] aos-migration-readiness.md modified and committed (hpe-networking-mcp@f89b9f5)
- [x] Sub-path heading present once at line 364
- [x] ENRICH-01..04 sections all present, one each
- [x] Sub-path block ends before A-table header at line 428
- [x] A1..A13 (13 rows) all present, unchanged
- [x] `tools:` frontmatter line 23 unchanged
- [x] Skill regression test passes (8/8)
- [x] Full unit suite passes (790/790)
- [x] No frontmatter edit, no Stage 5 / Stage 6 / Decision matrix touched
