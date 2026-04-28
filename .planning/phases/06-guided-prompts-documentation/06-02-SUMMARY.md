---
phase: 06-guided-prompts-documentation
plan: "02"
subsystem: documentation
tags: [docs, readme, changelog, tools-reference, instructions, versioning]
dependency_graph:
  requires: ["06-01"]
  provides: ["DOCS-01", "DOCS-02", "DOCS-03", "DOCS-04", "DOCS-05"]
  affects: ["hpe-networking-mcp/INSTRUCTIONS.md", "hpe-networking-mcp/README.md", "hpe-networking-mcp/docs/TOOLS.md", "hpe-networking-mcp/CHANGELOG.md", "hpe-networking-mcp/pyproject.toml"]
tech_stack:
  added: []
  patterns: ["Keep-a-Changelog format", "operator-facing vs AI-facing doc distinction", "cross-reference links between doc layers"]
key_files:
  created:
    - hpe-networking-mcp/INSTRUCTIONS.md
  modified:
    - hpe-networking-mcp/README.md
    - hpe-networking-mcp/docs/TOOLS.md
    - hpe-networking-mcp/CHANGELOG.md
    - hpe-networking-mcp/pyproject.toml
decisions:
  - "New INSTRUCTIONS.md at repo root is operator-facing; in-package src/hpe_networking_mcp/INSTRUCTIONS.md (AI-facing) was NOT modified"
  - "TOOLS.md Overview table row: Aruba OS 8 = 26 read + 12 write + 9 prompts (47 total vs 38 underlying tools count used in section heading)"
  - "README capability table updated from 6 to 7 platforms; dynamic mode count updated from 24 to 27 tools"
  - "Pre-existing integration test failure (test_no_visibility_transform_when_write_tools_enabled) confirmed pre-dates this plan; deferred to deferred-items"
metrics:
  duration: "~7 minutes"
  completed_date: "2026-04-28"
  tasks_completed: 3
  files_changed: 5
---

# Phase 6 Plan 2: Documentation Update Summary

Operator-facing documentation for the AOS8 platform addition: new repo-root INSTRUCTIONS.md, README.md capability table + secrets + auto-disable + counts, docs/TOOLS.md AOS8 section (38 tools + 9 prompts), CHANGELOG.md 2.4.0.0 entry, and pyproject.toml version bump to 2.4.0.0.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create repo-root INSTRUCTIONS.md | 80fe5a7 | hpe-networking-mcp/INSTRUCTIONS.md (NEW) |
| 2 | Update README.md and docs/TOOLS.md | 5cd47f7 | hpe-networking-mcp/README.md, hpe-networking-mcp/docs/TOOLS.md |
| 3 | Bump version to 2.4.0.0 and add CHANGELOG entry | f1e2943 | hpe-networking-mcp/pyproject.toml, hpe-networking-mcp/CHANGELOG.md |

## Files Modified / Created

### hpe-networking-mcp/INSTRUCTIONS.md (NEW)
Operator-facing reference document at repo root. Covers:
- Connecting to a Conductor vs. standalone controller (topology table)
- `config_path` semantics with inheritance model
- `aos8_write_memory` contract (explicit two-step flow, never auto-called)
- `aos8_show_command` passthrough with useful invocation table
- Guided Prompt Index: all 9 prompts with parameters and workflow descriptions
- Cross-reference link to `src/hpe_networking_mcp/INSTRUCTIONS.md` (Pitfall 3 mitigation)

### hpe-networking-mcp/README.md (MODIFIED)
- Capability table: added AOS8 column across all 37 capability rows
- Bottom row: `38 + 9 prompts` in AOS8 column
- Guided Prompts row: `✅` for AOS8
- AOS8 Guided Prompts section with all 9 prompt descriptions
- Aruba OS 8 secrets reference section (5 secrets: aos8_host/username/password/port/verify_ssl)
- `ENABLE_AOS8_WRITE_TOOLS` added to write tools table and configuration table
- Auto-disable section updated: "seven platforms", AOS8 listed in examples, startup log example includes AOS8 disabled message
- Architecture ASCII diagram updated to include AOS8 box (7-platform layout)
- Dynamic mode tool count updated from 24 to 27
- Project structure: added `aos8/` entry
- Test count: updated from 639+ to 767+

