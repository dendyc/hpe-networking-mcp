# Phase 10: Live Detection & Collection - Context

**Gathered:** 2026-04-29
**Amended:** 2026-04-29 (D-04 clarified per checker review)
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 10 modifies `src/hpe_networking_mcp/skills/aos-migration-readiness.md` (markdown only — no Python code changes) so that when an operator runs the migration-readiness skill against an AOS8/Conductor source, the skill:

1. Auto-detects AOS8 connectivity at session start
2. Announces API mode (live data) before the Stage 0 interview begins
3. Gathers all Stage 1 data points via live AOS8 tool calls — no CLI paste required
4. Narrates collection in 4 grouped batches matching COLLECT-01..04
5. Falls back to targeted paste requests (exact commands only) for any tool that fails

AOS6 and IAP paste paths are entirely unchanged — no regression in non-AOS8 source paths.

**Requirements in scope:** DETECT-01, COLLECT-01, COLLECT-02, COLLECT-03, COLLECT-04

</domain>

<decisions>
## Implementation Decisions

### Detection (DETECT-01)

- **D-01:** Detection happens **before Stage 0** — the skill calls `health()` at session start. If AOS8 returns reachable, it announces "AOS8 API mode — live data" before beginning the Stage 0 interview. The operator knows upfront that paste won't be needed for AOS8 source.
- **D-02:** If AOS8 is NOT reachable, the skill proceeds **silently** into the existing Stage 0 interview — no announcement, no change in UX. The paste-driven flow is unchanged.
- **D-03:** Detection mechanism: call `health()` (or `health(platform="aos8")`) — consistent with the existing `infrastructure-health-check.md` pattern.

### Stage 1 AOS8 Paste Section (COLLECT-01..04)

- **D-04 (AMENDED 2026-04-29):** D-04 applies **only to the live-mode sub-path** of Stage 1 for AOS8 sources. The 16-command AOS8 paste table is removed from the **live-mode narration** — when AOS8 is reachable, Stage 1 runs API-only with no CLI paste table shown to the operator.

  The paste table is **retained in the skill file as the unreachable-AOS8 paste-fallback sub-path**, used when D-02's silent fallback is triggered (AOS8 not configured / unreachable). This honors both D-04 (no paste table in the live path) and D-02 (paste-driven flow unchanged when AOS8 is unreachable).

  Original phrasing of D-04 ("removed entirely") referred only to the live narration — the paste table must continue to exist somewhere in the skill so the unreachable-AOS8 path remains functional. The commands are also documented in the v1.0 TOOLS.md / INSTRUCTIONS.md for reference.
- **D-05:** Stage 0 paste paths for AOS6 and IAP are **fully preserved** — those sections are not touched.

### Live Collection Narration

- **D-06:** Collection is narrated in **4 grouped batches** matching the COLLECT requirements:
  - Batch 1 (COLLECT-01): MD hierarchy + per-node effective config — `aos8_get_md_hierarchy()` + `aos8_get_effective_config()`
  - Batch 2 (COLLECT-02): Full AP inventory — `aos8_get_ap_database()`
  - Batch 3 (COLLECT-03): Cluster state + running config + local-user db — `aos8_get_cluster_state()` + `aos8_show_command()` passthrough
  - Batch 4 (COLLECT-04): Client baseline + BSS/SSID table + active AP RF state + AP wired ports — `aos8_get_clients()` / `aos8_get_bss_table()` / `aos8_get_active_aps()` / `aos8_get_ap_wired_ports()`
  Each batch gets one line of narration (e.g., "Retrieving MD hierarchy and effective config...") followed by the tool calls, then a brief confirmation.

### Partial Collection Policy

- **D-07:** If some AOS8 tools fail mid-collection (any batch), the skill uses a **hybrid approach**: use what the API collected successfully, then request paste for the failed data points only.
- **D-08:** The paste request is **exact commands only** — tell the operator exactly which CLI command(s) to run for the failed data points (e.g., "show ap database long" failed — please run and paste that output). No reprint of the full table, no VSG anchors in the paste request.
- **D-09:** The hybrid fallback is per-batch granularity — a failure in Batch 3 doesn't invalidate the data already collected in Batches 1 and 2.

### Claude's Discretion

- Exact wording for the "AOS8 API mode — live data" announcement (tone, length)
- Exact wording for per-batch narration lines
- How the compiled Stage 1 inventory is presented after collection completes (table format, data structure)
- Whether collection batches run sequentially or the skill indicates parallel intent

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Skill file being modified
- `src/hpe_networking_mcp/skills/aos-migration-readiness.md` — The skill being rewritten for Phase 10. Read the full file to understand current Stage 0, Stage 1 (AOS8 paste table), and Stage 2 structure before planning edits.

