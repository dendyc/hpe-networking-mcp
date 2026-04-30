# HPE Networking MCP — AOS8 Extension

## What This Is

A fork of the `hpe-networking-mcp` community project that adds **Aruba OS 8 (AOS8) / Mobility Conductor** as a seventh platform module alongside the existing six (Juniper Mist, Aruba Central, HPE GreenLake, Aruba ClearPass, Juniper Apstra, Axis Atmos Cloud). Ships full feature parity with the existing Aruba Central module — health overview, device management, client troubleshooting, alerts/events, WLAN/config management, and AOS8-unique differentiators (Conductor hierarchy, effective config, cluster state, RF neighbors, IPsec) — surfaced through the same MCP tool patterns (dynamic mode, write-gate, Docker secrets) as every other platform.

**Shipped v1.0:** 47 AOS8 tools + 9 guided prompts, 766-test suite, full operator documentation. Version 2.4.0.1.

## Core Value

An AI assistant that can monitor, troubleshoot, and configure an AOS8/Mobility Conductor network with the same depth and safety as it does Aruba Central — without the operator needing to touch the CLI.

## Current Milestone: v1.1 AOS8-Powered Migration Readiness

**Goal:** Enhance the `aos-migration-readiness` skill to replace paste-driven CLI data collection with live AOS8 API calls and produce a professional-grade, copy-pasteable migration assessment for field SEs and operators.

**Target features:**
- Stage 1: Live AOS8 data collection via platform tools — no CLI paste required for AOS8 source path
- Stage 3: Deeper VSG-anchored rules unlocked by live data (VRRP VIP, ARM profiles, cluster health, local user count, config objects)
- Stage 4: AOS8 inventory cross-referenced against Central API (AP model → firmware rec, count gap, SSID/role conflict detection)
- Stage 5: Real-time cutover prerequisite validation (live cluster state, controller firmware version, AP counts from AOS8)
- Output: Executive summary paragraph (GO/BLOCKED/PARTIAL) + structured REGRESSION/DRIFT/INFO findings, copy-pasteable into customer docs

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

*(Validated in v1.0 — Phases 1–9)*

- ✓ AOS8 platform foundation: secrets, write-gate, registry, Docker Compose wiring, graceful auto-disable — v1.0
- ✓ `AOS8Client` with UIDARUBA token reuse, asyncio.Lock refresh serialization, lazy login, token masking — v1.0
- ✓ 26 read tools: controller/AP inventory, client lookup, alerts/audit, WLAN/VAP/role config, troubleshooting (ping/traceroute/show_command) — v1.0
- ✓ 9 differentiator tools: MD hierarchy, effective config, pending changes, RF neighbors, cluster state, air monitors, AP wired ports, IPsec tunnels, unified MD health check — v1.0
- ✓ 12 gated write tools with elicitation: SSID/VAP/AP-group/role/VLAN/AAA/ACL/netdest lifecycle + client disconnect + AP reboot + write_memory — v1.0
- ✓ 9 guided workflow prompts + operator INSTRUCTIONS.md + README/TOOLS.md/CHANGELOG documentation — v1.0
- ✓ 766-test suite: UIDARUBA token-leak detection, all 6 existing platform suites passing unmodified — v1.0
- ✓ DIFF production bug fix: canonical `run_show`/`get_object` helpers enforced across all 9 differentiator tools — v1.0

### Active

- [ ] **SKILL-01**: Skill auto-detects AOS8 connectivity and switches to API mode — no paste required for AOS8 source path
- [ ] **SKILL-02**: Stage 1 collects all 16 AOS8 data points via platform tools (hierarchy, AP database, cluster state, clients, WLAN, etc.)
- [ ] **SKILL-03**: Stage 3 adds live-data-only VSG rules (VRRP VIP validation, ARM profile detection, local user count, cluster L2 health)
- [x] **SKILL-04**: Stage 4 cross-references AOS8 AP inventory against Central firmware recommendations and AP count gap — Validated in Phase 12: central-enrichment-cutover-validation
- [x] **SKILL-05**: Stage 5 validates cutover prerequisites from live AOS8 state (controller firmware, cluster health, AP count) — Validated in Phase 12: central-enrichment-cutover-validation
- [x] **SKILL-06**: Report includes executive summary paragraph before the structured findings — Validated in Phase 13: executive-output-quality-gate
- [x] **SKILL-07**: Structured findings remain clean enough to paste into a customer-facing document — Validated in Phase 13: executive-output-quality-gate
- [x] **SKILL-08**: `tools:` frontmatter and `platforms:` tag updated to include AOS8 tools; skill regression test passes — Validated in Phase 13: executive-output-quality-gate

