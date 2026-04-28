# Phase 4: Differentiator Tools - Context

**Gathered:** 2026-04-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Implement 9 AOS8-specific differentiator read tools (`DIFF-01..DIFF-09`) — Conductor hierarchy, effective config, pending changes, RF neighbors, cluster state, air monitors, AP wired ports, IPsec tunnels, and a unified MD health check. These expose capabilities unique to AOS8 that have no Aruba Central equivalent.

Deliverables:
- `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` — all 9 DIFF tools
- `tests/unit/test_aos8_read_differentiators.py` — unit tests with mocked API responses
- `platforms/aos8/__init__.py` updated — `TOOLS["differentiators"]` key added, existing 5 keys unchanged

**Not in scope for Phase 4:** Write tools (Phase 5), guided prompts (Phase 6), docs updates (Phase 6).

</domain>

<decisions>
## Implementation Decisions

### File Organization
- **D-01:** All 9 DIFF tools go in a single new file: `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py`. At ~250–300 lines it fits well under the 500-line limit.
- **D-02:** A new `"differentiators"` category key is added to the `TOOLS` dict in `platforms/aos8/__init__.py`. The 9 DIFF tool names live under this key. Existing 5 category keys (health, clients, alerts, wlan, troubleshooting) are not modified.

### DIFF-09: aos8_get_md_health_check
- **D-03:** Implemented as multi-call aggregation — makes 3–4 targeted API calls for the given `config_path` scope and merges the results into one response dict. Mirrors the Central `get_site_health` pattern.
- **D-04:** The four data points to aggregate: AP counts (up/down total), connected client count, active alarm summary (count by severity: critical/major/minor), and AOS8 firmware version on the target MD.
- **D-05:** `config_path` is required for `aos8_get_md_health_check` (no default) — the operator must specify which MD or scope they want the health check for.

### DIFF-02: aos8_get_effective_config
- **D-06:** Tool accepts an `object_name: str` parameter (e.g., `"ssid_prof"`, `"ap_group"`, `"virtual_ap"`) and a `config_path: str` (defaulting to `/md`). Researcher to confirm which AOS8 config objects support inheritance resolution.
- **D-07:** Primary path: config-object endpoint (`/v1/configuration/object/<object_name>?config_path=<path>`) with `config_path` pointing to the specific MD or AP group — AOS8 returns the inherited/resolved value at that node. Fallback: appropriate show command if no clean config-object path exists.

### DIFF-03: aos8_get_pending_changes
- **D-08:** Primary path: config-object or dedicated API endpoint if one exists. Fallback: show command (e.g., `"show pending-config"` or equivalent). Researcher to identify the best available path.

### API Uncertainty Fallback Strategy (DIFF-02, DIFF-03)
- **D-09:** If no direct config-object endpoint exists for a DIFF tool, researcher falls back to the show command API (`/v1/configuration/showcommand`). The show command string that provides the needed data should be documented in the plan so the executor has zero ambiguity.

### Plan Sequencing (TDD)
- **D-10:** 3 plans mirroring Phase 3 discipline, right-sized for 9 tools:
  - **Plan 04-01 (Wave 0):** Write all 9 DIFF tool tests (`test_aos8_read_differentiators.py`) — all red baseline. No implementation.
  - **Plan 04-02 (Implementation):** Implement all 9 DIFF tools in `differentiators.py`. Tests go green.
  - **Plan 04-03 (Wiring):** Update `platforms/aos8/__init__.py` — add `"differentiators"` TOOLS dict entry, importlib loop covers new module automatically.

