# Phase 3: Read Tools - Context

**Gathered:** 2026-04-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Implement 26 read-only AOS8 tools (`READ-01..READ-26`) across five functional categories and wire them into the `platforms/aos8/` module. Deliverables:

- `src/hpe_networking_mcp/platforms/aos8/tools/health.py` — controllers, APs, version, licenses (READ-01..08)
- `src/hpe_networking_mcp/platforms/aos8/tools/clients.py` — client list, find, detail, history (READ-09..12)
- `src/hpe_networking_mcp/platforms/aos8/tools/alerts.py` — alarms, audit trail, events (READ-13..15)
- `src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — SSID profiles, virtual APs, AP groups, user roles (READ-16..19)
- `src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py` — ping, traceroute, show_command passthrough, logs, controller stats, ARM history, RF monitor (READ-20..26)
- `tests/unit/test_aos8_read_*.py` — category-aligned test files
- `platforms/aos8/__init__.py` updated — TOOLS dict populated, `build_meta_tools` called

**Not in scope for Phase 3:** DIFF tools (Phase 4), write tools (Phase 5), guided prompts (Phase 6), docs updates (Phase 6).

</domain>

<decisions>
## Implementation Decisions

### File Organization
- **D-01:** 5 category files, one per functional area: `health.py`, `clients.py`, `alerts.py`, `wlan.py`, `troubleshooting.py`. Mirrors Central's per-category layout. Each file stays under 500 lines.
- **D-02:** File placement: `src/hpe_networking_mcp/platforms/aos8/tools/<category>.py`. Import via the existing `_registry.tool()` shim pattern (same as all other platforms).

### config_path Parameter
- **D-03:** `config_path` is **selective** — only tools that are genuinely scope-sensitive accept it. Default `/md` for those that do.
- **D-04:** Tools that accept `config_path` (scope-sensitive):
  - All `wlan.py` tools (SSID profiles, virtual APs, AP groups, user roles — config lives under a hierarchy node)
  - `aos8_get_ap_database` (READ-02), `aos8_get_active_aps` (READ-03), `aos8_get_ap_detail` (READ-04), `aos8_get_bss_table` (READ-05), `aos8_get_radio_summary` (READ-06) — AP state is per-MD/AP-group scope
  - `aos8_get_clients` (READ-09), `aos8_find_client` (READ-10), `aos8_get_client_detail` (READ-11) — client visibility is scope-dependent
  - `aos8_get_alarms` (READ-13), `aos8_get_events` (READ-15) — alerting has config_path scope
- **D-05:** Tools that do NOT accept `config_path` (Conductor-root queries):
  - `aos8_get_controllers` (READ-01) — always queries the full Conductor inventory
  - `aos8_get_version` (READ-07), `aos8_get_licenses` (READ-08) — Conductor-level, no per-MD scope
  - `aos8_get_client_history` (READ-12) — client-specific lookup, not scope-filtered
  - `aos8_get_audit_trail` (READ-14) — Conductor-wide audit log
  - `aos8_ping` (READ-20), `aos8_traceroute` (READ-21) — target-IP operations, no config scope
  - `aos8_get_logs` (READ-23), `aos8_get_controller_stats` (READ-24), `aos8_get_arm_history` (READ-25), `aos8_get_rf_monitor` (READ-26) — researcher to confirm appropriate scope; default to Conductor-wide unless API requires config_path
  - `aos8_show_command` (READ-22) — passthrough; command string encodes any needed scope

### show_command Passthrough Design (READ-22)
- **D-06:** `aos8_show_command` accepts any `command` string but enforces it starts with `"show "` (case-insensitive). Any command not matching this prefix is rejected with a clear error message — prevents accidental config/write commands.
- **D-07:** `_meta` field is stripped from the response before returning. Where the response is structured JSON, it is returned as-is (minus `_meta`). Where it is a plain string, return it wrapped in `{"output": "..."}`.
- **D-08:** No allowlist — the `show ` prefix check is the only guard. This keeps the tool maximally useful for AI troubleshooting workflows while blocking obvious misuse.

### Plan Sequencing / TDD
- **D-09:** Wave 0 scaffold first: Plan 1 writes all failing tests across all 26 tools (5 test files, one per category) before any implementation. This mirrors Phase 2's approach and ensures CI stays green at every subsequent step.
- **D-10:** Plans 2–6 implement one category each: health, clients, alerts, wlan, troubleshooting. Each plan also registers the category in `TOOLS` and ensures `build_meta_tools` is wired by the final plan.

### Claude's Discretion
- Exact AOS8 REST endpoint paths per tool (researcher to verify against AOS8 API docs and Phase 2 RESEARCH.md)
- JSON fixture file names and structure for test mocks
- Which `aos8_get_logs`/`aos8_get_controller_stats`/`aos8_get_arm_history`/`aos8_get_rf_monitor` tools need `config_path` — researcher to confirm based on actual API endpoints
- Whether `aos8_get_client_history` uses a show command or a config object endpoint

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Tool Implementation Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/clients.py` — canonical example of how a category tool file is structured: imports, `@tool(annotations=READ_ONLY)`, Context injection, lifespan_context client access
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/devices.py` — reference for health/inventory tool patterns
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/troubleshooting.py` — reference for ping/traceroute/show-command style tools
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/alerts.py` — reference for alarms/events tool structure

### AOS8 Platform Module (Phase 2 outputs)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` — `AOS8Client.request(method, path, params=, json_body=)` is the only client interface tools should use; `AOS8APIError`, `AOS8AuthError` are the exception types
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — `TOOLS` dict and `register_tools()` stub to be replaced in Phase 3
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/_registry.py` — `tool()` shim for decorating AOS8 tools

### Dynamic Mode Meta-Tools
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/_common/meta_tools.py` — `build_meta_tools(platform, mcp)` call needed in `register_tools()` for dynamic mode

