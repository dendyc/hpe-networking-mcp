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

### Validated

*(Validated in Phase 1 & Phase 2)*

- ✓ AOS8 API client with UIDARUBA token auth (login/logout, session reuse, HTTPS port 4343) — Validated in Phase 2: api-client
- ✓ Docker secrets: `aos8_host`, `aos8_username`, `aos8_password`, `aos8_port`, `aos8_verify_ssl` — Validated in Phase 1: foundation
- ✓ Unit tests with mocked API responses (no live system required for CI) — Validated in Phase 2: api-client
- ✓ Platform auto-disable if AOS8 secrets absent/empty (consistent with other platforms) — Validated in Phase 2: api-client
- ✓ health tool updated to probe AOS8 alongside existing platforms — Validated in Phase 2: api-client
- ✓ Write tools gated behind `ENABLE_AOS8_WRITE_TOOLS=true` with elicitation confirmation — Validated in Phase 2: api-client

### Validated

*(Validated in Phase 3: read-tools)*

- ✓ AOS8 supports both Mobility Conductor hierarchy and standalone controllers — Validated in Phase 3: read-tools
- ✓ config_path defaults to `/md`, overridable per-call for targeted MD/AP-group operations — Validated in Phase 3: read-tools
- ✓ Health & devices: controller inventory, AP inventory, device stats, firmware status (READ-01..08) — Validated in Phase 3: read-tools
- ✓ Client connectivity: client lookup by MAC/IP/username, connection state, association details (READ-09..12) — Validated in Phase 3: read-tools
- ✓ Alerts & events: active alerts, event history, audit logs (READ-13..15) — Validated in Phase 3: read-tools
- ✓ WLAN & config: SSID/WLAN profile read, configuration object read (READ-16..19) — Validated in Phase 3: read-tools
- ✓ Troubleshooting: ping, traceroute, show command passthrough, logs, controller stats, ARM history, RF monitor (READ-20..26) — Validated in Phase 3: read-tools
- ✓ Dynamic mode: 3 meta-tools (`aos8_list_tools`, `aos8_get_tool_schema`, `aos8_invoke_tool`) wired via `build_meta_tools` — Validated in Phase 3: read-tools

### Validated

*(Validated in Phase 5: write-tools)*

- ✓ AOS8 write tools: 12 WRITE tools (WRITE-01..12) — SSID/VAP/AP-group/role/VLAN/AAA/ACL/netdest config, client disconnect, AP reboot, write-memory — Validated in Phase 5: write-tools
- ✓ Write tools gated behind `ENABLE_AOS8_WRITE_TOOLS=true`; elicitation middleware extended for `{"aos8_write","aos8_write_delete"}` tags — Validated in Phase 5: write-tools
- ✓ `aos8_write_memory` uses dedicated `/v1/configuration/object/write_memory` endpoint; manage_X tools return `requires_write_memory_for=[config_path]` — Validated in Phase 5: write-tools
- ✓ TDD red→green cycle: 44 contract tests in `test_aos8_write.py`; full suite 737/737 passing — Validated in Phase 5: write-tools

### Validated

*(Validated in Phase 6: guided-prompts-documentation)*

- ✓ 9 AOS8 guided prompts (PROMPT-01..09): triage_client, triage_ap, health_check, audit_change, rf_analysis, wlan_review, client_flood, compare_md_config, pre_change_check — Validated in Phase 6: guided-prompts-documentation
- ✓ `aos8_pre_change_check` explicitly surfaces `aos8_write_memory` contract (WRITE-12) to operators — Validated in Phase 6: guided-prompts-documentation
- ✓ Operator-facing `INSTRUCTIONS.md` at repo root (config_path semantics, write_memory contract, show_command, Conductor vs. standalone, prompt index) — Validated in Phase 6: guided-prompts-documentation
- ✓ `docs/TOOLS.md` AOS8 section: all 38 tools + 9 prompts documented — Validated in Phase 6: guided-prompts-documentation
- ✓ README.md AOS8 capability row, secrets reference, auto-disable example, "38 + 9 prompts" count — Validated in Phase 6: guided-prompts-documentation
- ✓ Version bumped to 2.4.0.0; CHANGELOG.md [2.4.0.0] entry complete — Validated in Phase 6: guided-prompts-documentation

### Validated

*(Validated in Phase 7: testing-integration)*

- ✓ 9 AOS8 differentiator read tools (DIFF-01..09): MD hierarchy, effective config, pending changes, RF neighbors, cluster state, air monitors, AP wired ports, IPsec tunnels, unified MD health check aggregator — Validated in Phase 7: testing-integration
- ✓ AOS8 tool count raised to 47 (38 → 47); TOOLS["differentiators"] wired in __init__.py — Validated in Phase 7: testing-integration
- ✓ TDD red→green cycle: 13 contract tests in test_aos8_read_differentiators.py; 2 security token-leak tests; all 764 unit tests passing — Validated in Phase 7: testing-integration
- ✓ UIDARUBA token never logged at tool layer (TEST-04 verified end-to-end) — Validated in Phase 7: testing-integration
- ✓ All 6 existing platform test suites unmodified and passing (non-regression, D-07) — Validated in Phase 7: testing-integration

### Validated

*(Validated in Phase 8: fix-diff-tools-production-bug)*

- ✓ `differentiators.py` uses canonical `run_show`/`get_object` from `_helpers`; local `_show`/`_object` helpers deleted — response-contract bug fixed — Validated in Phase 8: fix-diff-tools-production-bug
- ✓ All 13 DIFF tool test mocks updated to wrap responses in `_resp()` (MagicMock with `.json.return_value`), matching real `AOS8Client.request()` contract — Validated in Phase 8: fix-diff-tools-production-bug
- ✓ 764 unit tests pass; zero regressions across all platforms — Validated in Phase 8: fix-diff-tools-production-bug

### Validated

*(Validated in Phase 9: phase-4-closure-documentation-accuracy)*

- ✓ `04-VERIFICATION.md` created as delegated stub (0/0 score) documenting Phase 4 was absorbed into Phase 7; cross-references all 9 DIFF tools and Phase 7 VERIFICATION — Validated in Phase 9: phase-4-closure-documentation-accuracy
- ✓ REQUIREMENTS.md DIFF-01..09 all `[x]`; traceability table shows Complete for all 9 DIFF rows — Validated in Phase 9: phase-4-closure-documentation-accuracy
- ✓ README.md, docs/TOOLS.md, CHANGELOG.md corrected to 47 AOS8 tools (was 38); Differentiators (9) subsection added to TOOLS.md — Validated in Phase 9: phase-4-closure-documentation-accuracy
- ✓ `server.py` `execute_description` includes `aos8_` prefix; regression test in `test_server_code_mode.py` guards all 7 platform prefixes — Validated in Phase 9: phase-4-closure-documentation-accuracy
- ✓ Version bumped to 2.4.0.1; CHANGELOG.md [2.4.0.1] entry documents Phase 8 + Phase 9 fixes — Validated in Phase 9: phase-4-closure-documentation-accuracy
- ✓ Full unit suite at 766 tests (764 + 2 new code-mode tests); zero regressions — Validated in Phase 9: phase-4-closure-documentation-accuracy

### Active

*(No active requirements — all milestone work complete through Phase 9)*

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
*Last updated: 2026-04-29 after Phase 9 (phase-4-closure-documentation-accuracy) completion — v1.0 milestone fully complete (all 9 phases, 26 plans, 766 tests)*