### Platform tooling available for Stage 1 collection
- `src/hpe_networking_mcp/platforms/aos8/tools/` — All AOS8 tool files. Key tools for Phase 10: `differentiators.py` (hierarchy, effective config, cluster state, wired ports) and `clients.py`, `wlan.py`, `health.py`. Verify exact tool names and parameter signatures before referencing in skill prose.
- `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` — `aos8_get_md_hierarchy`, `aos8_get_effective_config`, `aos8_get_cluster_state`, `aos8_get_ap_wired_ports`
- `src/hpe_networking_mcp/platforms/aos8/tools/clients.py` — `aos8_get_clients`, `aos8_get_active_aps`
- `src/hpe_networking_mcp/platforms/aos8/tools/wlan.py` — `aos8_get_bss_table`, `aos8_get_ap_database`
- `src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py` — `aos8_show_command` (passthrough for running-config, local-user db, inventory, port/trunk)

### Detection pattern reference
- `src/hpe_networking_mcp/skills/infrastructure-health-check.md` — The established pattern for platform reachability detection via `health()`. Read the Step 1 procedure to understand how to probe and branch on result.

### Skill regression test
- `tests/unit/test_skill_tool_references.py` — Validates that every tool name referenced in a skill file's `tools:` frontmatter resolves to a real catalog entry. Any AOS8 tool names added to the frontmatter in Phase 10 must already exist in the catalog (they do — shipped in v1.0).

### Requirements
- `.planning/REQUIREMENTS.md` §DETECT-01, §COLLECT-01..04 — authoritative acceptance criteria for Phase 10

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- `health()` — Standard platform reachability probe used by `infrastructure-health-check.md`. Returns per-platform status including AOS8. Use this for detection; no new tooling needed.
- `aos8_get_md_hierarchy()` — Already in catalog from Phase 7 (differentiators.py). Replaces `show configuration node-hierarchy` paste.
- `aos8_get_effective_config(object_name, config_path)` — Replaces `show configuration effective detail <node>` paste.
- `aos8_get_ap_database()` — Replaces `show ap database long` paste.
- `aos8_get_cluster_state()` — Replaces `show lc-cluster group-membership` paste.
- `aos8_show_command(command)` — Passthrough for `show running-config`, `show local-user db`, `show inventory`, `show port status`, `show trunk` — replaces multiple paste items via one flexible tool.
- `aos8_get_clients()` — Replaces `show ap association` + `show user-table` paste.
- `aos8_get_bss_table()` — Replaces `show ap essid` paste.
- `aos8_get_active_aps()` — Replaces `show ap active` paste.
- `aos8_get_ap_wired_ports()` — Replaces `show ap lldp neighbors` paste.

### Established Patterns

- Skills reference tools by name in `tools:` frontmatter — every AOS8 tool called in the skill body must appear in the frontmatter list.
- Platform detection via `health()` followed by branching prose is the established skill pattern (see `infrastructure-health-check.md`).
- This is markdown-only work — REQUIREMENTS.md explicitly marks "Python code changes" as Out of Scope for v1.1.

### Integration Points

- Stage 1 in `aos-migration-readiness.md` currently has conditional sections per source platform (AOS8, AOS6, IAP). The AOS8 section is the one being replaced — the AOS6 and IAP sections are untouched.
- The `tools:` frontmatter currently lists only Central/ClearPass/GreenLake tools. Phase 10 adds AOS8 tool names to that list (Phase 13 QUALITY requirements enforce this, but the tools must be referenced correctly from Phase 10 onward).

</code_context>

<specifics>
## Specific Ideas

- Detection announcement before Stage 0: "AOS8 API mode — live data" (short, operator-friendly, matches the requirement wording from DETECT-01)
- Paste fallback is silent — no announcement when AOS8 not connected, existing paste UX unchanged
- The 16-command AOS8 CLI table is removed from the live-mode narration; it is retained as the unreachable-AOS8 paste-fallback sub-path within the same Stage 1 AOS8 section. Commands also remain in v1.0 TOOLS.md/INSTRUCTIONS.md.
- Narration is 4 batches grouped by COLLECT req — not individual tool-by-tool
- Partial failure → exact command name only in the paste request (e.g., "run `show ap database long` and paste the output") — no table reprint

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 10-live-detection-collection*
*Context gathered: 2026-04-29*
*D-04 amended: 2026-04-29 (live-mode scope clarification)*
