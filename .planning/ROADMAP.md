# Roadmap — AOS8/Mobility Conductor Platform Module

## Overview

This roadmap delivers a seventh platform module — Aruba OS 8 / Mobility Conductor — to the `hpe-networking-mcp` fork, achieving feature parity with the existing Aruba Central module. Work is structured in seven phases that build from foundation outward: secrets and gating, then the API client, then read tools, then AOS8-specific differentiators, then gated write tools, then guided prompts and documentation, and finally end-to-end test and integration verification.

Every phase delivers a coherent, verifiable capability. Read tools are split into a single Phase 3 (health/inventory, clients, alerts, WLAN/config, troubleshooting) to keep the read surface as one cohesive milestone. Differentiators (DIFF) are isolated in Phase 4 because they are AOS8-unique value beyond Central parity. Write tools are walled off in Phase 5 behind the `ENABLE_AOS8_WRITE_TOOLS` gate. Prompts plus user-facing documentation form Phase 6. Phase 7 closes the loop with comprehensive testing and a regression check that no existing platform broke.

## Phases

- [ ] **Phase 1: Platform Foundation** — Secrets, gating, registry, and Compose plumbing in place
- [ ] **Phase 2: API Client** — `AOS8Client` with token-reuse auth, error handling, and health probe
- [ ] **Phase 3: Read Tools** — Health, clients, alerts, WLAN, and troubleshooting read surface
- [ ] **Phase 4: Differentiator Tools** — Conductor-hierarchy, RF, cluster, and IPsec read tools unique to AOS8
- [x] **Phase 5: Write Tools** — Gated config-mutation and operational write tools with explicit `write_memory` (completed 2026-04-28)
- [ ] **Phase 6: Guided Prompts & Documentation** — Workflow prompts plus `INSTRUCTIONS.md`/`README.md`/`TOOLS.md` updates
- [ ] **Phase 7: Testing & Integration** — Mocked unit tests, log-leak audits, and regression of existing platforms

## Phase Details

### Phase 1: Platform Foundation
**Goal:** Establish the AOS8 platform's secrets, write-gate, and registry wiring so the server can recognize and conditionally enable the module.
**Depends on:** Nothing (first phase)
**Requirements:** FOUND-01, FOUND-02, FOUND-03, FOUND-04, FOUND-05
**Success Criteria** (what must be TRUE):
  1. Operator can drop AOS8 secret files into `secrets/` and the server detects them on startup with no other configuration.
  2. With AOS8 secrets absent or empty, the server starts cleanly and the AOS8 platform reports as auto-disabled — no crash, no traceback.
  3. With `ENABLE_AOS8_WRITE_TOOLS` unset or `false`, no `aos8_write*`-tagged tools appear in the registry; setting it to `true` exposes them.
  4. `docker-compose.yml` includes the five AOS8 secrets and the write-gate env var, with `.example` templates committed.
**Plans:** 4/4 plans executed
  - [x] 01-01-PLAN.md — Wave 0 test scaffolding (test_aos8_config.py, conftest fixture, registry/config test extensions)
  - [x] 01-02-PLAN.md — config.py implementation (AOS8Secrets, _load_aos8, ServerConfig wiring)
  - [x] 01-03-PLAN.md — tool_registry.py three-dict registration (REGISTRIES, _WRITE_TAG_BY_PLATFORM, _GATE_CONFIG_ATTR)
  - [x] 01-04-PLAN.md — docker-compose.yml + 5 .example secret templates

### Phase 2: API Client
**Goal:** Deliver a robust, token-reusing AOS8 API client that authenticates safely, handles AOS8 error semantics, and integrates with the server's health probe.
**Depends on:** Phase 1
**Requirements:** CLIENT-01, CLIENT-02, CLIENT-03, CLIENT-04, CLIENT-05, CLIENT-06, CLIENT-07, CLIENT-08, CLIENT-09, CLIENT-10
**Success Criteria** (what must be TRUE):
  1. The client logs in once, reuses the UIDARUBA token across calls, and refreshes exactly once on a 401 — concurrent requests never trigger parallel logins.
  2. An unreachable Conductor at startup does not block the server; the client only attempts login on the first tool invocation.
  3. AOS8 `_global_result.status != 0` responses surface as `AOS8APIError` to the caller — never silently as bad data.
  4. The UIDARUBA token never appears in any log line, error message, or URL fragment that gets logged.
  5. The server's existing `health` tool reports AOS8 reachability alongside the other six platforms.
