# Requirements — AOS8/Mobility Conductor Platform Module

## v1 Requirements

### FOUND — Platform Foundation
- [x] **FOUND-01**: AOS8Secrets dataclass loads `aos8_host`, `aos8_username`, `aos8_password`, `aos8_port` (default 4343), `aos8_verify_ssl` from Docker secrets via `_read_secret()`
- [x] **FOUND-02**: Platform auto-disables gracefully when any required secret is absent or empty — no server crash
- [x] **FOUND-03**: `ENABLE_AOS8_WRITE_TOOLS` environment variable gates write tool visibility (default false), consistent with all other platforms
- [x] **FOUND-04**: `aos8` registered in `_common/tool_registry.py` REGISTRIES, write-tag sets, and gate-config map
- [x] **FOUND-05**: `docker-compose.yml` updated with 5 AOS8 secrets and `ENABLE_AOS8_WRITE_TOOLS` env var; `.example` template files created for each secret

### CLIENT — API Client
- [x] **CLIENT-01**: `AOS8Client` authenticates via POST login, stores UIDARUBA session token in httpx cookie jar and `self._token`; token is reused across all subsequent calls (not re-login per request)
- [x] **CLIENT-02**: `asyncio.Lock` serializes token refresh — concurrent requests do not trigger parallel logins
- [x] **CLIENT-03**: Login is lazy (deferred to first tool call, not blocking server startup) — unreachable Conductor does not prevent server from starting
- [x] **CLIENT-04**: 401 response triggers exactly one token refresh attempt; subsequent 401 surfaces error to caller without infinite loop
- [x] **CLIENT-05**: All HTTP responses checked for `_global_result.status` — non-zero raises `AOS8APIError`, not silently returns bad data
- [x] **CLIENT-06**: UIDARUBA token is never logged — `mask_secret()` applied; token stripped from URL strings before logging or error formatting
- [x] **CLIENT-07**: `verify_ssl` defaults to `True`; `False` emits a startup WARNING in logs — operator must opt in explicitly
- [x] **CLIENT-08**: Port is configurable via secret (never hardcoded to 4343)
- [x] **CLIENT-09**: `client.health_check()` method used by the server's `health` tool to probe AOS8 alongside other platforms
- [x] **CLIENT-10**: `aclose()` logs out session and closes httpx client on server shutdown

### READ — Health & Inventory Tools (Read)
- [ ] **READ-01**: `aos8_get_controllers` — list all controllers/MDs under the Conductor with status and role
- [ ] **READ-02**: `aos8_get_ap_database` — full AP database with name, MAC, IP, model, status, AP group
- [ ] **READ-03**: `aos8_get_active_aps` — currently active (up) APs with association counts
- [ ] **READ-04**: `aos8_get_ap_detail` — detailed stats for a single AP by name or MAC
- [ ] **READ-05**: `aos8_get_bss_table` — BSS table showing all radio/BSSID associations at a scope
- [ ] **READ-06**: `aos8_get_radio_summary` — per-AP radio state: channel, power, utilization, noise floor
- [ ] **READ-07**: `aos8_get_version` — AOS8 software version on Conductor and MDs
- [ ] **READ-08**: `aos8_get_licenses` — installed licenses and feature entitlements

### READ — Client Tools (Read)
- [ ] **READ-09**: `aos8_get_clients` — list connected wireless clients at a config_path scope
- [ ] **READ-10**: `aos8_find_client` — find client by MAC address, IP address, or username
- [ ] **READ-11**: `aos8_get_client_detail` — detailed association, auth, and connection info for a single client
- [ ] **READ-12**: `aos8_get_client_history` — historical connection events for a client

### READ — Alerts & Audit Tools (Read)
- [ ] **READ-13**: `aos8_get_alarms` — active alarms with severity, category, and timestamp
- [ ] **READ-14**: `aos8_get_audit_trail` — configuration audit log (who changed what, when)
- [ ] **READ-15**: `aos8_get_events` — system event log filtered by type, severity, or time window

### READ — WLAN & Config Tools (Read)
- [ ] **READ-16**: `aos8_get_ssid_profiles` — list all SSID profiles with key settings
- [ ] **READ-17**: `aos8_get_virtual_aps` — virtual AP profiles mapped to SSIDs and AP groups
- [ ] **READ-18**: `aos8_get_ap_groups` — AP group list with member APs and applied profiles
- [ ] **READ-19**: `aos8_get_user_roles` — defined user roles and policy assignments

