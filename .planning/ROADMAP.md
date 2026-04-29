# Roadmap — AOS8/Mobility Conductor Platform Module

## Milestones

- ✅ **v1.0 AOS8 MVP** — Phases 1-9 (shipped 2026-04-29) — [archive](milestones/v1.0-ROADMAP.md)
- 🚧 **v1.1 AOS8-Powered Migration Readiness** — Phases 10-13 (in progress)

## Phases

<details>
<summary>✅ v1.0 AOS8 MVP (Phases 1-9) — SHIPPED 2026-04-29</summary>

- [x] Phase 1: Platform Foundation (4/4 plans) — completed 2026-04-27
- [x] Phase 2: API Client (3/3 plans) — completed 2026-04-28
- [x] Phase 3: Read Tools (7/7 plans) — completed 2026-04-28
- [x] Phase 4: Differentiator Tools (merged into Phase 7) — completed 2026-04-29
- [x] Phase 5: Write Tools (3/3 plans) — completed 2026-04-28
- [x] Phase 6: Guided Prompts & Documentation (2/2 plans) — completed 2026-04-28
- [x] Phase 7: Testing & Integration (3/3 plans) — completed 2026-04-29
- [x] Phase 8: Fix DIFF Tools Production Bug (1/1 plan) — completed 2026-04-28
- [x] Phase 9: Phase 4 Closure & Documentation Accuracy (3/3 plans) — completed 2026-04-28

</details>

### v1.1 AOS8-Powered Migration Readiness

- [ ] **Phase 10: Live Detection & Collection** — Skill auto-detects AOS8 and pulls all Stage 1 source data via API
- [ ] **Phase 11: Live VSG Rules** — Stage 3 evaluates VRRP/ARM/local-user/static-AP rules from live data
- [ ] **Phase 12: Central Enrichment & Cutover Validation** — Stage 4 cross-references Central + Stage 5 validates live cutover prerequisites
- [ ] **Phase 13: Executive Output & Quality Gate** — Customer-grade report format + frontmatter/regression test green

## Phase Details

### Phase 10: Live Detection & Collection
**Goal**: When the operator runs the migration-readiness skill against an AOS8/Conductor source, the skill auto-detects connectivity and gathers every Stage 1 data point through live API tools — no CLI paste required.
**Depends on**: v1.0 AOS8 platform (Phases 1-9, shipped)
**Requirements**: DETECT-01, COLLECT-01, COLLECT-02, COLLECT-03, COLLECT-04
**Success Criteria** (what must be TRUE):
  1. Skill announces "AOS8 API mode — live data" at session start when AOS8 platform is reachable, or falls back to paste mode when not
  2. Skill retrieves MD hierarchy and per-node effective config via `aos8_get_md_hierarchy()` + `aos8_get_effective_config()` (replacing the `show configuration node-hierarchy` / `show configuration effective detail` paste)
  3. Skill retrieves the full AP inventory (model, MAC, serial, IP mode, AP group) via `aos8_get_ap_database()` (replacing `show ap database long`)
  4. Skill retrieves cluster state, running-config, and local-user db via `aos8_get_cluster_state()` + `aos8_show_command()` passthrough (replacing the cluster/running-config/local-user/inventory/port/trunk pastes)
  5. Skill retrieves client baseline, BSS/SSID table, active AP RF state, and AP wired ports via `aos8_get_clients()` / `aos8_get_bss_table()` / `aos8_get_active_aps()` / `aos8_get_ap_wired_ports()` (replacing the association/user-table/essid/active/lldp pastes)
  6. AOS6 and IAP paste paths still load and execute unchanged — no regression in non-AOS8 source paths
**Plans**: TBD

### Phase 11: Live VSG Rules
**Goal**: Stage 3 of the skill applies VSG-anchored rules (VRRP VIP, ARM/radio profiles, local-user count, static AP IPs) directly from live AOS8 data, auto-flagging REGRESSION/DRIFT findings without operator paste.
**Depends on**: Phase 10
**Requirements**: RULES-01, RULES-02, RULES-03, RULES-04
**Success Criteria** (what must be TRUE):
  1. Skill flags a REGRESSION when AP system profiles use individual controller IPs instead of the VRRP VIP for LMS (VSG §1654-§1657)
  2. Skill flags a DRIFT listing every active ARM, dot11a/g radio, and regulatory-domain profile detected per AP-group from live config (VSG §1163-§1166)
  3. Skill cross-checks AOS8 `show local-user db` count against ClearPass `clearpass_get_local_users()` count and flags a DRIFT when both are non-zero
  4. Skill flags a REGRESSION listing every AP whose IP mode in `aos8_get_ap_database()` is static (VSG §1232-§1234)
  5. Each rule cites its VSG section reference in the finding text and identifies the source tool call that produced the evidence
