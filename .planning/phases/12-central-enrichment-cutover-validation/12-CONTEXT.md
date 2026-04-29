# Phase 12: Central Enrichment & Cutover Validation - Context

**Gathered:** 2026-04-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 12 modifies `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` (markdown only — no Python code changes) so that:

1. **Stage 4** gains an AOS8 live-mode sub-path that cross-references AOS8 inventory against the destination Central tenant — AP count gap, per-model firmware recommendations, and SSID/role/VLAN conflict detection — without operator paste.
2. **Stage 5** gains an AOS8 live-mode sub-path that validates live cutover prerequisites (cluster health, controller firmware floor, AP count baseline) from AOS8 API — removing the current "Operator confirms via paste" requirement for these checks.

Both sub-paths follow the same conditional pattern established in Phases 10–11:
- If AOS8 live mode: evaluate from live data / tool calls
- Existing paste-mode tables remain intact as fallback

**Requirements in scope:** ENRICH-01, ENRICH-02, ENRICH-03, ENRICH-04, CUTOVER-01, CUTOVER-02, CUTOVER-03

</domain>

<decisions>
## Implementation Decisions

### Stage 4 Sub-Path Structure (ENRICH-01..04)

- **D-01:** Insert a self-contained `##### AOS8 live-mode sub-path — Central enrichment` section **above** the existing A1–A13 table in Stage 4. The existing table stays intact as the paste fallback for non-live paths. Consistent with Stage 3's live-mode sub-path placement.
- **D-02:** The sub-path covers all four ENRICH requirements: ENRICH-01 (AP count gap), ENRICH-02 (per-model firmware recs), ENRICH-03 (SSID conflicts), ENRICH-04 (role/VLAN conflicts).
- **D-03:** The live-mode sub-path block explicitly notes "no re-fetching" — all AOS8 data is already in Stage 1 context (Batch 2 for AP inventory, Batch 1 effective config for roles/VLANs, Batch 4 bss_table for SSID names).

### ENRICH-01: AP Count Gap

- **D-04:** Compare `aos8_get_ap_database()` total (from Stage 1 Batch 2 context) against `central_get_aps()` total. Emit: "Source AP count: X. Central onboarded: Y. Gap: Z not yet onboarded." (INFO severity — not a blocker, surfaces the migration scope.)

### ENRICH-02: Model → Firmware Recommendation

- **D-05:** Call `central_recommend_firmware()` **once with no filter**. Central returns a fleet-wide recommendation list already grouped by model. The AI extracts per-model rows by cross-referencing against the distinct AP models from AOS8 Batch 2 inventory. One call for any fleet size — no per-model iteration needed.

### ENRICH-03 + ENRICH-04: Conflict Detection

- **D-06:** Check **all three** conflict surfaces: SSID names (A7 via `central_get_wlan_profiles()`), role names (A8 via `central_get_roles()`), named VLAN IDs (A9 via `central_get_named_vlans()`). Cross-reference against:
  - SSID names: from `aos8_get_bss_table()` Batch 4
  - Role names + VLAN IDs: from `aos8_get_effective_config()` Batch 1
- **D-07:** Naming conflicts are **REGRESSION** severity — a conflicting SSID or role name in Central blocks a clean migration. This upgrades from the current INFO severity on A7/A8/A9.
- **D-08:** Each conflict is **listed individually** (one finding per conflicting item), matching the RULES-04 pattern of listing affected AP names. E.g.: `REGRESSION — SSID "Corp-WiFi" already exists in Central (profile ID: X). (source: central_get_wlan_profiles())`

### Stage 5 Sub-Path Structure (CUTOVER-01..03)

- **D-09:** Insert a self-contained `##### AOS8 live-mode sub-path — cutover prerequisites` section **before** the Phase 0–8 cutover table in Stage 5. The existing table stays intact for paste paths. Mirrors Stage 3 and Stage 4 sub-path pattern.
- **D-10:** The sub-path covers CUTOVER-01 (cluster health), CUTOVER-02 (firmware floor), CUTOVER-03 (AP count snapshot) — evaluated once, before the operator walks through the phase table.

### CUTOVER-01: Live Cluster Health

- **D-11:** Call `aos8_get_cluster_state()` (already in Stage 1 Batch 3 context — no re-fetch). Flag **REGRESSION** if anything other than L2-connected. E.g.: `REGRESSION — Cluster not L2-connected: <state>. Resolve before single-controller upgrade. (source: aos8_get_cluster_state(), Batch 3)`

### CUTOVER-02: Controller Firmware Floor

- **D-12:** Call `aos8_show_command(command='show version')` **fresh in the Stage 5 sub-path** (not from Stage 1 context — firmware version is not reliably available from `show inventory`). Parse controller firmware version; flag **REGRESSION** if below 8.10.0.12 / 8.12.0.1 (VSG §1643-§1649). E.g.: `REGRESSION — Controller firmware 8.8.0.5 is below the 8.10.0.12 / 8.12.0.1 prerequisite. (source: aos8_show_command(command='show version'), VSG §1643-§1649)`
- **D-13:** `aos8_show_command` is already in the skill frontmatter `tools:` list (added Phase 10) — no new frontmatter addition required.

### CUTOVER-03: AP Count Baseline

- **D-14:** Surface the live AP count from `aos8_get_ap_database()` Batch 2 context as an **INFO finding** in the Stage 5 sub-path: `INFO — Pre-cutover AP baseline: X APs. (source: aos8_get_ap_database(), Batch 2) Use this count for post-cutover diff.` Lives alongside cluster health and firmware check in the Stage 5 block.

### Claude's Discretion

