# Feature Landscape — AOS8 / Mobility Conductor MCP Module

**Domain:** On-premise wireless controller management (Aruba OS 8 / Mobility Conductor)
**Researched:** 2026-04-27
**Goal:** Parity with the existing Aruba Central module (73 tools + 12 prompts) within the constraints of the AOS8 API surface.

---

## Executive Summary

AOS8 differs from Aruba Central in three architectural ways that drive the feature mapping:

1. **No REST CRUD parity.** AOS8 is GET + POST only; "delete" is a `_delete: true` payload on POST, not DELETE. Many Central "manage_*" tools that depend on PUT/PATCH must be re-shaped as POST-with-action.
2. **Show commands are first-class.** Central exposes structured monitoring objects via OData; AOS8's richest data lives behind the show-command passthrough (`/v1/configuration/showcommand`). To reach Central parity we must wrap the most operationally valuable show commands as named tools, not just expose a raw passthrough.
3. **Hierarchical config_path.** Every config object lives at `/md`, `/md/<device>`, or `/md/<ap-group>`. Tools must accept `config_path` as a first-class parameter and default to `/md` for Conductor-wide reads.

The Central module's 73 tools split roughly: ~40 read/monitoring, ~20 write/manage, ~13 actions/troubleshooting/prompts. Aiming for **38–46 AOS8 tools + 8–10 prompts** for v1 hits parity in operator coverage without bloating the surface with low-value config objects.

---

## Central -> AOS8 Tool Category Mapping

| Central Category | Central Tool Count | AOS8 Source | AOS8 Tool Estimate | Notes |
|------------------|-------------------:|-------------|-------------------:|-------|
| Sites | 3 | N/A — AOS8 has no "site" object | 0 | Replaced by `config_path` and AP groups |
| Devices (APs/switches/gateways) | 2 | `show ap database`, `show switches`, `show ap active` | 4 | Controllers + APs only; no switches/gateways here |
| Monitoring (AP/switch/gateway details) | 5 | `show ap bss-table`, `show ap radio-summary`, `show ap arm history` | 5 | One per device sub-view |
| Stats / utilization | 6 | `show ap monitor`, `show datapath`, `show memory`, `show cpuload` | 4 | Controller + AP perf |
| Clients | 2 | `show user-table`, `show user mac <m>`, `show client-trail-info` | 3 | MAC/IP/username lookup |
| Alerts | 1 | `show alarm` (pre-9.x) / `show ap-rules` / SNMP | 1 | Slim — AOS8 alerting is weak vs Central |
| Events / Audit logs | 3 | `show audit-trail`, `show log all`, `show log security` | 3 | Direct equivalents exist |
| WLANs / WLAN profiles | 5 | `wlan_ssid_profile`, `wlan_virtual_ap`, `show wlan ssid-profile` | 4 | Read + manage SSID, virtual-AP, bind to AP-group |
| Roles | 2 | `role`, `user_role`, `show rights` | 2 | user-role read + manage |
| Configuration (groups/sites) | 3 | `ap_group`, `ap_sys_prof` | 3 | AP-group as the config-grouping primitive |
| Config assignments | 2 | `ap_regulatory_domain_prof`, `ap_group` bindings | 1 | Single bind/unbind tool |
| Scope (RBAC, effective config) | 5 | `mgmt-user`, `show running-config` per node | 2 | Slimmer — no full RBAC tree like Central |
| Security policy / ACLs | 12+4 | `acl_sess`, `netdestination`, `netservice`, `firewall_visibility` | 6 | Folded into fewer composite manage tools |
| Server groups (AAA) | 1 | `aaa_server_group`, `aaa_auth_server` | 2 | RADIUS/LDAP servers + groups |
| Named VLANs | 1 | `vlan_name`, `vlan_id` | 1 | Direct mapping |
| Aliases / Applications | 2 | `netdestination` aliases | 1 | Folded into security |
| Firmware | 6 | `show image version`, `show ap image version`, `copy` | 3 | Read-only status v1; upgrade orchestration is risky |
| Troubleshooting (ping/trace/cable/show) | 5 | `ping`, `traceroute`, `show *` passthrough | 5 | Strong AOS8 fit |
| Actions (disconnect/bounce) | 12 | `aaa user delete`, `apboot`, controller reload, `clear datapath` | 5 | Trim to the high-value 5 |
| Write-memory / staged commit | (implicit in Central) | `/v1/configuration/object/write_memory` | 1 | Explicit, AOS8-specific tool |
| Switch PoE / cable test | 2 | N/A in AOS8 (no switch mgmt here) | 0 | Out of scope |
| Prompts (guided workflows) | 12 | — | 8–10 | See "Guided Prompts" below |
| **Total** | **73 + 12** | | **~42 + 9** | |