### Test Patterns
- `hpe-networking-mcp/tests/unit/test_aos8_client.py` — existing AOS8 test file with mocked httpx patterns and `loguru_capture` fixture usage
- `hpe-networking-mcp/tests/conftest.py` — shared fixtures (`secrets_dir`, `loguru_capture`)

### AOS8 API Reference (from Phase 2 Research)
- `hpe-networking-mcp/.planning/phases/02-api-client/02-RESEARCH.md` — AOS8 API endpoint patterns, show-command URL structure, `_global_result` response semantics, config object paths

### Style & Constraints
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/__init__.py` — `READ_ONLY` annotations constant pattern
- Project constraint: 500 lines max per file, 50 lines max per function, Google-style docstrings

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `AOS8Client.request("GET"|"POST", path, params=, json_body=)` in `client.py` — the single call interface for all tool HTTP operations
- `_registry.tool()` shim in `aos8/_registry.py` — decorates tool functions for registration (same pattern as all other platforms)
- `ctx.lifespan_context["aos8_client"]` — how tools access the client at runtime
- `ctx.lifespan_context["aos8_config"]` — how tools access `AOS8Secrets` if needed (e.g., for config_path default)
- `loguru_capture` fixture in `conftest.py` — captures log output for token-leak assertions in tests
- `responses` library (already in deps) — used for mocking in existing tests; tool tests should use `pytest-asyncio` + `unittest.mock` or `responses` per existing patterns

### Established Patterns
- Tool function decorated with `@tool(annotations=READ_ONLY)` using `aos8_read` tag
- `async def aos8_<name>(ctx: Context, ...) -> dict[str, Any]` — all tool functions are async, return dicts
- Error handling: `try/except (AOS8APIError, AOS8AuthError, httpx.HTTPError) as exc` → return structured error dict (align with how Central tools handle network errors)
- `config_path: str = "/md"` as a keyword-only parameter with explicit default on tools that accept it
- Tool name registration: `@tool(name="aos8_get_controllers")` — explicit string name in decorator

### Integration Points
- `platforms/aos8/__init__.py:TOOLS` dict — must be populated with tool names by the end of Phase 3
- `platforms/aos8/__init__.py:register_tools()` — must call `build_meta_tools("aos8", mcp)` for dynamic mode
- `platforms/aos8/tools/__init__.py` — needs a `READ_ONLY` annotations constant (or import from `_common`)

</code_context>

<specifics>
## Specific Ideas

- `aos8_show_command` should return `{"output": <structured_json_or_string>}` with `_meta` stripped — not just the raw response. This makes it consistent with other tool return shapes.
- The `show ` prefix enforcement in `aos8_show_command` should return a clear error: `{"error": "Only 'show' commands are permitted. Received: '<command>'"}` — not a generic exception.
- Test fixture JSON files should live in `tests/unit/fixtures/aos8/` — a new directory following the pattern that other platforms may use.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 03-read-tools*
*Context gathered: 2026-04-28*