### hpe-networking-mcp/docs/TOOLS.md (MODIFIED)
- Overview table: added `Aruba OS 8` row (26 read, 12 write, 9 prompts, 47 total)
- Appended `## Aruba OS 8 / Mobility Conductor (38 tools + 9 prompts)` section:
  - Health & Inventory (8 tools)
  - Clients (4 tools)
  - Alerts & Audit (3 tools)
  - WLAN & Config (4 tools)
  - Troubleshooting (7 tools)
  - Writes (12 tools) with `ENABLE_AOS8_WRITE_TOOLS=true` gate note
  - Guided Prompts (9 prompts) table
  - Cross-reference link to `../INSTRUCTIONS.md`

### hpe-networking-mcp/CHANGELOG.md (MODIFIED)
New `## [2.4.0.0] - 2026-04-28` entry inserted above 2.3.0.1:
- ### Added: AOS8 module description (38 tools, 9 prompts, client auth, write contract)
- ### Added: repo-root INSTRUCTIONS.md, docs/TOOLS.md section, README updates
- ### Changed: README counts, version bump rationale
- ### Tests: test count (767+), reference to test_aos8_prompts.py
- Prior entries (2.3.0.1 and earlier) fully preserved

### hpe-networking-mcp/pyproject.toml (MODIFIED)
- `version = "2.3.0.1"` → `version = "2.4.0.0"` (single line change)

## In-Package File Confirmation

`src/hpe_networking_mcp/INSTRUCTIONS.md` was **NOT modified** — confirmed via `git diff --name-only`. This is the AI-facing system prompt and lives at a different layer. The new `INSTRUCTIONS.md` at repo root is a separate, operator-facing companion document (Pitfall 3 honored).

## Tool / Prompt Count Strings Used

| Location | String |
|----------|--------|
| README capability table cell | `38 + 9 prompts` |
| README architecture diagram | `38 tools` / `+9 prmt` |
| README AOS8 Guided Prompts section | 9 prompts listed |
| docs/TOOLS.md section heading | `## Aruba OS 8 / Mobility Conductor (38 tools + 9 prompts)` |
| docs/TOOLS.md Overview table | 26 read, 12 write, 9 prompts, 47 total |
| INSTRUCTIONS.md Overview | "38 tools (8 health, 4 client, 3 alert, 4 WLAN, 7 troubleshooting, 12 write) plus 9 guided prompts" |
| CHANGELOG.md | "38 tools across 6 categories", "9 guided prompts" |

## Test Suite Results

```
767 passed, 11 skipped, 1 failed (pre-existing)
```

- 767 tests passing (matches Phase 6 Plan 01 baseline)
- 1 pre-existing failure: `test_no_visibility_transform_when_write_tools_enabled` in `tests/integration/test_server.py` — confirmed pre-dates this plan (fails identically on the commit before our changes). Deferred to `deferred-items.md`.
- Documentation-only changes have no impact on test outcomes.

## CLAUDE.md Documentation Checklist

- [x] README.md — tool counts, capability table, architecture diagram, auto-disable, project structure
- [x] CHANGELOG.md — 2.4.0.0 entry with full description
- [x] INSTRUCTIONS.md — new operator-facing doc at repo root (AI-facing in-package file untouched)
- [x] docs/TOOLS.md — AOS8 section with all 38 tools and 9 prompts
- [x] pyproject.toml — version bumped to 2.4.0.0

## Phase 6 Status

Phase 6 complete. Both plans (06-01 guided prompts, 06-02 documentation) executed successfully. Proceed to Phase 7: Testing & Integration.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all documentation is complete and accurate. All 38 tool names and 9 prompt names reference live implementations from Phases 3-5 and Phase 6 Plan 01.

## Self-Check: PASSED