### READ — Troubleshooting & Stats Tools (Read)
- [ ] **READ-20**: `aos8_ping` — ping from a controller to a target IP, returns RTT and loss
- [ ] **READ-21**: `aos8_traceroute` — traceroute from a controller to a target IP
- [ ] **READ-22**: `aos8_show_command` — passthrough for any AOS8 CLI show command; strips `_meta` field from response; returns structured JSON where available
- [ ] **READ-23**: `aos8_get_logs` — recent system log entries filterable by severity
- [ ] **READ-24**: `aos8_get_controller_stats` — CPU, memory, uptime, and session counts on a controller
- [ ] **READ-25**: `aos8_get_arm_history` — ARM channel-change and power-adjustment history
- [ ] **READ-26**: `aos8_get_rf_monitor` — RF monitor data including interference and rogue detections

### DIFF — AOS8 Differentiator Tools (Read)
- [ ] **DIFF-01**: `aos8_get_md_hierarchy` — Conductor-to-Managed-Device relationship tree showing all MDs and their config_path
- [ ] **DIFF-02**: `aos8_get_effective_config` — resolved configuration a specific MD or AP group sees after inheritance from parent nodes
- [ ] **DIFF-03**: `aos8_get_pending_changes` — staged config changes on the Conductor that have not yet been persisted with write_memory
- [ ] **DIFF-04**: `aos8_get_rf_neighbors` — ARM neighbor graph for an AP showing co-channel and adjacent-channel neighbors
- [ ] **DIFF-05**: `aos8_get_cluster_state` — AP cluster membership, master/standby controller roles, and failover state
- [ ] **DIFF-06**: `aos8_get_air_monitors` — list of APs operating in air-monitor mode with scan results
- [ ] **DIFF-07**: `aos8_get_ap_wired_ports` — wired port configuration and state for APs with wired downlinks
- [ ] **DIFF-08**: `aos8_get_ipsec_tunnels` — site-to-site IPsec and Remote AP tunnel state
- [ ] **DIFF-09**: `aos8_get_md_health_check` — unified health report for a specific MD or config_path scope (equivalent of cross-platform site_health_check for AOS8)

### WRITE — Write Tools (gated behind ENABLE_AOS8_WRITE_TOOLS=true)
- [ ] **WRITE-01**: `aos8_manage_ssid_profile` — create, update, or delete an SSID profile; requires explicit `config_path`; response includes `requires_write_memory_for` field
- [ ] **WRITE-02**: `aos8_manage_virtual_ap` — create, update, or delete a virtual AP profile
- [ ] **WRITE-03**: `aos8_manage_ap_group` — create, update, or delete an AP group
- [ ] **WRITE-04**: `aos8_manage_user_role` — create, update, or delete a user role
- [ ] **WRITE-05**: `aos8_manage_vlan` — create, update, or delete a VLAN definition
- [ ] **WRITE-06**: `aos8_manage_aaa_server` — create, update, or delete an AAA server entry
- [ ] **WRITE-07**: `aos8_manage_aaa_server_group` — create, update, or delete an AAA server group
- [ ] **WRITE-08**: `aos8_manage_acl` — create, update, or delete a session ACL
- [ ] **WRITE-09**: `aos8_manage_netdestination` — create, update, or delete a network destination object
- [ ] **WRITE-10**: `aos8_disconnect_client` — force-disconnect a wireless client by MAC address
- [ ] **WRITE-11**: `aos8_reboot_ap` — reboot a specific AP by name
- [ ] **WRITE-12**: `aos8_write_memory` — persist staged config changes to startup config for a given config_path; this is always an explicit operator action, never called automatically by other write tools

