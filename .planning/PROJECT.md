# HPE Networking MCP — AOS8 Extension

## What This Is

A fork of the `hpe-networking-mcp` community project that adds **Aruba OS 8 (AOS8) / Mobility Conductor** as a seventh platform module alongside the existing six (Juniper Mist, Aruba Central, HPE GreenLake, Aruba ClearPass, Juniper Apstra, Axis Atmos Cloud). The goal is full feature parity with the existing Aruba Central module — health overview, device management, client troubleshooting, alerts/events, and WLAN/config management — surfaced through the same MCP tool patterns (dynamic mode, write-gate, Docker secrets) as every other platform.

## Core Value

An AI assistant that can monitor, troubleshoot, and configure an AOS8/Mobility Conductor network with the same depth and safety as it does Aruba Central — without the operator needing to touch the CLI.

## Requirements

### Validated

*(Inherited from forked codebase — already shipped upstream)*

- ✓ Juniper Mist integration — 35 tools + 2 guided prompts — existing
- ✓ Aruba Central integration — 73 tools + 12 guided prompts — existing
- ✓ HPE GreenLake integration — 10 tools — existing
- ✓ Aruba ClearPass integration — 140 tools — existing
- ✓ Juniper Apstra integration — 19 tools — existing
- ✓ Axis Atmos Cloud integration — 25 tools — existing
- ✓ Dynamic tool mode (3 meta-tools per platform; underlying tools hidden by default) — existing
- ✓ Cross-platform tools: site_health_check, site_rf_check, manage_wlan_profile — existing
- ✓ Write-tool safety gate (disabled by default, env flag + elicitation confirmation) — existing
- ✓ Docker Compose secrets management (per-credential files, never env vars) — existing
- ✓ Skills engine (markdown runbooks discoverable via skills_list / skills_load) — existing
- ✓ Middleware stack: null-strip, validation-catch, elicitation, retry (5xx + 429) — existing
- ✓ Non-root Docker container (mcpuser uid 1000), Streamable HTTP on port 8000 — existing

### Active

*(New work — this project)*

- [ ] AOS8 API client with UIDARUBA token auth (login/logout, session reuse, HTTPS port 4343)
- [ ] AOS8 supports both Mobility Conductor hierarchy and standalone controllers
- [ ] config_path defaults to `/md`, overridable per-call for targeted MD/AP-group operations
- [ ] Health & devices: controller inventory, AP inventory, device stats, firmware status
- [ ] Client connectivity: client lookup by MAC/IP/username, connection state, association details
- [ ] Troubleshooting: ping, traceroute, show command passthrough, client disconnect
- [ ] Alerts & events: active alerts, event history, audit logs
- [ ] WLAN & config: SSID/WLAN profile read, configuration object read/write
- [ ] Write tools gated behind `ENABLE_AOS8_WRITE_TOOLS=true` with elicitation confirmation
- [ ] Dynamic mode: 3 meta-tools (`aos8_list_tools`, `aos8_get_tool_schema`, `aos8_invoke_tool`)
- [ ] Guided prompts for common AOS8 workflows (parity with Central's 12 prompts)
- [ ] Docker secrets: `aos8_host`, `aos8_username`, `aos8_password`, `aos8_port`, `aos8_verify_ssl`
- [ ] Unit tests with mocked API responses (no live system required for CI)
- [ ] Platform auto-disable if AOS8 secrets absent/empty (consistent with other platforms)
- [ ] health tool updated to probe AOS8 alongside existing platforms

### Out of Scope

- AOS10 / Aruba Central cloud-managed APs — different API surface, different project
- HPE Aruba Instant (IAP) standalone mode — no Conductor, different auth model
- Replacing or modifying any existing platform module — fork extends, doesn't break
- GUI / web dashboard — pure MCP server, no frontend
- GreenLake write tools — GreenLake remains read-only (inherited constraint)

## Context

**Forked from:** `github.com/nowireless4u/hpe-networking-mcp` (community project, MIT license)

**AOS8 API characteristics:**
- HTTPS only, default port 4343 (Mobility Conductor and Managed Devices)
- Auth: POST `/v1/configuration/object/aaa_prof` login → `UIDARUBA` session token in cookie
- Only GET and POST HTTP methods supported (no PUT, PATCH, DELETE)
- Config API: `/v1/configuration/object/<object_name>?config_path=<path>`
- Show Command API: `/v1/configuration/showcommand?command=<cli-command>&UIDARUBA=<token>`
- Write Memory API: must be called per config_path after changes to persist config
- Responses: structured JSON; show command responses include `_meta` field listing available fields
- SSL: self-signed certs common in enterprise; `verify_ssl` flag needed

**Topology handled:**
- Mobility Conductor → Managed Devices hierarchy (config_path `/md`, `/md/<device>`, `/md/<ap-group>`)
- Standalone controllers (config_path `/mm` or direct device path)

**Existing project patterns to follow:**
- Platform module in `src/hpe_networking_mcp/platforms/aos8/`
- `_registry.py` module-level FastMCP holder, `client.py` API client, `tools/` directory
- `TOOLS` dict + `register_tools(mcp, config)` entry point
- Secrets loaded via `_read_secret()` from `SECRETS_DIR`
- All tools prefixed `aos8_*`
- Tags: `aos8_read` / `aos8_write` / `aos8_write_delete`
- Tests in `tests/unit/test_aos8_*.py` with `pytest.mark.unit`

## Constraints

- **API:** GET and POST only — no PUT/PATCH/DELETE; all mutations via POST
- **Auth:** UIDARUBA token must be reused across calls (not re-login per call) to avoid session exhaustion
- **Testing:** No live AOS8 system — all tests use mocked HTTP responses
- **Compatibility:** Must not break any existing platform module or test
- **Style:** Ruff lint + format, mypy type-checked, max 500 lines/file, Google-style docstrings
- **Write Memory:** Config changes require a follow-up Write Memory call per config_path — must be handled or documented clearly
- **SSL:** Default `verify_ssl=true`; support `false` for self-signed certs (common in AOS8 deployments)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Follow existing platform module pattern exactly | Consistency — new platform fits seamlessly into server bootstrap, middleware, dynamic mode | — Pending |
| config_path defaults to `/md`, overridable | Covers Conductor hierarchy (most common) while supporting targeted per-MD operations | — Pending |
| Session token reuse (not per-call login) | AOS8 docs explicitly warn against repeated login; prevents session exhaustion on Conductor | — Pending |
| Write Memory as explicit tool, not auto-called | Operator intent — AI should call it deliberately, not silently; matches staged-write pattern from Axis | — Pending |
| Mock-only tests (no live system) | No test hardware available; follow same mock pattern as existing unit tests | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-27 after initialization*