**Plans:** 3/3 plans executed
  - [x] 02-01-PLAN.md — Wave 0 TDD scaffold: test_aos8_client.py + loguru_capture fixture + show_version JSON fixture (CLIENT-01..CLIENT-10 red tests)
  - [x] 02-02-PLAN.md — AOS8Client implementation + platform module skeleton (__init__, _registry, client.py)
  - [x] 02-03-PLAN.md — health.py _probe_aos8 + server.py lifespan/registration/write-gate wiring

### Phase 3: Read Tools
**Goal:** Surface comprehensive read-only visibility into AOS8 — health, clients, alerts, WLAN config, and troubleshooting — matching Aruba Central's read coverage.
**Depends on:** Phase 2
**Requirements:** READ-01, READ-02, READ-03, READ-04, READ-05, READ-06, READ-07, READ-08, READ-09, READ-10, READ-11, READ-12, READ-13, READ-14, READ-15, READ-16, READ-17, READ-18, READ-19, READ-20, READ-21, READ-22, READ-23, READ-24, READ-25, READ-26
**Success Criteria** (what must be TRUE):
  1. An operator can enumerate every controller, AP, and connected client in the network without leaving the MCP interface.
  2. An operator can locate a specific client by MAC, IP, or username and retrieve full association/auth/connection detail.
  3. An operator can review active alarms, configuration audit trail, and event history for any scope.
  4. An operator can read every SSID profile, virtual AP, AP group, and user role definition currently configured.
  5. An operator can run ping, traceroute, and any AOS8 CLI show command through the MCP and receive structured JSON (with `_meta` stripped).
**Plans:** 7 plans
  - [x] 03-01-PLAN.md — Wave 0 scaffold: conftest stub + tools/__init__.py + _helpers.py + JSON fixtures + 26 red tests
  - [x] 03-02-PLAN.md — health.py: READ-01..08 (controllers, AP database/active/detail, BSS/radio, version, licenses)
  - [x] 03-03-PLAN.md — clients.py: READ-09..12 (list, find, detail, history)
  - [x] 03-04-PLAN.md — alerts.py: READ-13..15 (alarms, audit-trail, events)
  - [x] 03-05-PLAN.md — wlan.py: READ-16..19 (SSID/VAP/AP-group/role via config-object endpoint)
  - [x] 03-06-PLAN.md — troubleshooting.py: READ-20..26 (ping, traceroute, show_command, logs, controller stats, ARM, RF monitor)
  - [x] 03-07-PLAN.md — __init__.py wire-up: TOOLS dict + build_meta_tools

### Phase 4: Differentiator Tools
**Goal:** Expose the AOS8-specific capabilities (Conductor hierarchy, effective config, ARM neighbors, cluster state, IPsec) that go beyond Central parity.
**Depends on:** Phase 3
**Requirements:** DIFF-01, DIFF-02, DIFF-03, DIFF-04, DIFF-05, DIFF-06, DIFF-07, DIFF-08, DIFF-09
**Success Criteria** (what must be TRUE):
  1. An operator can render the full Conductor → Managed Device hierarchy with config_path values for every node.
  2. An operator can request the effective resolved configuration for a specific MD or AP group after inheritance is applied.
  3. An operator can identify staged configuration changes that have not yet been persisted via `write_memory`.
  4. An operator can retrieve a unified per-MD health report (`aos8_get_md_health_check`) covering APs, clients, alarms, and firmware in one call.
  5. An operator can inspect cluster membership, RF neighbors, air-monitor APs, AP wired ports, and IPsec tunnel state through dedicated tools.
**Plans:** TBD

### Phase 5: Write Tools
**Goal:** Deliver gated, operator-confirmed write capabilities for SSID/VAP/AP-group/role/VLAN/AAA/ACL/netdest configuration plus client/AP operational actions and explicit `write_memory`.
**Depends on:** Phase 4
**Requirements:** WRITE-01, WRITE-02, WRITE-03, WRITE-04, WRITE-05, WRITE-06, WRITE-07, WRITE-08, WRITE-09, WRITE-10, WRITE-11, WRITE-12
**Success Criteria** (what must be TRUE):
  1. With `ENABLE_AOS8_WRITE_TOOLS=true`, an operator can create, update, or delete SSID profiles, virtual APs, AP groups, user roles, VLANs, AAA servers, AAA server groups, ACLs, and netdestinations — each requiring an explicit `config_path` (no default).
  2. Every write tool's response surfaces a `requires_write_memory_for` field listing the config_paths needing persistence.
  3. `aos8_write_memory` is the only path to persist config; no other write tool calls it implicitly.
  4. An operator can force-disconnect a client by MAC and reboot a specific AP by name through MCP write tools.
  5. Every write and delete tool fires the elicitation confirmation middleware before executing.
