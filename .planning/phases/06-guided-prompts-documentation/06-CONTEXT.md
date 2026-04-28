# Phase 6: Guided Prompts & Documentation - Context

**Gathered:** 2026-04-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver 9 AOS8 guided workflow prompts (`PROMPT-01..09`) in `platforms/aos8/tools/prompts.py` using the `@mcp.prompt` + `register(mcp)` pattern, wire them into `aos8/__init__.py`, and update/create all user-facing documentation (`INSTRUCTIONS.md`, `README.md`, `TOOLS.md`, `CHANGELOG.md`, `pyproject.toml`).

**Not in scope for Phase 6:** Any new AOS8 tools, DIFF tool work, comprehensive test suite (Phase 7).

</domain>

<decisions>
## Implementation Decisions

### Prompt Structure
- **D-01:** All prompts follow the **fully explicit** style — numbered steps with specific `aos8_*` tool names and key parameter examples. Matches Central's `prompts.py` depth. Guides AI models reliably without ambiguity.
- **D-02:** Every prompt ends with a **summary + next steps** section — summarize findings and recommend operator actions. Matches how Central prompts close.
- **D-03:** Contextual prompts accept **typed parameters** where it improves workflow specificity:
  - `aos8_triage_client(mac_address: str)` — find a specific client
  - `aos8_triage_ap(ap_name: str)` — deep-dive a specific AP
  - `aos8_rf_analysis(config_path: str = "/md")` — scoped RF report
  - `aos8_compare_md_config(md_path_1: str, md_path_2: str)` — compare two MD config paths
  - `aos8_client_flood(config_path: str = "/md")` — scoped flood investigation
  - `aos8_pre_change_check(config_path: str)` — required scope for change check
  - Parameterless: `aos8_health_check`, `aos8_audit_change`, `aos8_wlan_review`

### Prompt Coverage
- **D-04:** Implement **all 9 prompts** (PROMPT-01..09) per requirements. No prompts deferred to Phase 7.

### Prompt Registration Pattern
- **D-05:** `prompts.py` in `src/hpe_networking_mcp/platforms/aos8/tools/` with a `register(mcp)` function. Registered in `aos8/__init__.py` in a `try/except` block after tool registration — exactly mirrors `central/__init__.py` pattern.

### INSTRUCTIONS.md
- **D-06:** Create a new `INSTRUCTIONS.md` at repo root (same level as `README.md`) covering:
  1. `config_path` semantics — Conductor hierarchy, `/md`, `/md/<device>`, `/md/<ap-group>`, standalone
  2. `write_memory` contract — why it's explicit, how `requires_write_memory_for` works, when to call it
  3. `show_command` passthrough — AOS8 CLI show commands via `aos8_show_command`, `_meta` stripping
  4. Conductor vs. standalone behavior — how to connect to each, config_path differences
  5. **Guided prompt index** — list all 9 prompts with their purpose and parameters