**Plans**: TBD

### Phase 12: Central Enrichment & Cutover Validation
**Goal**: Stage 4 cross-references AOS8 inventory against the destination Central tenant, and Stage 5 validates live cutover prerequisites — together producing the data the SE needs to plan the actual move.
**Depends on**: Phase 10
**Requirements**: ENRICH-01, ENRICH-02, ENRICH-03, ENRICH-04, CUTOVER-01, CUTOVER-02, CUTOVER-03
**Success Criteria** (what must be TRUE):
  1. Skill reports exact AP-count gap (X source APs, Y in Central, Z not yet onboarded) using AOS8 + Central tools
  2. Skill emits a per-AP-model recommended AOS10 firmware version table by calling `central_recommend_firmware()` for each distinct model from AOS8 inventory
  3. Skill flags every SSID name and every role/VLAN ID that already exists in Central (via `central_get_wlan_profiles()` / `central_get_roles()` / `central_get_named_vlans()`) as a naming-conflict finding
  4. Skill flags an unhealthy AOS8 cluster (anything other than L2-connected from `aos8_get_cluster_state()`) as a REGRESSION blocking single-controller upgrade
  5. Skill flags controllers below the 8.10.0.12 / 8.12.0.1 firmware floor as REGRESSION (VSG §1643-§1649) using the live controller version from AOS8
  6. Skill captures a live AP-count snapshot from `aos8_get_ap_database()` and surfaces it as the pre-cutover baseline for post-cutover diffing
**Plans**: TBD

### Phase 13: Executive Output & Quality Gate
**Goal**: The skill renders a customer-grade report (executive summary + clean structured findings) and ships with frontmatter + regression test that prove every referenced AOS8 tool is real.
**Depends on**: Phase 11, Phase 12
**Requirements**: OUTPUT-01, OUTPUT-02, QUALITY-01, QUALITY-02, QUALITY-03
**Success Criteria** (what must be TRUE):
  1. Report opens with a one-paragraph executive summary stating GO / BLOCKED / PARTIAL, total counts of REGRESSION / DRIFT / INFO findings, and one sentence of context an SE can paste verbatim into a customer email
  2. Structured findings render as clean markdown — no raw JSON blobs, no tool-call artifacts, no orphan stack traces — and paste cleanly into a customer-facing document
  3. Skill frontmatter `tools:` list includes every AOS8 tool name referenced in the skill body, and the `platforms:` tag includes `aos8` alongside `central`
  4. `tests/unit/test_skill_tool_references.py` passes with the updated skill, validating every referenced tool against the live catalog
**Plans**: TBD

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Platform Foundation | v1.0 | 4/4 | Complete | 2026-04-27 |
| 2. API Client | v1.0 | 3/3 | Complete | 2026-04-28 |
| 3. Read Tools | v1.0 | 7/7 | Complete | 2026-04-28 |
| 4. Differentiator Tools | v1.0 | merged | Complete | 2026-04-29 |
| 5. Write Tools | v1.0 | 3/3 | Complete | 2026-04-28 |
| 6. Guided Prompts & Documentation | v1.0 | 2/2 | Complete | 2026-04-28 |
| 7. Testing & Integration | v1.0 | 3/3 | Complete | 2026-04-29 |
| 8. Fix DIFF Tools Production Bug | v1.0 | 1/1 | Complete | 2026-04-28 |
| 9. Phase 4 Closure & Documentation Accuracy | v1.0 | 3/3 | Complete | 2026-04-28 |
| 10. Live Detection & Collection | v1.1 | 0/0 | Not started | — |
| 11. Live VSG Rules | v1.1 | 0/0 | Not started | — |
| 12. Central Enrichment & Cutover Validation | v1.1 | 0/0 | Not started | — |
| 13. Executive Output & Quality Gate | v1.1 | 0/0 | Not started | — |

---
*Archive: [v1.0 roadmap details](milestones/v1.0-ROADMAP.md)*