**Plans:** 3/3 plans complete
  - [x] 05-01-PLAN.md — Wave 0 red-baseline tests for all 12 WRITE tools (test_aos8_write.py + fixtures)
  - [x] 05-02-PLAN.md — Implement writes.py (12 tools), add post_object() helper, extend ElicitationMiddleware for aos8 tags
  - [x] 05-03-PLAN.md — Wire TOOLS["writes"] in platforms/aos8/__init__.py

### Phase 6: Guided Prompts & Documentation
**Goal:** Provide AOS8 workflow prompts that stitch tools into recognizable operator tasks, and update all user-facing documentation to reflect the new platform.
**Depends on:** Phase 5
**Requirements:** PROMPT-01, PROMPT-02, PROMPT-03, PROMPT-04, PROMPT-05, PROMPT-06, PROMPT-07, PROMPT-08, PROMPT-09, DOCS-01, DOCS-02, DOCS-03, DOCS-04, DOCS-05
**Success Criteria** (what must be TRUE):
  1. An operator can invoke a single guided prompt to triage a problem client, deep-dive an AP, run a network-wide health check, audit recent changes, or perform a pre-change checklist.
  2. `INSTRUCTIONS.md` documents AOS8 config_path semantics, the `write_memory` contract, show command usage, and Conductor-vs-standalone behavior.
  3. `README.md`'s capability table includes an AOS8 row with secrets reference and updated tool count, and the auto-disable example covers AOS8.
  4. `docs/TOOLS.md` lists every new AOS8 tool with its description and tags.
  5. `CHANGELOG.md` carries a new version entry and `pyproject.toml` reflects the bumped version.
**Plans:** TBD

### Phase 7: Testing & Integration
**Goal:** Prove correctness through mocked unit tests, prove safety through token-leak audits, and prove non-regression across all six existing platforms.
**Depends on:** Phase 6
**Requirements:** TEST-01, TEST-02, TEST-03, TEST-04, TEST-05, TEST-06
**Success Criteria** (what must be TRUE):
  1. The full pytest suite (unit + integration markers) passes locally and in CI without any live AOS8 system.
  2. `AOS8Client` tests cover login success/failure, token reuse, 401 single-refresh behavior, `_global_result` error detection, `verify_ssl` warning, and token masking.
  3. Every read tool category and every write tool has at least one fixture-driven unit test, and write/delete tag coverage is asserted automatically.
  4. A dedicated test scans captured log output across the entire client+tool surface and fails if any UIDARUBA value appears.
  5. Mist, Central, GreenLake, ClearPass, Apstra, and Axis test suites all pass unmodified.
**Plans:** TBD

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Platform Foundation | 4/4 | Complete | 2026-04-28 |
| 2. API Client | 3/3 | Complete | 2026-04-28 |
| 3. Read Tools | 0/7 | Not started | - |
| 4. Differentiator Tools | 0/0 | Not started | - |
| 5. Write Tools | 3/3 | Complete   | 2026-04-28 |
| 6. Guided Prompts & Documentation | 0/0 | Not started | - |
| 7. Testing & Integration | 0/0 | Not started | - |

## Coverage

- **v1 requirements mapped:** 71 / 71
- **Orphans:** 0
- **Duplicates:** 0

| Category | Count | Phase |
|----------|-------|-------|
| FOUND-01 — FOUND-05 | 5 | Phase 1 |
| CLIENT-01 — CLIENT-10 | 10 | Phase 2 |
| READ-01 — READ-26 | 26 | Phase 3 |
| DIFF-01 — DIFF-09 | 9 | Phase 4 |
| WRITE-01 — WRITE-12 | 12 | Phase 5 |
| PROMPT-01 — PROMPT-09 + DOCS-01 — DOCS-05 | 14 | Phase 6 |
| TEST-01 — TEST-06 | 6 | Phase 7 |

---
*Last updated: 2026-04-28 after Phase 2 Plan 03 execution (Phase 2 complete)*