---

## Table Stakes

Operators expect these from any controller-management tool. Missing any of these makes the module feel half-built.

### Health & Inventory (8 tools)
| Tool | AOS8 Source | Why Expected | Complexity |
|------|-------------|--------------|------------|
| `aos8_get_controllers` | `show switches` show command | "What controllers do I have?" — first question every operator asks | Low |
| `aos8_get_aps` | `show ap database` show command | AP inventory with status, group, model, IP | Low |
| `aos8_get_ap_active` | `show ap active` show command | Only the *up* APs, with uptime and clients | Low |
| `aos8_get_ap_details` | `show ap details ap-name <n>` | Single-AP deep dive: radios, neighbors, BSSIDs | Med |
| `aos8_get_ap_bss_table` | `show ap bss-table` | What BSSIDs/SSIDs are radiating where | Low |
| `aos8_get_ap_radio_summary` | `show ap radio-summary` | Radio state, channel, power, EIRP per AP | Low |
| `aos8_get_controller_version` | `show version` show command | Firmware + uptime — universal triage starting point | Low |
| `aos8_get_licenses` | `show license` show command | Capacity vs entitlement; common ticket cause | Low |

### Clients (4 tools)
| Tool | AOS8 Source | Why Expected | Complexity |
|------|-------------|--------------|------------|
| `aos8_get_clients` | `show user-table` show command | List associated clients with VLAN, role, AP, RSSI | Low |
| `aos8_find_client` | `show user mac <m>` / `show user ip <i>` / `show user name <u>` | The single most-called troubleshooting tool — works across MAC, IP, username | Med |
| `aos8_get_client_history` | `show client-trail-info mac <m>` | Roaming/auth history for a client | Med |
| `aos8_disconnect_client` | `aaa user delete mac <m>` action | Force re-auth — table-stakes troubleshooting verb | Med (write) |

### Troubleshooting (4 tools)
| Tool | AOS8 Source | Why Expected | Complexity |
|------|-------------|--------------|------------|
| `aos8_ping` | `ping <target>` action | Reachability from controller | Low |
| `aos8_traceroute` | `traceroute <target>` action | Path verification | Low |
| `aos8_show_command` | `/v1/configuration/showcommand?command=<cmd>` | Generic passthrough — escape hatch for any show command | Med |
| `aos8_get_logs` | `show log all` / `show log errorlog` / `show log security` | Recent log lines for diagnosis | Med |

### Alerts & Audit (3 tools)
| Tool | AOS8 Source | Why Expected | Complexity |
|------|-------------|--------------|------------|
| `aos8_get_alarms` | `show alarm` show command | Active alarms on the Conductor | Low |
| `aos8_get_audit_trail` | `show audit-trail` | Who changed what — change-tracking baseline | Low |
| `aos8_get_events` | `show log security all` / `show log management` | Security and mgmt event stream | Low |

### WLAN / Config Read (4 tools)
| Tool | AOS8 Source | Why Expected | Complexity |
|------|-------------|--------------|------------|
| `aos8_get_ssid_profiles` | GET `wlan_ssid_profile` config object | Inventory of SSIDs and their security mode | Low |
| `aos8_get_virtual_aps` | GET `wlan_virtual_ap` config object | The bind layer between SSIDs and AP-groups | Low |
| `aos8_get_ap_groups` | GET `ap_group` config object | Group hierarchy + which profiles each carries | Low |
| `aos8_get_user_roles` | GET `user_role` / `show rights` | Role definitions and applied ACLs | Med |

### Stats / Performance (3 tools)
| Tool | AOS8 Source | Why Expected | Complexity |
|------|-------------|--------------|------------|
| `aos8_get_controller_stats` | `show cpuload`, `show memory`, `show datapath utilization` | Controller health snapshot | Med |
| `aos8_get_ap_arm_history` | `show ap arm history` | Channel/power changes — explains "my AP keeps moving channels" | Med |
| `aos8_get_ap_monitor` | `show ap monitor stats` | RF monitor — interferers, rogues | Med |

