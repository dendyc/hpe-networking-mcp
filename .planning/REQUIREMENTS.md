# Requirements — v1.1 AOS8-Powered Migration Readiness

## Milestone Goal

Enhance the `aos-migration-readiness` skill to replace paste-driven CLI data collection with live AOS8 API calls and produce a professional-grade, copy-pasteable migration assessment for field SEs and network operators.

---

## Active Requirements

### Detection

- [x] **DETECT-01**: Skill detects AOS8 connectivity at session start and announces API mode (live data) vs paste-mode fallback to operator

### Collection (Stage 1)

- [x] **COLLECT-01**: Skill retrieves MD hierarchy + per-node effective config via `aos8_get_md_hierarchy()` + `aos8_get_effective_config()` — replaces `show configuration node-hierarchy` + `show configuration effective detail` paste
- [x] **COLLECT-02**: Skill retrieves full AP inventory (model, MAC, serial, IP mode, AP group) via `aos8_get_ap_database()` — replaces `show ap database long` paste
- [x] **COLLECT-03**: Skill retrieves cluster state, running-config, and local-user db via `aos8_get_cluster_state()` + `aos8_show_command()` passthrough — replaces `show lc-cluster group-membership`, `show running-config`, `show local-user db`, `show inventory`, `show port status`, `show trunk` pastes
- [x] **COLLECT-04**: Skill retrieves client baseline, BSS/SSID table, active AP RF state, and AP wired ports via `aos8_get_clients()` / `aos8_get_bss_table()` / `aos8_get_active_aps()` / `aos8_get_ap_wired_ports()` — replaces `show ap association`, `show user-table`, `show ap essid`, `show ap active`, `show ap lldp neighbors` pastes

### Live Rules (Stage 3)

- [x] **RULES-01**: VRRP VIP check — skill parses effective config (via API) to verify AP system profiles use the VRRP virtual IP for LMS, not individual controller IPs; auto-flags as REGRESSION if not (VSG §1654-§1657)
- [x] **RULES-02**: ARM / radio profile detection — skill detects active ARM profiles, dot11a/g radio profiles, and regulatory domain profiles from AP-group objects + effective config via API; auto-flags as DRIFT (VSG §1163-§1166)
- [x] **RULES-03**: Local user count cross-check — skill calls `aos8_show_command("show local-user db")` to get actual count, compares against `clearpass_get_local_users()` count; flags dual-source-of-truth as DRIFT automatically
- [x] **RULES-04**: Static AP IP detection — skill parses IP mode field from `aos8_get_ap_database()` response to automatically detect statically addressed APs; flags as REGRESSION (VSG §1232-§1234)

### Central Enrichment (Stage 4)

- [x] **ENRICH-01**: AP count gap — skill compares AOS8 `aos8_get_ap_database()` total against `central_get_aps()` total; reports exact counts (X source APs, Y in Central, Z not yet onboarded)
- [x] **ENRICH-02**: Model → firmware recommendation — skill extracts distinct AP models from AOS8 inventory, calls `central_recommend_firmware()` per model, surfaces recommended AOS10 firmware version in the report
- [x] **ENRICH-03**: SSID conflict detection — skill compares AOS8 SSID names (from `aos8_get_bss_table()` / WLAN objects) against existing Central WLAN profiles via `central_get_wlan_profiles()`; flags naming collisions
- [x] **ENRICH-04**: Role/VLAN conflict detection — skill extracts role names and VLAN IDs from AOS8 effective config, cross-checks against `central_get_roles()` and `central_get_named_vlans()`; flags conflicts

### Cutover Readiness (Stage 5)

- [x] **CUTOVER-01**: Live cluster health — skill calls `aos8_get_cluster_state()` to confirm L2-connected status before single-controller upgrade; flags unhealthy cluster as REGRESSION
- [x] **CUTOVER-02**: Controller firmware version check — skill parses controller firmware version from AOS8 API against the 8.10.0.12 / 8.12.0.1 prerequisite; auto-flags REGRESSION if below (VSG §1643-§1649)
- [x] **CUTOVER-03**: AP count snapshot — skill retrieves live AP count from `aos8_get_ap_database()` for use in pre/during-cutover planning and post-cutover diff

### Output Format

- [x] **OUTPUT-01**: Report includes an executive summary paragraph at the top: GO/BLOCKED/PARTIAL verdict, count of REGRESSIONs/DRIFTs/INFOs, and one-sentence context suitable for an SE to share directly with a customer
- [x] **OUTPUT-02**: Structured REGRESSION/DRIFT/INFO findings formatted with clean markdown (no raw JSON, no tool call artifacts) suitable for direct paste into a customer-facing document

### Quality

- [x] **QUALITY-01**: `tools:` frontmatter in skill file updated to include all AOS8 tool names referenced in the skill body
- [x] **QUALITY-02**: `platforms:` tag updated to include `aos8` alongside `central`
- [x] **QUALITY-03**: `tests/unit/test_skill_tool_references.py` passes with all new AOS8 tool name references resolving to real catalog entries

---

## Future Requirements

*(Deferred — not in scope for v1.1)*

- AOS6 source path API automation (no API client for AOS6)
- IAP source path API automation (no API client for IAP)
- PII redaction / tokenization for paste-mode paths
- VALID8-equivalent automated discovery (channel-partner scope)
- Bulk AP model firmware pre-staging workflow

---

## Out of Scope

- AOS6 and IAP source paths remain paste-driven — no API client exists for those platforms
- Python code changes — this milestone is skill (markdown) only; no new tools or API endpoints
- Removing the PoC label — kept project-wide pending broader production review
- GUI or reporting dashboard — pure MCP skill output

---

## Traceability

| REQ-ID | Phase |
|--------|-------|
| DETECT-01 | Phase 10 |
| COLLECT-01 | Phase 10 |
| COLLECT-02 | Phase 10 |
| COLLECT-03 | Phase 10 |
| COLLECT-04 | Phase 10 |
| RULES-01 | Phase 11 |
| RULES-02 | Phase 11 |
| RULES-03 | Phase 11 |
| RULES-04 | Phase 11 |
| ENRICH-01 | Phase 12 |
| ENRICH-02 | Phase 12 |
| ENRICH-03 | Phase 12 |
| ENRICH-04 | Phase 12 |
| CUTOVER-01 | Phase 12 |
| CUTOVER-02 | Phase 12 |
| CUTOVER-03 | Phase 12 |
| OUTPUT-01 | Phase 13 |
| OUTPUT-02 | Phase 13 |
| QUALITY-01 | Phase 13 |
| QUALITY-02 | Phase 13 |
| QUALITY-03 | Phase 13 |
