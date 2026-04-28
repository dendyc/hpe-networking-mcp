---
phase: 03-read-tools
plan: 06
subsystem: aos8-troubleshooting
tags: [aos8, read-tools, troubleshooting, show-command]
requires:
  - 03-01 (helpers, registry, READ_ONLY annotations)
  - 02-* (AOS8Client + showcommand wiring)
provides:
  - aos8_ping (READ-20)
  - aos8_traceroute (READ-21)
  - aos8_show_command (READ-22)
  - aos8_get_logs (READ-23)
  - aos8_get_controller_stats (READ-24)
  - aos8_get_arm_history (READ-25)
  - aos8_get_rf_monitor (READ-26)
affects:
  - 03-07 (wiring plan must register the 7 new tools in TOOLS dict)
tech_stack_added: []
patterns_used:
  - run_show + format_aos8_error helper pattern (matches Plans 03-02..03-05)
  - direct client.request for show_command to allow non-JSON text wrapping
  - in-tool input validation (count <= 1000, "show " prefix guard)
key_files_created:
  - hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py
key_files_modified: []
decisions:
  - "ctx typed as Any (matches sibling wlan.py); fastmcp.Context import not required"
  - "aos8_show_command preserves original casing in outgoing params; only the prefix CHECK is case-insensitive"
  - "aos8_ping/traceroute use run_show (config_path=None) — params dict matches test contract exactly"
metrics:
  duration: ~7 min
  completed: "2026-04-28T13:49:40Z"
---

# Phase 3 Plan 06: AOS8 Troubleshooting / Diagnostics Read Tools Summary

Implemented READ-20..26 in `aos8/tools/troubleshooting.py` — 7 read tools covering ping, traceroute, the strict `show` passthrough, system-log retrieval, controller stat aggregation, ARM history, and RF monitor — all green against the Wave 0 contract tests.

## What Was Built

A single new module `troubleshooting.py` (211 lines) exposing seven `@tool`-decorated coroutines:

| Tool                        | REQ      | Endpoint shape                          | Notes |
|-----------------------------|----------|------------------------------------------|-------|
| `aos8_ping`                 | READ-20  | `showcommand?command=ping <dest>`        | bypasses prefix guard (builds command internally) |
| `aos8_traceroute`           | READ-21  | `showcommand?command=traceroute <dest>`  | bypasses prefix guard |
| `aos8_show_command`         | READ-22  | `showcommand?command=<verbatim>`         | strict prefix guard, text-response wrapping |
| `aos8_get_logs`             | READ-23  | `showcommand?command=show log system N`  | 1000-line cap |
| `aos8_get_controller_stats` | READ-24  | 3 sequential `showcommand` calls         | aggregates cpu/memory/uptime |
| `aos8_get_arm_history`      | READ-25  | `showcommand?command=show ap arm history` + config_path | default `/md` |
| `aos8_get_rf_monitor`       | READ-26  | `showcommand?command=show ap monitor stats` + config_path | default `/md` |

## Key Implementation Details

- `aos8_show_command` normalizes input via `command.strip()`, then matches case-insensitively against `"show "` to admit both `"show version"` and `"  SHOW version"`. Crucially, the **outgoing** `params["command"]` keeps original casing — only the gate is case-insensitive — which matches the test contract (`test_show_command_case_insensitive` only asserts `assert_awaited_once`, not exact case).
- Non-JSON responses (e.g. `show running-config` text dumps) are detected by catching `ValueError` from `response.json()` and wrapped as `{"output": response.text}`.
- `aos8_get_controller_stats` wraps all three sequential `run_show` calls in a single `try/except` so any one failure produces a single, coherent error string rather than a partial dict.
- `aos8_get_logs` validates `count` BEFORE constructing the client (per the test, `client.request.assert_not_awaited()`).
- `aos8_ping` and `aos8_traceroute` use `run_show(client, command)` — with `config_path=None`, the helper omits the `config_path` query param, exactly matching the test contract `params={"command": "ping 8.8.8.8"}`.

## Verification

- `pytest tests/unit/test_aos8_read_troubleshooting.py -x -q` → **15 passed in 0.51s**
- `ruff check` → **All checks passed**
- `ruff format --check` → **1 file already formatted**
- `mypy` → **Success: no issues found in 1 source file**
- `wc -l troubleshooting.py` → **211** (well under 500-line cap)
- `grep -c '@tool(name="aos8_'` → **7**
- `grep -n 'lower().startswith("show ")'` → **line 102**

Full unit suite: 690 passing. The 3 pre-existing failures in `test_aos8_init.py` are red baseline tests reserved for plan 03-07 (the TOOLS-dict wiring plan); they expect `register_tools` to return `26`, which only becomes true after 03-07 lands.

## Deviations from Plan

**None — plan executed exactly as written.**

The plan suggested typing `ctx` as `fastmcp.Context`, but the existing sibling module `wlan.py` (plan 03-05) types it as `Any` and omits the `fastmcp` import. I followed the sibling convention to keep the package internally consistent. This is a stylistic alignment, not a deviation from any test contract or behavior.

## Commits

- `9f8a627` — feat(03-06): implement AOS8 troubleshooting/diagnostics read tools (READ-20..26)

## Self-Check

- [x] `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py` exists (211 lines)
- [x] Commit `9f8a627` present in `hpe-networking-mcp` repo
- [x] All 15 plan-scope tests pass
- [x] Lint + format + type check clean

## Self-Check: PASSED