**Table-stakes total: 26 tools.** This is the floor for operator parity.

---

## Differentiators

AOS8-specific capabilities that go beyond what Central can offer for the same fleet.

### Conductor-Specific (4 tools)
| Tool | AOS8 Source | Differentiator | Complexity |
|------|-------------|----------------|------------|
| `aos8_get_md_hierarchy` | `show switches` + config_path tree | Visualize MD/Conductor relationships — Central has no equivalent | Med |
| `aos8_get_effective_config` | GET running-config at `<config_path>` | See exactly what config a single MD or AP-group sees after inheritance | High |
| `aos8_get_pending_changes` | `show configuration committed-state` | What's staged on Conductor but not pushed to MDs | Med |
| `aos8_write_memory` | POST `write_memory` per config_path | Persist config — uniquely AOS8 step (no auto-save) | Low (write) |

### RF & Cluster (3 tools)
| Tool | AOS8 Source | Differentiator | Complexity |
|------|-------------|----------------|------------|
| `aos8_get_rf_neighbors` | `show ap arm neighbors` | ARM neighbor graph — richer than Central's view | Med |
| `aos8_get_cluster_state` | `show lc-cluster group-membership` | Active/standby AP cluster state — pure AOS8 concept | Med |
| `aos8_get_air_monitors` | `show ap monitor ap-list` | AM-mode AP visibility — air-monitor-specific | Med |

### Wired-Port-on-AP & Tunnels (2 tools)
| Tool | AOS8 Source | Differentiator | Complexity |
|------|-------------|----------------|------------|
| `aos8_get_ap_wired_ports` | GET `ap_wired_ap_prof` + `show ap port status` | Per-AP wired port profile + state — AOS8 strength | Med |
| `aos8_get_ipsec_tunnels` | `show crypto isakmp sa` / `show crypto ipsec sa` | Site-to-site IPsec state — RAP/branch deployments | Med |

**Differentiators total: 9 tools.** These are what justify having an AOS8 module rather than just routing everything through Central.

---

## Write Tools (Gated by `ENABLE_AOS8_WRITE_TOOLS=true`)

All writes require elicitation confirmation per existing platform pattern. AOS8-specific gotcha: every write needs a follow-up `aos8_write_memory` at the appropriate `config_path`, or it dies on next reboot.

| Tool | Action | Risk | AOS8 Source |
|------|--------|------|-------------|
| `aos8_manage_ssid_profile` | Create/update/delete SSID | High — service impact | POST `wlan_ssid_profile` (`_delete:true` for delete) |
| `aos8_manage_virtual_ap` | Bind SSID to AP-group | High | POST `wlan_virtual_ap` |
| `aos8_manage_user_role` | Create/update role | High | POST `user_role` |
| `aos8_manage_ap_group` | Create/update AP-group | High | POST `ap_group` |
| `aos8_manage_named_vlan` | Manage VLAN ID/name | Med | POST `vlan_id` / `vlan_name` |
| `aos8_manage_aaa_server` | Manage RADIUS/LDAP server | High | POST `aaa_auth_server` |
| `aos8_manage_aaa_server_group` | Manage AAA server group | Med | POST `aaa_server_group` |
| `aos8_manage_acl` | Manage session ACL | High | POST `acl_sess` |
| `aos8_manage_netdestination` | Manage net-destination alias | Low | POST `netdestination` |
| `aos8_disconnect_client` | Force client off | Low — reversible | `aaa user delete` |
| `aos8_reboot_ap` | `apboot` an AP | High | `apboot ap-name <n>` action |
| `aos8_write_memory` | Persist staged config | Med — but required | POST `write_memory` |

**Write tools total: 12.** Conservative — the AOS8 config tree has hundreds of objects; only the ones operators routinely touch are wrapped.

---

## Anti-Features