### PROMPT — Guided Prompts
- [ ] **PROMPT-01**: `aos8_triage_client` — multi-step workflow: find client, check AP health, review auth events, identify likely root cause
- [ ] **PROMPT-02**: `aos8_triage_ap` — deep-dive on a specific AP: radio state, clients, alarms, event timeline, next-step recommendations
- [ ] **PROMPT-03**: `aos8_health_check` — network-wide health assessment across all MDs: AP counts, client counts, alarm summary, firmware versions
- [ ] **PROMPT-04**: `aos8_audit_change` — review recent audit trail entries, summarize config changes, flag high-risk modifications
- [ ] **PROMPT-05**: `aos8_rf_analysis` — RF environment report for a scope: channel distribution, co-channel clusters, ARM action history, recommendations
- [ ] **PROMPT-06**: `aos8_wlan_review` — inventory all SSID profiles and virtual APs, flag mismatches or inconsistencies across AP groups
- [ ] **PROMPT-07**: `aos8_client_flood` — investigate high client counts or failed connections at a scope; identify affected APs and likely cause
- [ ] **PROMPT-08**: `aos8_compare_md_config` — side-by-side comparison of effective config between two MDs or AP groups
- [ ] **PROMPT-09**: `aos8_pre_change_check` — pre-maintenance checklist: current health, pending changes, confirm write_memory state before proceeding

### TEST — Testing
- [ ] **TEST-01**: Unit tests for `AOS8Client` using mocked httpx responses — cover: login success, login failure, token reuse, 401 refresh, `_global_result` error detection, `verify_ssl` behavior, token masking in logs
- [ ] **TEST-02**: Unit test per read tool category (health, clients, alerts, WLAN, troubleshoot) using fixture JSON responses
- [ ] **TEST-03**: Unit tests for write tools — confirm `aos8_write` tag on all write tools; confirm `aos8_write_delete` on delete operations; confirm `config_path` required (no default) on all writes
- [ ] **TEST-04**: Unit test that UIDARUBA token never appears in any log output across client and tool tests
- [ ] **TEST-05**: Unit test that platform auto-disables cleanly when secrets are absent
- [ ] **TEST-06**: Existing platform tests (Mist, Central, GreenLake, ClearPass, Apstra, Axis) all pass without modification

### DOCS — Documentation
- [ ] **DOCS-01**: `INSTRUCTIONS.md` updated with AOS8 section covering config_path semantics, write_memory contract, show_command usage, and Conductor vs standalone behavior
- [ ] **DOCS-02**: `README.md` updated — AOS8 row in capability table, AOS8 secrets reference section, platform auto-disable example updated, tool count updated
- [ ] **DOCS-03**: `CHANGELOG.md` updated with new version entry
- [ ] **DOCS-04**: `docs/TOOLS.md` updated with AOS8 tool reference
- [ ] **DOCS-05**: `pyproject.toml` version bumped

---

## v2 Requirements (Deferred)

- Cross-platform `site_health_check` integration for AOS8 (AOS8 has no flat "site" concept; design deferred)
- Firmware upgrade orchestration (high blast radius; defer until write tools are proven stable)
- PCAP / packet capture workflows (requires unsupported HTTP methods or special AOS8 API extensions)
- Generic config-object CRUD passthrough (footgun — 500+ objects with no schema discovery)
- Bulk AP operations (move AP group, mass reboot)

---

## Out of Scope

- AOS10 / Aruba Central cloud-managed APs — different API, different project
- HPE Aruba Instant (IAP) standalone mode — different auth model, no Conductor
- Modifying any existing platform module — AOS8 is additive only
- GUI or web dashboard — pure MCP server
- GreenLake write tools — inherited read-only constraint from upstream project

---

## Traceability

