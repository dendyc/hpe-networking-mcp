# Phase 11: Live VSG Rules - Context

**Gathered:** 2026-04-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 11 modifies `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` (markdown only — no Python code changes) so that Stage 3's four AOS8-specific rules are evaluated from live Stage 1 data, auto-flagging REGRESSION/DRIFT findings without operator paste:

- **RULES-01**: VRRP VIP check — AP system profiles route APs to VRRP virtual IP, not individual controller IPs (flags REGRESSION, VSG §1654-§1657)
- **RULES-02**: ARM / radio profile detection — ARM, dot11a/g radio, and regulatory domain profiles per AP-group (flags DRIFT, VSG §1163-§1166)
- **RULES-03**: Local user count cross-check — AOS8 `show local-user db` count vs ClearPass; deferred to Stage 4 A11 (flags DRIFT)
- **RULES-04**: Static AP IP detection — APs with `ip_mode` != DHCP (flags REGRESSION, VSG §1232-§1234)

All four rules cite their VSG section reference in the finding text and identify the Stage 1 tool call that produced the evidence.

**Requirements in scope:** RULES-01, RULES-02, RULES-03, RULES-04

</domain>

<decisions>
## Implementation Decisions

### Stage 3 Structure (RULES-01, RULES-02, RULES-04)

- **D-01:** Stage 3 mirrors Stage 1's live/paste sub-path pattern. Add an explicit **"AOS8 live-mode sub-path"** section within Stage 3's AOS8-specific rules block — parallel to the paste-driven evaluation path that already exists implicitly (Stage 2 parsed data).
- **D-02:** The live-mode sub-path runs **before** the existing universal + AOS6/8 rule tables. It evaluates RULES-01, RULES-02, and RULES-04 from Stage 1 context data (no re-fetching). After the live-mode block, the universal rules (U1, U3, U5, etc.) continue as normal — those apply regardless of source path.

### Stage 2 Live Bypass

- **D-03:** Add a **skip clause at the top of Stage 2** for AOS8 live mode: *"If AOS8 live mode (Stage -1 announced API mode): Stage 1 data is already available — skip paste parsing for AOS8 data points and proceed directly to Stage 3."* This matches Stage 1's established sub-path pattern and removes ambiguity for the AI about whether to wait for paste.

### RULES-03 ClearPass Timing

- **D-04:** RULES-03 **defers to Stage 4 A11**. Stage 3's live-mode sub-path notes: *"RULES-03 result pending Stage 4 check A11"* — and Stage 4 A11 (`clearpass_get_local_users()`) gets expanded to perform the cross-check using the AOS8 local-user count already in context from Stage 1 Batch 3. No duplicate ClearPass call. Clear separation: Stage 3 evaluates AOS8-only data; Stage 4 handles cross-platform comparisons.

### Rule Parsing Guidance (Field-Level)

- **D-05:** Each rule in the live-mode sub-path specifies the **exact config object field** to check. Reduces AI hallucination risk on ambiguous JSON structure:
  - **RULES-01 (VRRP VIP)**: In the `ap_sys_prof` effective config response from Batch 1, check the LMS IP field — flag REGRESSION if it matches a controller management IP rather than the VRRP virtual IP (VSG §1654-§1657). Each finding cites which AP system profile name contains the wrong IP.
  - **RULES-02 (ARM/radio profiles)**: In the Batch 1 effective config results for `arm_profile`, `dot11a_radio_prof`, `dot11g_radio_prof`, `reg_domain_profile` — detect presence (any non-empty response) and enumerate which profiles are active per AP-group. Flag as DRIFT (VSG §1163-§1166). Report all detected profile names.
  - **RULES-04 (static AP IPs)**: In the `aos8_get_ap_database()` response from Batch 2, check the `ip_mode` field per AP — flag REGRESSION for every AP where `ip_mode` is not DHCP (VSG §1232-§1234). List affected AP names/MACs.

### Claude's Discretion

- Exact wording of the "AOS8 live-mode sub-path" header and intro line in Stage 3
- How to format per-rule findings inline vs in the final rule tables (e.g., whether to annotate the C2/C4/U2 rows or produce a separate list)
- Whether to add a "fallback note" to each RULES-0X if the relevant Stage 1 batch failed (could reuse D-07/D-08 from Phase 10 — ask operator for the specific CLI command only)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Skill file being modified
- `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` — The skill being modified for Phase 11. Read the full file to understand: Stage -1 detection block (added in Phase 10), Stage 1 live-mode sub-path structure (Batches 1-4), Stage 2 AOS8 extraction table, Stage 3 rule tables (C2, C4, U2 are the paste-mode equivalents of RULES-01, RULES-02, RULES-04), and Stage 4 A11 check.