Things to explicitly NOT build in v1.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **Generic config object CRUD** (`aos8_get_object` / `aos8_set_object`) | The AOS8 config tree has 500+ objects with no schema discovery; LLMs will hallucinate field names; trivially easy to brick a controller | Wrap only the ~12 objects with named manage tools |
| **DELETE-style tools** | AOS8 has no DELETE method — all deletes are POST with `_delete:true` payload | Fold delete into the matching `manage_*` tool with an `action="delete"` parameter |
| **Bulk firmware upgrade orchestration** | Multi-stage operation (download + activate + reload) with high blast radius; AOS8 upgrade workflows are deeply version-specific | v1: read-only firmware status (`show image version`). Upgrades stay manual or come in v2 |
| **Switch / gateway management** | AOS8 controllers don't manage AOS-S/CX switches; that's Central's domain | Direct operators to the Central module for switch ops |
| **Site / floor-plan management** | AOS8 has no "site" or floor-plan object — it has AP-groups | Use `config_path` and AP-group tools instead |
| **Topology graph / connectivity-template builder** | These are SDN/datacenter concepts (Apstra) — irrelevant to AOS8 | Skip |
| **Cable test on AP wired port** | AOS8 doesn't expose a structured cable-test API; only ad-hoc CLI | If needed, expose via the generic `aos8_show_command` passthrough |
| **Live PCAP / packet capture orchestration** | Capture writes to controller flash, requires file retrieval, complex cleanup | Out of scope v1 |
| **Auto-`write_memory` after every change** | Operators want explicit control; mass auto-saves on a Conductor with many MDs is dangerous | Make `aos8_write_memory` an explicit deliberate step |
| **API key / OAuth flows** | AOS8 has no OAuth — only username/password + UIDARUBA cookie | Don't try to retrofit; document the auth model honestly |

---

## High-Value Show Commands for Troubleshooting

The `aos8_show_command` passthrough is the universal escape hatch, but the following 18 show commands cover ~90% of real troubleshooting and should be wrapped as named tools (or be the documented first-line `command` value the LLM uses with the passthrough):

| Show Command | Use Case |
|--------------|----------|
| `show version` | Firmware + uptime |
| `show switches` | Conductor + MD inventory and state |
| `show ap database` | All known APs |
| `show ap active` | Currently up APs |
| `show ap bss-table` | What's radiating, where |
| `show ap radio-summary` | Per-radio state |
| `show ap arm history` | Why an AP changed channel/power |
| `show ap arm neighbors` | RF neighborhood |
| `show ap monitor stats` | RF interference / rogues |
| `show ap details ap-name <n>` | Single-AP deep dive |
| `show user-table` | All clients |
| `show user mac <m>` / `ip <i>` / `name <u>` | Single-client lookup |
| `show client-trail-info mac <m>` | Client roam/auth history |
| `show alarm` | Active alarms |
| `show audit-trail` | Recent config changes |
| `show log all` / `show log security` / `show log errorlog` | Log streams |
| `show license` | Entitlement |
| `show running-config` (scoped) | Effective config at a node |
| `show lc-cluster group-membership` | Cluster state |
| `show crypto isakmp sa` / `show crypto ipsec sa` | RAP / IPsec tunnels |

---

## Feature Dependencies

```
aos8_login (UIDARUBA token) -> EVERY other tool
aos8_get_controllers -> aos8_get_md_hierarchy -> aos8_get_effective_config (config_path resolution)
aos8_get_aps -> aos8_get_ap_details / aos8_get_ap_bss_table / aos8_get_ap_radio_summary
aos8_get_ap_groups -> aos8_manage_virtual_ap (need to know group name)
aos8_get_ssid_profiles -> aos8_manage_virtual_ap (need profile name)
aos8_manage_* (any write) -> aos8_write_memory (or change is lost on reload)
aos8_find_client -> aos8_get_client_history / aos8_disconnect_client
```

---

## Guided Prompts (Parity with Central's 12)

Aim for 8–10. Drop the Central prompts that don't apply (switch-specific, site-specific) and add AOS8-specific ones.

| Prompt | Equivalent Central Prompt? | Purpose |
|--------|----------------------------|---------|
| `aos8_triage_client` | yes | "User says Wi-Fi is broken" — MAC -> client trail -> AP -> RF -> role |
| `aos8_triage_ap` | yes | "AP is down" — DB -> active -> log -> ARM history |
| `aos8_audit_change` | yes | "What changed in the last hour?" — audit-trail + log security |
| `aos8_review_ssid` | yes | Full SSID review — profile + virtual-AP + bound AP-groups + role |
| `aos8_health_check` | yes | Conductor + MD + AP fleet + cluster + license rollup |
| `aos8_rf_check` | yes | Per-AP radio + ARM + neighbors + monitor for an AP-group |
| `aos8_rogue_review` | new (AOS8 strength) | Walk `show ap monitor` rogue list with classification |
| `aos8_pre_change_snapshot` | new | Capture running-config + audit-trail + AP/client counts before a change |
| `aos8_write_memory_walkthrough` | new (AOS8-specific) | Identify which `config_path`s have pending changes; confirm + persist |
| `aos8_cluster_status` | new (AOS8-specific) | Cluster group + AP cluster membership + standby controller |