### Plan Sequencing
- **D-07:** **2-plan structure** (no TDD Wave 0 — string-returning functions don't benefit):
  - **Plan 06-01:** `prompts.py` (all 9 prompts + `register()`) + `aos8/__init__.py` wiring + `tests/unit/test_aos8_prompts.py` smoke test
  - **Plan 06-02:** All documentation — `INSTRUCTIONS.md` (new), `README.md` (AOS8 row + secrets + tool count), `docs/TOOLS.md` (AOS8 tool reference), `CHANGELOG.md` (new version entry), `pyproject.toml` (version bump)

### Test Coverage for Prompts
- **D-08:** **Registration smoke test only** — `test_aos8_prompts.py` calls `register(mcp)` and asserts all 9 prompts are registered and return non-empty strings. No assertions on string content (brittle). One `pytest.mark.unit` test per prompt registration check.

### Claude's Discretion
- Exact parameter names and defaults for the 3 parameterless prompts vs. the 6 parameterized ones (researcher to confirm which benefit most from optional vs. required params)
- Precise version number to bump in `pyproject.toml` (patch vs. minor — researcher to check current version and project conventions)
- Order of sections in `INSTRUCTIONS.md` (researcher to check if any existing docs establish a structure pattern)
- Whether `README.md` has an existing AOS8 row stub or needs a full new row inserted (researcher to check current table state)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Guided Prompt Pattern (reference implementations)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/prompts.py` — canonical prompt file: `@mcp.prompt`, `register(mcp)` function, typed parameters, numbered steps with explicit tool names, summary+next-steps endings
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/mist/tools/prompts.py` — second reference: multi-step provisioning prompts with complex branching logic

### Prompt Registration in __init__.py
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/__init__.py` — canonical `try/except` registration block for `prompts.register(mcp)` after tool loop
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — file to modify: add prompts registration block after `build_meta_tools`

### AOS8 Tool Surface (prompts reference these tools)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py` — READ-01..08 tool names
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/clients.py` — READ-09..12 tool names
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/alerts.py` — READ-13..15 tool names
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — READ-16..19 tool names
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py` — READ-20..26 tool names (ping, traceroute, show_command, logs, controller_stats, arm_history, rf_monitor)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/writes.py` — WRITE-01..12 tool names (manage_ssid_profile, manage_virtual_ap, manage_ap_group, manage_user_role, manage_vlan, manage_aaa_server, manage_aaa_server_group, manage_acl, manage_netdestination, disconnect_client, reboot_ap, write_memory)

### Documentation Files to Update
- `hpe-networking-mcp/README.md` — existing file; needs AOS8 row in capability table, secrets reference section, updated tool count
- `hpe-networking-mcp/docs/TOOLS.md` — existing file; needs AOS8 tool reference section
- `hpe-networking-mcp/CHANGELOG.md` — existing file; needs new version entry
- `hpe-networking-mcp/pyproject.toml` — existing file; version bump

### Requirements Reference
- `.planning/REQUIREMENTS.md` §PROMPT — PROMPT-01..09 definitions with full descriptions
- `.planning/REQUIREMENTS.md` §DOCS — DOCS-01..05 definitions for what each doc must contain

### Test Pattern
- `hpe-networking-mcp/tests/unit/test_aos8_read_health.py` — test file structure: `pytest.mark.unit`, fixture imports, async test functions
- `hpe-networking-mcp/tests/conftest.py` — shared fixtures

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `register(mcp)` function pattern — copy exactly from `central/tools/prompts.py`; FastMCP `@mcp.prompt` decorator works identically on AOS8 `mcp` instance
- `ctx.lifespan_context["aos8_client"]` — NOT needed in prompts; prompts are string templates, not async functions that call the API
- All 38 AOS8 tool names are already wired and tested (Phases 3–5) — prompts can reference them directly by name

### Established Patterns
- Prompt functions are **synchronous** and return `str` (not `async def`) — Central and Mist both use plain `def`
- `@mcp.prompt` decorator auto-discovers the function name as the prompt name
- Parameters become part of the prompt's call signature in MCP — typed `str` params with defaults work as in normal Python
- `register(mcp)` receives the FastMCP instance that was bound in `_registry.py`

### Integration Points
- `aos8/__init__.py`: add prompts import block after the `build_meta_tools` call (or after the `logger.info` static mode line) — see Central's `__init__.py` for exact placement
- `tests/unit/`: add `test_aos8_prompts.py` (smoke test — register and check 9 prompt names registered)
- `INSTRUCTIONS.md`: new file at repo root alongside `README.md`

### write_memory in Prompts
- `aos8_pre_change_check` (PROMPT-09) should reference `aos8_write_memory` and explain the staged-config contract to the operator — this is the most important write-safety prompt
- Other prompts that invoke write tools (if any) should include a reminder step to call `aos8_write_memory` with the `config_path` from `requires_write_memory_for`

</code_context>

<specifics>
## Specific Ideas

- Central's `client_connectivity_check(mac_address)` is the closest analogue to `aos8_triage_client(mac_address)` — use it as the structural template
- `aos8_pre_change_check` should check pending changes, alarm state, and write_memory status before any maintenance window — the most operator-safety-critical prompt
- INSTRUCTIONS.md guided prompt index should list each prompt name, its parameters (if any), and a one-line description of the workflow — acts as a quick-reference card

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 06-guided-prompts-documentation*
*Context gathered: 2026-04-28*