- Exact wording for the Stage 4 and Stage 5 sub-path header and intro lines
- Whether the Stage 4 sub-path emits a brief "enrichment complete" summary before the live-mode block ends
- How to handle partial failures in Stage 4 (e.g., `central_get_wlan_profiles()` fails) — reuse D-07/D-08 pattern from Phase 10 (exact command or tool name only, no table reprint)
- Format of the per-model firmware recommendation table produced by ENRICH-02

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Skill file being modified
- `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` — Read the full file, focusing on: Stage 4 (lines ~360–379) for the A1–A13 table structure; Stage 5 (lines ~380–395) for the Phase 0–8 cutover table; Stage 3 live-mode sub-path (lines ~232–262) as the pattern to replicate in Stages 4 and 5.

### Prior phase contexts (live-mode sub-path patterns)
- `.planning/phases/10-live-detection-collection/10-CONTEXT.md` — D-04..D-09 define the two-sub-path structure and partial failure fallback policy. Stage 4/5 sub-paths follow the same fallback pattern.
- `.planning/phases/11-live-vsg-rules/11-CONTEXT.md` — D-01..D-05 define the Stage 3 live-mode sub-path structure: conditional header, runs before universal tables, per-rule field-level guidance. Phase 12 replicates this for Stages 4 and 5.

### Central tools (verify signatures before referencing in skill prose)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/firmware.py` — `central_recommend_firmware()` — accepts optional `serial_number`, `device_type`, `site_id`, `site_name`, `include_up_to_date`; call with no filter for fleet-wide model list.
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/monitoring.py` — `central_get_aps()` — returns onboarded AP count; use for ENRICH-01 gap comparison.
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/wlan_profiles.py` — `central_get_wlan_profiles()` — returns existing WLAN/SSID profiles; use for ENRICH-03 SSID conflict check.
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/roles.py` — `central_get_roles()` — returns existing Central roles; use for ENRICH-04 role conflict check.
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/named_vlans.py` — `central_get_named_vlans()` — returns existing named VLANs; use for ENRICH-04 VLAN conflict check.

### AOS8 tools (Stage 1 data already in context — no re-fetch for most)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — `aos8_get_bss_table()` (Batch 4) provides SSID names for ENRICH-03; `aos8_get_ap_database()` (Batch 2) provides AP count and distinct models for ENRICH-01/02.
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` — `aos8_get_effective_config()` (Batch 1) provides role names and VLAN IDs for ENRICH-04; `aos8_get_cluster_state()` (Batch 3) provides cluster L2-connected state for CUTOVER-01.
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py` — `aos8_show_command()` — fresh call with `command='show version'` needed for CUTOVER-02 firmware floor check.

### Requirements
- `.planning/REQUIREMENTS.md` §ENRICH-01..04, §CUTOVER-01..03 — authoritative acceptance criteria for Phase 12.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Stage 3 `##### AOS8 live-mode sub-path` block (lines ~232–262 in skill) — exact structural model for Stage 4 and Stage 5 sub-paths. Conditional header, sequential rule evaluation from Stage 1 context, per-rule fallback note if batch failed, VSG anchor citations in finding text with source tool call in parentheses.
- Stage 2 skip clause — the "if AOS8 live mode, skip paste parsing" pattern; Stage 4 sub-path similarly front-runs the A1–A13 table.
- `tools:` frontmatter (line 23) — already includes `central_recommend_firmware`, `central_get_aps`, `central_get_wlan_profiles`, `central_get_roles`, `central_get_named_vlans`, `aos8_show_command`. No new frontmatter additions required for Phase 12.

### Established Patterns
- Finding format: `**SEVERITY — Description. (source: tool_name(), Batch N) (VSG §anchor)**`
- Sub-path block structure: conditional header line → brief intro → per-check prose → fallback note if data unavailable
- Partial failure fallback: "If [Batch N] was unavailable: [exact CLI command] required — [rule] cannot be evaluated from live data."
- One finding per conflicting item (matches RULES-04 per-AP list style from Phase 11).

### Integration Points
- Stage 3 → Stage 4 handoff: RULES-03 finding is "pending Stage 4 A11" — the A11 row is in the existing table. Phase 12's Stage 4 live-mode sub-path precedes A11; the RULES-03 result should already be in context when A11 executes (no ordering conflict).
- Stage 4 live-mode block feeds Stage 5: the ENRICH findings (REGRESSIONs from conflict detection) are part of the "all REGRESSIONs from stages 3–4 must be resolved" prerequisite in Phase 0 of the cutover table.
- No Python code changes — markdown-only per REQUIREMENTS.md Out of Scope.

</code_context>

<specifics>
## Specific Ideas

- Stage 4 sub-path header should follow the Phase 11 pattern: `##### AOS8 live-mode sub-path — Central enrichment (ENRICH-01..04)`
- Stage 5 sub-path header: `##### AOS8 live-mode sub-path — cutover prerequisites (CUTOVER-01..03)`
- ENRICH-01 output phrasing: "Source AP count: X. Central onboarded: Y. Gap: Z not yet onboarded."
- ENRICH-02 output: a compact per-model table (AP Model | Recommended AOS10 Firmware Version) extracted from `central_recommend_firmware()` response
- ENRICH-03/04 REGRESSION format mirrors RULES-04: one bullet per conflict with the conflicting name/ID and the Central tool that surfaced it
- CUTOVER-03 INFO phrasing: "Pre-cutover AP baseline: X APs. (source: `aos8_get_ap_database()`, Batch 2) Use this count for post-cutover diff."
- CUTOVER-02 `show version` call: fresh call in Stage 5 sub-path, not from Stage 1 context — `aos8_show_command(command='show version')`

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 12-central-enrichment-cutover-validation*
*Context gathered: 2026-04-29*