### Phase 10 context (data already collected)
- `.planning/phases/10-live-detection-collection/10-CONTEXT.md` — D-04 through D-09 define what Stage 1 collects and how partial failures work. Phase 11's live-mode sub-path builds on this — particularly: Batch 1 collects `ap_sys_prof`, `arm_profile`, `dot11a_radio_prof`, `dot11g_radio_prof`, `reg_domain_profile` via `aos8_get_effective_config()`; Batch 2 collects `ip_mode` per AP; Batch 3 collects `show local-user db` count.

### Requirements
- `.planning/REQUIREMENTS.md` §RULES-01..04 — Authoritative acceptance criteria for Phase 11. Each rule specifies the VSG section it anchors to and the tool call that provides evidence.

### VSG rule anchors (Stage 3 in the skill file)
- The existing rule tables in Stage 3 (C2 at VSG §1651-§1657, C4 at VSG §1163-§1166, U2 at VSG §1232-§1234) define the paste-mode equivalents of RULES-01, RULES-02, RULES-04 respectively. Phase 11's live-mode sub-path evaluates the same rules with the same severity/anchor citations — no new rules are invented.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- `Stage -1 detection block` — Establishes the live/paste branching pattern already in the skill. Phase 11's Stage 3 live-mode sub-path uses the same conditional ("if AOS8 live mode") as Stage -1.
- `Stage 1 live-mode sub-path structure` — Four-batch narration format with per-batch fallback behavior (D-07/D-08 from Phase 10). Phase 11 may reuse the fallback note pattern for rules that depend on a failed batch.
- Stage 2 AOS8 extraction table (lines ~186-201 in the skill) — Lists all AOS8 data points and their VSG anchors. Phase 11's Stage 2 skip clause sits above this table; the table itself is unchanged and remains valid for the paste-fallback path.
- Stage 4 A11 check (`clearpass_get_local_users()`) — RULES-03 defers here; Phase 11 expands A11 to include the cross-check comparison using the AOS8 count from Stage 1 Batch 3 context.

### Established Patterns

- Skills reference tools by name in `tools:` frontmatter — all AOS8 tools already added in Phase 10; no new frontmatter additions needed for Phase 11 (the tools used — `aos8_get_effective_config`, `aos8_get_ap_database`, `aos8_show_command` — are already listed).
- Live/paste sub-path structure from Stage 1 is the model: each sub-path starts with a conditional header, uses direct prose for the live path, and has a fallback note for errors.
- Finding format in Stage 3 rule tables uses: **Severity — Description. (VSG §anchor)**. Phase 11's live-mode findings use the same format.

### Integration Points

- Stage 2 → Stage 3 flow: Phase 11 inserts a skip clause at the top of Stage 2 (AOS8 live mode only), then Stage 3 gains its live-mode sub-path block (before the universal rule tables).
- Stage 3 → Stage 4 handoff: RULES-03 result is marked "pending Stage 4 A11" in Stage 3; Stage 4 A11 is expanded to complete the cross-check.
- No Python code changes — markdown-only per REQUIREMENTS.md Out of Scope.

</code_context>

<specifics>
## Specific Ideas

- Stage 3 live-mode sub-path header should follow the Stage 1 pattern: **`##### AOS8 live-mode sub-path — rules evaluated from Stage 1 data`**
- Stage 2 skip clause placement: above the AOS8 extraction table, similar to the Stage 1 sub-path conditional
- RULES-04 uses `ip_mode` field from `aos8_get_ap_database()` Batch 2 directly — ROADMAP.md success criterion explicitly notes "Stage 3 RULES-04 reuses `ip_mode` directly" (already in the Stage 1 Batch 2 prose)
- RULES-03 finding should include both counts: "AOS8 local users: X. ClearPass local users: Y. Dual-source-of-truth — DRIFT."
- Each finding in the live-mode sub-path should end with a parenthetical citing the source tool call, e.g., "(source: `aos8_get_effective_config(object_name='ap_sys_prof')`, Batch 1)"
- Fallback policy for RULES-01/02/04 if their respective Stage 1 batch failed: note that the rule cannot be evaluated from live data and remind the AI to check the pasted fallback data (if the operator provided it) or mark as "inconclusive — paste required"

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 11-live-vsg-rules*
*Context gathered: 2026-04-29*
