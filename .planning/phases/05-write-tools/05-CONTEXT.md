# Phase 5: Write Tools - Context

**Gathered:** 2026-04-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Implement 12 gated AOS8 write tools (`WRITE-01..WRITE-12`) — 9 configuration management tools (SSID profile, virtual AP, AP group, user role, VLAN, AAA server, AAA server group, ACL, netdestination) plus 2 operational actions (client disconnect, AP reboot) and explicit write_memory. All tools are gated behind `ENABLE_AOS8_WRITE_TOOLS=true`.

Deliverables:
- `src/hpe_networking_mcp/platforms/aos8/tools/writes.py` — all 12 WRITE tools in a single file
- `tests/unit/test_aos8_write.py` — unit tests with mocked API responses
- `platforms/aos8/__init__.py` updated — `TOOLS["writes"]` key added, existing category keys unchanged

**Not in scope for Phase 5:** Guided prompts (Phase 6), documentation updates (Phase 6), testing & integration audit (Phase 7).

</domain>

<decisions>
## Implementation Decisions

### File Organization
- **D-01:** All 12 WRITE tools go in a single new file: `src/hpe_networking_mcp/platforms/aos8/tools/writes.py`. Estimated ~350–400 lines — well under the 500-line limit. Mirrors Phase 4's `differentiators.py` single-file approach.
- **D-02:** A new `"writes"` category key is added to the `TOOLS` dict in `platforms/aos8/__init__.py`. The 12 WRITE tool names live under this key. Existing 5 category keys (health, clients, alerts, wlan, troubleshooting) and the Phase 4 `"differentiators"` key are not modified.

### Tool Pattern (manage_X)
- **D-03:** WRITE-01..09 follow the `aos8_manage_<object>` unified pattern: a single tool per config object type with `action_type: str` accepting `"create"`, `"update"`, or `"delete"`. Mirrors Central's `central_manage_wlan_profile` pattern.
- **D-04:** WRITE-10 (`aos8_disconnect_client`) and WRITE-11 (`aos8_reboot_ap`) are specific named action tools — not `manage_X` pattern. They are simple one-shot operational commands with no create/update/delete concept.
- **D-05:** WRITE-12 (`aos8_write_memory`) is a standalone tool that takes `config_path: str` (required) and persists staged config changes. It is never called by other write tools.

