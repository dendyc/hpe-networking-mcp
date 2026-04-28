# Phase 3: Read Tools - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-28
**Phase:** 03-read-tools
**Areas discussed:** File layout, config_path scope, show_command design, Plan sequencing

---

## File Layout

| Option | Description | Selected |
|--------|-------------|----------|
| Category per file | 5 files: health.py, clients.py, alerts.py, wlan.py, troubleshooting.py — mirrors Central's layout | ✓ |
| Fewer, broader files | 3 files: devices_clients.py, alerts_wlan.py, diagnostics.py — fewer imports but mixed concern | |
| Single tools.py | One file for everything — violates 500-line rule with 26 tools | |

**User's choice:** Category per file (recommended)
**Notes:** Mirrors Central's layout; clean separation of concern; each file stays comfortably under 500 lines.

---

## config_path Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Selective — scope-sensitive only | Only AP/WLAN/client tools accept config_path; version/license/controller-inventory tools do not | ✓ |
| Universal — every tool | All 26 tools accept config_path; most health/version tools would ignore it | |
| None — derive from secrets | config_path fixed at /md for all tools; per-MD scope only via show_command | |

**User's choice:** Selective — scope-sensitive only (recommended)
**Notes:** Health/version tools always query Conductor root; adding config_path to them would add schema noise with no benefit.

---

## show_command Design

| Option | Description | Selected |
|--------|-------------|----------|
| Open passthrough — show only | Any command accepted if it starts with "show " (case-insensitive); _meta stripped | ✓ |
| Allowlist of safe commands | Only predefined known-safe show commands accepted | |
| Fully open passthrough | Any command string, no prefix check | |

**User's choice:** Open passthrough — show only (recommended)
**Notes:** `show ` prefix check blocks config/write commands while keeping full AI troubleshooting flexibility. Returns structured JSON with `_meta` stripped.

---

## Plan Sequencing

| Option | Description | Selected |
|--------|-------------|----------|
| Wave 0 scaffold then implement | Plan 1: all failing tests. Plans 2–6: one category per plan. CI green at every step. | ✓ |
| Per-category waves with inline tests | Each plan: implementation + tests for one category | |
| Implement first, test last | Plans 1–5 implement, Plan 6 adds all tests. CI fails on each implementation plan. | |

**User's choice:** Wave 0 scaffold then implement (recommended)
**Notes:** Mirrors Phase 2 approach — establishes full test contract before any implementation lands.

---

## Claude's Discretion

- Exact AOS8 REST endpoint paths per tool (researcher to verify)
- JSON fixture file names/structure for test mocks
- Which of READ-23..26 tools need config_path (researcher to confirm)
- Whether `aos8_get_client_history` uses a show command or config object endpoint

## Deferred Ideas

None — discussion stayed within phase scope.