---

## MVP Recommendation

**Phase the implementation as:**

1. **MVP / v0.1 — Read parity (~26 tools, 4 prompts):** All table-stakes read tools above + `aos8_show_command` passthrough + `aos8_triage_client`, `aos8_triage_ap`, `aos8_health_check`, `aos8_audit_change`. Ships the operator value.
2. **v0.2 — Differentiators (+9 tools, +3 prompts):** Conductor hierarchy, effective config, RF/cluster, wired-port-on-AP, IPsec, plus `aos8_rf_check`, `aos8_rogue_review`, `aos8_cluster_status`.
3. **v0.3 — Writes (+12 tools, +2 prompts):** Gated manage_* tools + `aos8_write_memory` + `aos8_pre_change_snapshot`, `aos8_write_memory_walkthrough`. Requires the most testing — ship last.

**Total v1 target: 47 tools + 9 prompts** — comfortably in the 30–50 tool range, materially less surface than Central (73) but covering the same operator workflows because AOS8's domain is narrower (no switches, no gateways, no sites).

**Defer to v2:**
- Firmware upgrade orchestration (high risk, version-specific)
- PCAP / packet-capture workflows (file-handling complexity)
- Generic config-object CRUD (footgun)
- Cross-platform `site_health_check` integration (depends on how AOS8 maps to "site")

---

## Quality Gate Self-Check

- [x] Each tool category maps to a specific AOS8 API endpoint or show command (see table-stakes / differentiators / write tables)
- [x] Table stakes (26) vs differentiators (9) clearly separated
- [x] Write tools (12) identified and flagged with `ENABLE_AOS8_WRITE_TOOLS` requirement and `write_memory` follow-up
- [x] Realistic tool count: **47 tools + 9 prompts** for v1 — within the 30–50 target

---

## Sources & Confidence

| Claim | Source | Confidence |
|-------|--------|------------|
| AOS8 API is GET + POST only; deletes use `_delete:true` | PROJECT.md context block (project-supplied authoritative) | HIGH |
| `/v1/configuration/object/<name>` and `/v1/configuration/showcommand` endpoints | PROJECT.md context block | HIGH |
| `write_memory` required per config_path after writes | PROJECT.md Constraints section | HIGH |
| Central tool inventory (73 tools, by category) | Direct scan of `platforms/central/tools/` (this session) | HIGH |
| Specific show command names (`show ap database`, `show user-table`, `show ap arm history`, etc.) | Aruba OS 8 standard CLI reference (training data — these are stable across 8.x) | MEDIUM — recommend verifying exact syntax against the deployed AOS8 version's CLI guide before implementation |
| Specific config object names (`wlan_ssid_profile`, `wlan_virtual_ap`, `ap_group`, `user_role`, `acl_sess`, `netdestination`, `aaa_server_group`, `vlan_id`) | AOS8 Configuration API reference (training data) | MEDIUM — these are standard objects but field names within each object should be verified against the target controller version's API browser at `https://<conductor>:4343/api` |
| Cluster (`show lc-cluster group-membership`) and IPsec (`show crypto isakmp sa`) commands | AOS8 8.x CLI conventions (training data) | MEDIUM |
| AP-group hierarchy as the AOS8 substitute for Central "sites" | Architectural inference — AOS8 has no first-class site object; AP-groups are the closest grouping primitive | HIGH (architectural fact) |

**Recommended verification before implementation begins:**
- Hit a live AOS8 Conductor's API explorer (`https://<conductor>:4343/api`) and confirm the exact object names + field schemas for the 12 manage_* tools.
- Run each of the 18 high-value show commands once and snapshot the JSON response shape — these are the test fixtures for the unit tests.
- Confirm `write_memory` semantics on the target version: in some 8.x builds it's an action endpoint, in others it's a config-object POST.