| REQ-ID | Phase | Status |
|--------|-------|--------|
| FOUND-01 | Phase 1 — Platform Foundation | Complete |
| FOUND-02 | Phase 1 — Platform Foundation | Complete |
| FOUND-03 | Phase 1 — Platform Foundation | Complete |
| FOUND-04 | Phase 1 — Platform Foundation | Complete |
| FOUND-05 | Phase 1 — Platform Foundation | Complete |
| CLIENT-01 | Phase 2 — API Client | Complete |
| CLIENT-02 | Phase 2 — API Client | Complete |
| CLIENT-03 | Phase 2 — API Client | Complete |
| CLIENT-04 | Phase 2 — API Client | Complete |
| CLIENT-05 | Phase 2 — API Client | Complete |
| CLIENT-06 | Phase 2 — API Client | Complete |
| CLIENT-07 | Phase 2 — API Client | Complete |
| CLIENT-08 | Phase 2 — API Client | Complete |
| CLIENT-09 | Phase 2 — API Client | Complete |
| CLIENT-10 | Phase 2 — API Client | Complete |
| READ-01 | Phase 3 — Read Tools | Pending |
| READ-02 | Phase 3 — Read Tools | Pending |
| READ-03 | Phase 3 — Read Tools | Pending |
| READ-04 | Phase 3 — Read Tools | Pending |
| READ-05 | Phase 3 — Read Tools | Pending |
| READ-06 | Phase 3 — Read Tools | Pending |
| READ-07 | Phase 3 — Read Tools | Pending |
| READ-08 | Phase 3 — Read Tools | Pending |
| READ-09 | Phase 3 — Read Tools | Pending |
| READ-10 | Phase 3 — Read Tools | Pending |
| READ-11 | Phase 3 — Read Tools | Pending |
| READ-12 | Phase 3 — Read Tools | Pending |
| READ-13 | Phase 3 — Read Tools | Pending |
| READ-14 | Phase 3 — Read Tools | Pending |
| READ-15 | Phase 3 — Read Tools | Pending |
| READ-16 | Phase 3 — Read Tools | Pending |
| READ-17 | Phase 3 — Read Tools | Pending |
| READ-18 | Phase 3 — Read Tools | Pending |
| READ-19 | Phase 3 — Read Tools | Pending |
| READ-20 | Phase 3 — Read Tools | Pending |
| READ-21 | Phase 3 — Read Tools | Pending |
| READ-22 | Phase 3 — Read Tools | Pending |
| READ-23 | Phase 3 — Read Tools | Pending |
| READ-24 | Phase 3 — Read Tools | Pending |
| READ-25 | Phase 3 — Read Tools | Pending |
| READ-26 | Phase 3 — Read Tools | Pending |
| DIFF-01 | Phase 4 — Differentiator Tools | Pending |
| DIFF-02 | Phase 4 — Differentiator Tools | Pending |
| DIFF-03 | Phase 4 — Differentiator Tools | Pending |
| DIFF-04 | Phase 4 — Differentiator Tools | Pending |
| DIFF-05 | Phase 4 — Differentiator Tools | Pending |
| DIFF-06 | Phase 4 — Differentiator Tools | Pending |
| DIFF-07 | Phase 4 — Differentiator Tools | Pending |
| DIFF-08 | Phase 4 — Differentiator Tools | Pending |
| DIFF-09 | Phase 4 — Differentiator Tools | Pending |
| WRITE-01 | Phase 5 — Write Tools | Pending |
| WRITE-02 | Phase 5 — Write Tools | Pending |
| WRITE-03 | Phase 5 — Write Tools | Pending |
| WRITE-04 | Phase 5 — Write Tools | Pending |
| WRITE-05 | Phase 5 — Write Tools | Pending |
| WRITE-06 | Phase 5 — Write Tools | Pending |
| WRITE-07 | Phase 5 — Write Tools | Pending |
| WRITE-08 | Phase 5 — Write Tools | Pending |
| WRITE-09 | Phase 5 — Write Tools | Pending |
| WRITE-10 | Phase 5 — Write Tools | Pending |
| WRITE-11 | Phase 5 — Write Tools | Pending |
| WRITE-12 | Phase 5 — Write Tools | Pending |
| PROMPT-01 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-02 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-03 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-04 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-05 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-06 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-07 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-08 | Phase 6 — Guided Prompts & Documentation | Pending |
| PROMPT-09 | Phase 6 — Guided Prompts & Documentation | Pending |
| DOCS-01 | Phase 6 — Guided Prompts & Documentation | Pending |
| DOCS-02 | Phase 6 — Guided Prompts & Documentation | Pending |
| DOCS-03 | Phase 6 — Guided Prompts & Documentation | Pending |
| DOCS-04 | Phase 6 — Guided Prompts & Documentation | Pending |
| DOCS-05 | Phase 6 — Guided Prompts & Documentation | Pending |
| TEST-01 | Phase 7 — Testing & Integration | Pending |
| TEST-02 | Phase 7 — Testing & Integration | Pending |
| TEST-03 | Phase 7 — Testing & Integration | Pending |
| TEST-04 | Phase 7 — Testing & Integration | Pending |
| TEST-05 | Phase 7 — Testing & Integration | Pending |
| TEST-06 | Phase 7 — Testing & Integration | Pending |

**Coverage:** 71 / 71 v1 requirements mapped, 0 orphans, 0 duplicates.
