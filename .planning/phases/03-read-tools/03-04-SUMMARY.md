---
phase: 03-read-tools
plan: 04
subsystem: aos8-read-tools
tags: [aos8, read-tools, alerts, events, audit-trail]
requires:
  - 03-01  # Wave 0 scaffold (_helpers.run_show, READ_ONLY, _registry.tool)
provides:
  - aos8_get_alarms
  - aos8_get_audit_trail
  - aos8_get_events
affects:
  - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/alerts.py
tech-stack:
  added: []
  patterns:
    - "Mirror Plan 03-05 wlan.py structure with run_show instead of get_object"
    - "Per-tool try/except wrapping run_show + format_aos8_error verb"
key-files:
  created:
    - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/alerts.py
  modified: []
decisions:
  - "audit-trail tool intentionally omits config_path per CONTEXT D-04/D-05 — show audit-trail is controller-wide"
  - "alarms verb: 'list alarms' (multiple, enumerable); events verb: 'fetch events'; audit verb: 'fetch audit trail'"
metrics:
  duration: ~5 min
  completed: 2026-04-28
  tasks: 1
  files: 1
  loc: 91
requirements: [READ-13, READ-14, READ-15]
---

# Phase 3 Plan 04: AOS8 Alerts/Events Read Tools Summary

Implemented READ-13..15 — three AOS8 read tools wrapping `show alarms`, `show audit-trail`, and `show events` showcommand calls — flipping `test_aos8_read_alerts.py` from red to green.

## What Was Built

A single new file `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/alerts.py` (91 lines) containing three async tool functions, each decorated `@tool(name="aos8_<x>", annotations=READ_ONLY)`, all returning `dict[str, Any] | str`:

| Tool | Show Command | config_path | Error Verb |
|------|--------------|-------------|------------|
| `aos8_get_alarms` | `show alarms` | optional, default `/md` | "list alarms" |
| `aos8_get_audit_trail` | `show audit-trail` | NOT accepted | "fetch audit trail" |
| `aos8_get_events` | `show events` | optional, default `/md` | "fetch events" |

All three route through the shared `_helpers.run_show()` and wrap exceptions in `format_aos8_error()`. Module structure mirrors Plan 03-05's `wlan.py` exactly (which uses `get_object` instead of `run_show`).

## Verification

- `uv run pytest tests/unit/test_aos8_read_alerts.py -x -q` → **3 passed in 0.28s**
- `uv run ruff check src/hpe_networking_mcp/platforms/aos8/tools/alerts.py` → **All checks passed**
- `uv run mypy src/hpe_networking_mcp/platforms/aos8/tools/alerts.py` → **Success: no issues**
- Full suite: 690 passed; the 3 failures in `test_aos8_init.py` are out-of-scope and expected — they will flip green when Plan 03-07 populates the `TOOLS` dict.

## Deviations from Plan

None — plan executed exactly as written.

## Commits

- `ee1295f` — feat(03-04): implement aos8 alerts/events/audit-trail read tools (READ-13..15)

## Self-Check: PASSED

- alerts.py exists at expected path
- 3 `@tool(name="aos8_` decorators present (verified during review)
- Commit `ee1295f` present in main branch of `hpe-networking-mcp/` repo
- Ruff + mypy clean; targeted test file fully green