### Out of Scope

- AOS10 / Aruba Central cloud-managed APs — different API surface, different project
- HPE Aruba Instant (IAP) standalone mode — no Conductor, different auth model
- Replacing or modifying any existing platform module — fork extends, doesn't break
- GUI / web dashboard — pure MCP server, no frontend
- GreenLake write tools — GreenLake remains read-only (inherited constraint)

## Context

**Current state (v1.0):**
- AOS8 platform module: 47 tools (26 read + 9 differentiator + 12 write) + 9 guided prompts
- Version: 2.4.0.1 (pyproject.toml)
- Test suite: 766 unit tests, all platforms green
- ~148K total Python LOC in project; AOS8 module ~3K LOC
- No live AOS8 test system — all tests use mocked HTTP responses

**Forked from:** `github.com/nowireless4u/hpe-networking-mcp` (community project, MIT license)

**AOS8 API characteristics:**
- HTTPS only, default port 4343 (Mobility Conductor and Managed Devices)
- Auth: POST login → `UIDARUBA` session token in cookie
- Only GET and POST HTTP methods supported (no PUT, PATCH, DELETE)
- Config API: `/v1/configuration/object/<object_name>?config_path=<path>`
- Show Command API: `/v1/configuration/showcommand?command=<cli-command>&UIDARUBA=<token>`
- Write Memory API: must be called per config_path after changes to persist config
- SSL: self-signed certs common in enterprise; `verify_ssl` flag needed

**v2 deferred:**
- Cross-platform `site_health_check` integration for AOS8 (no flat "site" concept; design deferred)
- Firmware upgrade orchestration (high blast radius; defer until write tools proven stable)
- PCAP / packet capture workflows (requires unsupported HTTP methods or special extensions)
- Generic config-object CRUD passthrough (footgun — 500+ objects with no schema discovery)
- Bulk AP operations (move AP group, mass reboot)

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
| Follow existing platform module pattern exactly | Consistency — new platform fits seamlessly into server bootstrap, middleware, dynamic mode | ✓ Good — zero friction integrating with existing middleware and dynamic mode |
| config_path defaults to `/md`, overridable | Covers Conductor hierarchy (most common) while supporting targeted per-MD operations | ✓ Good — clean operator UX; config_path semantics documented in INSTRUCTIONS.md |
| Session token reuse (not per-call login) | AOS8 docs explicitly warn against repeated login; prevents session exhaustion on Conductor | ✓ Good — asyncio.Lock prevents race; lazy login prevents startup block |
| Write Memory as explicit tool, not auto-called | Operator intent — AI should call it deliberately, not silently; matches staged-write pattern from Axis | ✓ Good — `pre_change_check` prompt surfaces this contract explicitly |
| Mock-only tests (no live system) | No test hardware available; follow same mock pattern as existing unit tests | ✓ Good — 766 tests run in CI with no infrastructure |
| Phase 4 (Differentiators) merged into Phase 7 | Discovery: differentiator tools could be tested alongside integration work without separate phase | ✓ Good — Phase 7 delivered DIFF tools + integration in one coherent phase |
| Local `_show`/`_object` helpers in differentiators.py (Phase 7) | Frozen test mocks returned dict directly; appeared correct under test | ⚠️ Revisit — was a bug; Phase 8 fixed to use canonical `run_show`/`get_object`; don't repeat this pattern |
| Gap closure phases (08, 09) added post-audit | Milestone audit found DIFF production bug + doc/planning drift; needed correction before tagging | ✓ Good — right call to fix before shipping; kept tech debt list short |

---
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
*Last updated: 2026-04-29 — Phase 12 complete: Central enrichment (ENRICH-01..04) and cutover prerequisite (CUTOVER-01..03) sub-paths inserted into Stages 4 and 5 of aos-migration-readiness.md; SKILL-04 and SKILL-05 validated*