### Write Tool Annotations and Tags
- **D-06:** `manage_X` tools (WRITE-01..09) use `tags={"aos8_write", "aos8_write_delete"}` — the delete capability makes them potentially destructive.
- **D-07:** Operational tools WRITE-10 and WRITE-11 use `tags={"aos8_write"}` only (not destructive by default).
- **D-08:** `aos8_write_memory` (WRITE-12) uses `tags={"aos8_write"}` — it is a commit/persist operation, not destructive in the delete sense.
- **D-09:** All write tools use `ToolAnnotations(readOnlyHint=False, destructiveHint=True, idempotentHint=False, openWorldHint=True)` as their annotations constant (same as Central's `WRITE_DELETE`). A `WRITE` constant can be defined with `destructiveHint=False` for WRITE-10, WRITE-11, WRITE-12 if the distinction matters — researcher to verify if any of those are idempotent.

### write_memory Response Contract
- **D-10:** Every config write tool (WRITE-01..09) always includes `requires_write_memory_for` in its response — even if the operation failed (in which case it is an empty list). Shape:
  ```python
  {"result": ..., "requires_write_memory_for": ["<config_path>"]}
  ```
- **D-11:** Operational tools WRITE-10 and WRITE-11 also include `requires_write_memory_for: []` in their response — always present, always empty. This lets AI callers trust the field exists unconditionally without type-checking the tool identity.
- **D-12:** `aos8_write_memory` itself does NOT include `requires_write_memory_for` in its response — it IS the write_memory call, so the field would be circular.

### config_path on Writes
- **D-13:** `config_path: str` is a **required** parameter (no default) on all 9 `manage_X` config tools (WRITE-01..09) and `aos8_write_memory` (WRITE-12). The operator must always specify the target hierarchy node explicitly. Implemented as a non-optional `Annotated[str, Field(...)]` — a missing `config_path` raises a Pydantic validation error before the tool runs.
- **D-14:** `aos8_disconnect_client` (WRITE-10) and `aos8_reboot_ap` (WRITE-11) do NOT take `config_path` — they operate on a specific device/MAC by identifier, not a config hierarchy node.

### Write Payload Design
- **D-15:** Each `manage_X` tool accepts a freeform `payload: dict` for the config object body, plus `config_path: str` and `action_type: str`. Matches Central's `central_manage_wlan_profile` approach. Researcher documents key payload fields per object type in the plan — schema knowledge lives in docstrings, not tool type annotations.
- **D-16:** For `delete` actions, `payload` may be an empty dict or minimal identifier. Researcher to confirm whether AOS8 requires a body for delete operations or just the object name in the URL path.

### AOS8 Delete Mechanics
- **D-17:** AOS8 is POST-only (no HTTP DELETE). Researcher is responsible for identifying the correct delete endpoint per config object type. Likely patterns: POST to `/v1/configuration/object/<name>?config_path=<path>` with a body flag (e.g., `{"_action": "delete"}`), or a dedicated delete path. Researcher must confirm for all 9 object types (WRITE-01..09) and document the exact endpoint + body shape in the plan.
- **D-18:** If a single delete endpoint pattern covers all 9 object types, the `writes.py` implementation can use a shared helper. If patterns differ per object, each `manage_X` function handles its own delete path — no forced abstraction.

### Plan Sequencing (TDD)
- **D-19:** 3 plans mirroring Phase 4 discipline:
  - **Plan 05-01 (Wave 0):** Write all WRITE tool tests (`test_aos8_write.py`) — all red baseline. No implementation.
  - **Plan 05-02 (Implementation):** Implement all 12 WRITE tools in `writes.py`. Tests go green.
  - **Plan 05-03 (Wiring):** Update `platforms/aos8/__init__.py` — add `"writes"` TOOLS dict entry; importlib loop covers new module automatically.

### Claude's Discretion
- Exact AOS8 REST endpoint paths for each config object create/update/delete (researcher to verify)
- Whether AOS8 uses a single delete endpoint pattern or per-object paths (researcher to confirm for all 9 types)
- JSON fixture file names and structure for write tool test mocks (place in `tests/unit/fixtures/aos8/` per Phase 3-4 pattern)
- `ToolAnnotations` constant naming in `writes.py` — whether to define separate `WRITE` and `WRITE_DELETE` constants or a single constant for all 12 tools
- Whether `aos8_disconnect_client` and `aos8_reboot_ap` are idempotent — researcher to confirm for `idempotentHint`

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Write Tool Pattern (Central reference)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/wlan_profiles.py` — canonical `manage_X` write tool: `action_type`, `payload: dict`, `confirmed: bool`, `WRITE_DELETE` annotations, `tags={"central_write_delete"}`, `elicitation_handler` usage
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/roles.py` — second write tool reference for role/ACL style objects

### AOS8 Tool Implementation Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py` — canonical AOS8 tool file: `@tool(annotations=READ_ONLY)`, Context injection, `run_show()` / `get_object()` usage, error handling
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — reference for tools using `get_object()` with `config_path`

### AOS8 Shared Helpers
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` — `run_show()`, `get_object()`, `strip_meta()`, `format_aos8_error()`. Write tools will likely need a `post_object()` helper; researcher to determine if one needs to be added.

### AOS8 Platform Module
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` — `AOS8Client.request()`, `AOS8APIError`, `AOS8AuthError`
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — `TOOLS` dict structure to extend; `register_tools()` import loop
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/_registry.py` — `tool()` shim for decorating WRITE tool functions

### Write Gate / Tag Registry
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/_common/tool_registry.py` — `_WRITE_TAG_BY_PLATFORM["aos8"]` = `{"aos8_write", "aos8_write_delete"}`; gate config attr = `"enable_aos8_write_tools"`

### Elicitation Middleware
- `hpe-networking-mcp/src/hpe_networking_mcp/middleware/elicitation.py` — `elicitation_handler` used in Central write tools for the `confirmed` parameter pattern; WRITE tools should follow the same pattern

### Prior Phase Research
- `.planning/phases/03-read-tools/03-RESEARCH.md` — AOS8 API endpoint patterns, config-object URL shapes, `_global_result` semantics
- `.planning/phases/04-differentiator-tools/04-CONTEXT.md` — confirmed `get_object()` / `run_show()` helper contract

### Test Patterns
- `hpe-networking-mcp/tests/unit/test_aos8_read_health.py` — category test file structure with httpx mock, `pytest.mark.unit`, fixture loading from `tests/unit/fixtures/aos8/`
- `hpe-networking-mcp/tests/conftest.py` — shared `secrets_dir` and `loguru_capture` fixtures

### Style & Constraints
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/__init__.py` — `READ_ONLY` annotations constant (WRITE tools will define their own `WRITE` / `WRITE_DELETE` constants in `writes.py`)
- Project constraint: 500 lines max per file, 50 lines max per function, Google-style docstrings

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `run_show(client, command, *, config_path=None)` and `get_object(client, object_path, *, config_path="/md")` in `_helpers.py` — read helpers; write tools will likely need a `post_object()` helper for mutation calls
- `ctx.lifespan_context["aos8_client"]` — client access pattern shared across all tool functions
- `format_aos8_error(exc)` in `_helpers.py` — consistent error formatting for `try/except (AOS8APIError, AOS8AuthError, httpx.HTTPError)` blocks
- `elicitation_handler` from `middleware/elicitation.py` — already used in Central write tools for the `confirmed` parameter

### Established Patterns
- `async def aos8_<name>(ctx: Context, ...) -> dict[str, Any]` — all tools are async, return dicts
- `@tool(name="aos8_<name>", annotations=<CONST>, tags={...})` decorator with explicit string name and tag set
- `try/except (AOS8APIError, AOS8AuthError, httpx.HTTPError) as exc` → `{"error": format_aos8_error(exc)}`
- Fixture JSON files in `tests/unit/fixtures/aos8/` (established in Phase 3)
- `pytest.mark.unit` on all test functions; `pytest-asyncio` auto mode

### Integration Points
- `platforms/aos8/__init__.py:TOOLS` — add `"writes": [list of 12 WRITE tool names]`
- The importlib loop in `register_tools()` already iterates `TOOLS.keys()` — adding `"writes"` key automatically triggers import of `writes.py`; no other wiring change needed
- Write tools are invisible unless `ENABLE_AOS8_WRITE_TOOLS=true` — gate filtering happens in `tool_registry.py:is_tool_enabled()` at registration time

### write_memory Note
- WRITE-12 (`aos8_write_memory`) takes `config_path: str` (required) — the same `config_path` value returned in `requires_write_memory_for` by any preceding write tool. This is the only path to persist AOS8 config changes to startup-config.

</code_context>

<specifics>
## Specific Ideas

- `requires_write_memory_for` field is always present in WRITE-01..11 responses (empty list for operational tools), so AI callers never need to check tool identity before deciding whether to call `aos8_write_memory`.
- `aos8_write_memory` does NOT include `requires_write_memory_for` in its own response to avoid circularity.
- Payload design mirrors Central `manage_X` — freeform `payload: dict` keeps the tool surface lean; schema knowledge lives in docstrings and the researcher's plan documentation.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 05-write-tools*
*Context gathered: 2026-04-28*