### Claude's Discretion
- Exact AOS8 REST endpoint paths or show commands for each DIFF tool (researcher to verify)
- JSON fixture file names and structure for test mocks (place in `tests/unit/fixtures/aos8/` per Phase 3 pattern)
- Whether DIFF-05 (cluster state), DIFF-06 (air monitors), DIFF-07 (AP wired ports), DIFF-08 (IPsec tunnels) use show commands or config-object endpoints — researcher to determine
- Whether `config_path` is scope-sensitive for DIFF-04 (RF neighbors), DIFF-06, DIFF-07, DIFF-08 — researcher to confirm

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Tool Implementation Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py` — canonical AOS8 tool file with `@tool(annotations=READ_ONLY)`, Context injection, `run_show()` / `get_object()` usage, and error handling pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py` — reference for tools using show command passthrough and optional `config_path`
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — reference for tools using `get_object()` with `config_path`

### AOS8 Shared Helpers
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` — `run_show()`, `get_object()`, `strip_meta()`, `format_aos8_error()` — all DIFF tool implementations use these; no direct httpx calls in tool functions

### AOS8 Platform Module
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` — `AOS8Client.request()`, `AOS8APIError`, `AOS8AuthError`
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — `TOOLS` dict structure to extend; `register_tools()` import loop pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/_registry.py` — `tool()` shim for decorating DIFF tool functions

### Prior Phase Research
- `.planning/phases/03-read-tools/03-RESEARCH.md` — AOS8 API endpoint patterns, config-object vs showcommand URL shapes, `_global_result` semantics, fixture patterns

### Test Patterns
- `hpe-networking-mcp/tests/unit/test_aos8_read_health.py` — category test file structure with httpx mock, `pytest.mark.unit`, fixture loading from `tests/unit/fixtures/aos8/`
- `hpe-networking-mcp/tests/conftest.py` — shared `secrets_dir` and `loguru_capture` fixtures

### Style & Constraints
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/__init__.py` — `READ_ONLY` annotations constant
- Project constraint: 500 lines max per file, 50 lines max per function, Google-style docstrings

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `run_show(client, command, *, config_path=None)` in `_helpers.py` — covers all DIFF tools that use show commands
- `get_object(client, object_path, *, config_path="/md")` in `_helpers.py` — covers DIFF-02 effective config if a config-object endpoint exists
- `strip_meta()` / `format_aos8_error()` in `_helpers.py` — same response cleanup as Phase 3
- `ctx.lifespan_context["aos8_client"]` — client access pattern shared across all tool functions
- `READ_ONLY` ToolAnnotations from `tools/__init__.py` — same annotation for all DIFF tools

### Established Patterns
- `async def aos8_<name>(ctx: Context, ...) -> dict[str, Any]` — all tools are async, return dicts
- `@tool(name="aos8_<name>", annotations=READ_ONLY)` decorator with explicit string name
- `try/except (AOS8APIError, AOS8AuthError, httpx.HTTPError) as exc` → `{"error": format_aos8_error(exc)}`
- Fixture JSON files in `tests/unit/fixtures/aos8/` (established in Phase 3)
- `pytest.mark.unit` on all test functions; `pytest-asyncio` auto mode

### Integration Points
- `platforms/aos8/__init__.py:TOOLS` — add `"differentiators": [list of 9 DIFF tool names]`
- The importlib loop in `register_tools()` already iterates `TOOLS.keys()` — adding `"differentiators"` key automatically triggers import of `differentiators.py`; no other wiring change needed

### DIFF-09 Aggregation Note
- `aos8_get_active_aps` (READ-03), `aos8_get_clients` (READ-09), `aos8_get_alarms` (READ-13), `aos8_get_version` (READ-07) are already implemented and tested in Phase 3 — DIFF-09 calls the **same API endpoints** those tools use (not the tool functions themselves) to avoid coupling tool implementations

</code_context>

<specifics>
## Specific Ideas

- `aos8_get_md_health_check` response shape should mirror `get_site_health` from Central's cross-platform tool pattern: a top-level dict with named sections (`"aps"`, `"clients"`, `"alarms"`, `"firmware"`) so callers can navigate predictably.
- DIFF-02 (`aos8_get_effective_config`) with `object_name` parameter mirrors how `aos8_show_command` is parameterized — gives operators flexibility without a separate tool per object type.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 04-differentiator-tools*
*Context gathered: 2026-04-28*
