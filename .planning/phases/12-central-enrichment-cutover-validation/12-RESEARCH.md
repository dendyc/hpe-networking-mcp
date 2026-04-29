# Phase 12: Central Enrichment & Cutover Validation - Research

**Researched:** 2026-04-29
**Domain:** Markdown skill authoring (FastMCP skills) — Central-side enrichment + cutover prerequisite narration over live AOS8 + Central data
**Confidence:** HIGH

## Summary

Phase 12 is a **markdown-only** edit to `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md`. No Python code, no new tools, no new frontmatter entries. All five Central tools (`central_recommend_firmware`, `central_get_aps`, `central_get_wlan_profiles`, `central_get_roles`, `central_get_named_vlans`) and `aos8_show_command` are already declared in the skill `tools:` frontmatter (line 23). The Stage 1 four-batch live-mode collection (Phase 10) and the Stage 3 live-mode sub-path (Phase 11) are already in place; Phase 12 adds **two more sub-paths** — one inside Stage 4 (Central enrichment, ENRICH-01..04) above the A1–A13 table, and one inside Stage 5 (cutover prerequisites, CUTOVER-01..03) before the Phase 0–8 cutover table.

Both new sub-paths replicate the structural pattern Phase 11 already proved out for Stage 3: conditional `#####` header naming the trigger, brief intro line, sequential per-rule prose blocks consuming Stage 1 batch data already in context, partial-failure fallback that names the exact CLI command for the missing batch, and `**SEVERITY — Description. (source: tool_name(args), Batch N) (VSG §anchor)**` finding format. The only **new** runtime call introduced in Phase 12 is `aos8_show_command(command='show version')` — fresh inside Stage 5 because controller firmware version is not reliably surfaced by `show inventory` (Stage 1 Batch 3) per D-12. All other data is reused from Stage 1 context.

**Primary recommendation:** Mirror the Phase 11 Stage 3 sub-path verbatim — same heading style, same fallback footer, same citation format. Edit in two localized blocks (Stage 4 enrichment sub-path, Stage 5 prerequisites sub-path). Rerun `pytest tests/unit/test_skill_tool_references.py` after each block to catch any tool-name typos in the new prose.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Stage 4 Sub-Path Structure (ENRICH-01..04)**
- **D-01:** Insert a self-contained `##### AOS8 live-mode sub-path — Central enrichment` section **above** the existing A1–A13 table in Stage 4. The existing table stays intact as the paste fallback for non-live paths. Consistent with Stage 3's live-mode sub-path placement.
- **D-02:** The sub-path covers all four ENRICH requirements: ENRICH-01 (AP count gap), ENRICH-02 (per-model firmware recs), ENRICH-03 (SSID conflicts), ENRICH-04 (role/VLAN conflicts).
- **D-03:** The live-mode sub-path block explicitly notes "no re-fetching" — all AOS8 data is already in Stage 1 context (Batch 2 for AP inventory, Batch 1 effective config for roles/VLANs, Batch 4 bss_table for SSID names).

**ENRICH-01: AP Count Gap**
- **D-04:** Compare `aos8_get_ap_database()` total (from Stage 1 Batch 2 context) against `central_get_aps()` total. Emit: "Source AP count: X. Central onboarded: Y. Gap: Z not yet onboarded." (INFO severity — not a blocker, surfaces the migration scope.)

**ENRICH-02: Model → Firmware Recommendation**
- **D-05:** Call `central_recommend_firmware()` **once with no filter**. Central returns a fleet-wide recommendation list already grouped by model. The AI extracts per-model rows by cross-referencing against the distinct AP models from AOS8 Batch 2 inventory. One call for any fleet size — no per-model iteration needed.

**ENRICH-03 + ENRICH-04: Conflict Detection**
- **D-06:** Check **all three** conflict surfaces: SSID names (A7 via `central_get_wlan_profiles()`), role names (A8 via `central_get_roles()`), named VLAN IDs (A9 via `central_get_named_vlans()`). Cross-reference against:
  - SSID names: from `aos8_get_bss_table()` Batch 4
  - Role names + VLAN IDs: from `aos8_get_effective_config()` Batch 1
- **D-07:** Naming conflicts are **REGRESSION** severity — a conflicting SSID or role name in Central blocks a clean migration. This upgrades from the current INFO severity on A7/A8/A9.
- **D-08:** Each conflict is **listed individually** (one finding per conflicting item), matching the RULES-04 pattern of listing affected AP names. E.g.: `REGRESSION — SSID "Corp-WiFi" already exists in Central (profile ID: X). (source: central_get_wlan_profiles())`

**Stage 5 Sub-Path Structure (CUTOVER-01..03)**
- **D-09:** Insert a self-contained `##### AOS8 live-mode sub-path — cutover prerequisites` section **before** the Phase 0–8 cutover table in Stage 5. The existing table stays intact for paste paths. Mirrors Stage 3 and Stage 4 sub-path pattern.
- **D-10:** The sub-path covers CUTOVER-01 (cluster health), CUTOVER-02 (firmware floor), CUTOVER-03 (AP count snapshot) — evaluated once, before the operator walks through the phase table.

**CUTOVER-01: Live Cluster Health**
- **D-11:** Call `aos8_get_cluster_state()` (already in Stage 1 Batch 3 context — no re-fetch). Flag **REGRESSION** if anything other than L2-connected. E.g.: `REGRESSION — Cluster not L2-connected: <state>. Resolve before single-controller upgrade. (source: aos8_get_cluster_state(), Batch 3)`

**CUTOVER-02: Controller Firmware Floor**
- **D-12:** Call `aos8_show_command(command='show version')` **fresh in the Stage 5 sub-path** (not from Stage 1 context — firmware version is not reliably available from `show inventory`). Parse controller firmware version; flag **REGRESSION** if below 8.10.0.12 / 8.12.0.1 (VSG §1643-§1649). E.g.: `REGRESSION — Controller firmware 8.8.0.5 is below the 8.10.0.12 / 8.12.0.1 prerequisite. (source: aos8_show_command(command='show version'), VSG §1643-§1649)`
- **D-13:** `aos8_show_command` is already in the skill frontmatter `tools:` list (added Phase 10) — no new frontmatter addition required.

**CUTOVER-03: AP Count Baseline**
- **D-14:** Surface the live AP count from `aos8_get_ap_database()` Batch 2 context as an **INFO finding** in the Stage 5 sub-path: `INFO — Pre-cutover AP baseline: X APs. (source: aos8_get_ap_database(), Batch 2) Use this count for post-cutover diff.` Lives alongside cluster health and firmware check in the Stage 5 block.

### Claude's Discretion

- Exact wording for the Stage 4 and Stage 5 sub-path header and intro lines
- Whether the Stage 4 sub-path emits a brief "enrichment complete" summary before the live-mode block ends
- How to handle partial failures in Stage 4 (e.g., `central_get_wlan_profiles()` fails) — reuse D-07/D-08 partial-failure fallback pattern from Phase 10 (exact command or tool name only, no table reprint)
- Format of the per-model firmware recommendation table produced by ENRICH-02

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ENRICH-01 | AP count gap — compare AOS8 total vs Central onboarded total; report exact counts (X source, Y in Central, Z not onboarded) | Stage 1 Batch 2 (`aos8_get_ap_database()`) provides source count. `central_get_aps()` (no filter) returns Central-onboarded list — `len(response)` is the count. INFO severity per D-04. |
| ENRICH-02 | Model → firmware recommendation — extract distinct models from AOS8 inventory; surface recommended AOS10 firmware per model | One call to `central_recommend_firmware()` with no filter returns fleet-wide rec list grouped by `firmwareClassification` and device model (per `firmware.py:223–276`). Cross-reference against distinct `model` values from Batch 2 AP database. Per D-05: one call regardless of fleet size. |
| ENRICH-03 | SSID conflict detection — flag every AOS8 SSID name that already exists as a Central WLAN profile | Batch 4 (`aos8_get_bss_table()`) provides AOS8 SSID names. `central_get_wlan_profiles()` (no `ssid` arg) returns full profile list. One REGRESSION finding per matching name (D-07/D-08). |
| ENRICH-04 | Role/VLAN conflict detection — flag every AOS8 role name and VLAN ID that already exists in Central | Batch 1 effective config (`aos8_get_effective_config()`) provides role names + VLAN IDs. `central_get_roles()` and `central_get_named_vlans()` (no filter) return full lists. One REGRESSION finding per conflicting role name; one per conflicting VLAN ID. |
| CUTOVER-01 | Live cluster health — confirm L2-connected before single-controller upgrade | Batch 3 (`aos8_get_cluster_state()`) already in context per Phase 10 collection. REGRESSION on any state other than L2-connected (D-11). |
| CUTOVER-02 | Controller firmware version check — flag REGRESSION if below 8.10.0.12 / 8.12.0.1 | Fresh call: `aos8_show_command(command='show version')` inside Stage 5 sub-path (D-12 — not from Batch 3 context). Parse firmware string; compare against floor. VSG §1643-§1649. |
| CUTOVER-03 | AP count snapshot for pre/post-cutover diff | Reuse Batch 2 (`aos8_get_ap_database()`) total. INFO severity (D-14). |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

Two CLAUDE.md files apply: the workspace root `C:\Dev\adams_mcp\CLAUDE.md` (project-level) and `C:\Dev\adams_mcp\hpe-networking-mcp\CLAUDE.md` (repo-level). Combined directives relevant to Phase 12:

- **Markdown-only**: REQUIREMENTS.md "Out of Scope" — no Python code changes for v1.1
- **No new tools**: All Central tools used in Phase 12 prose (`central_recommend_firmware`, `central_get_aps`, `central_get_wlan_profiles`, `central_get_roles`, `central_get_named_vlans`) plus `aos8_show_command`, `aos8_get_cluster_state`, `aos8_get_ap_database`, `aos8_get_bss_table`, `aos8_get_effective_config` are already declared in the skill `tools:` frontmatter (line 23). **No `tools:` frontmatter changes required.**
- **Skill regression test gates**: `tests/unit/test_skill_tool_references.py` validates every tool name in skill body against catalog + `tools:` frontmatter. Tool name typos in new Phase 12 prose will fail CI. Phase 10 extended `_TOOL_REF_PATTERN` regex with `|aos8` so AOS8 typos fail loudly.
- **Skill file size**: Skill is currently 578 lines. Phase 12 adds ~60–90 more lines for the two sub-paths. The 500-line/file limit is **Python-only** per CLAUDE.md and does not apply to markdown skill files. No structural concern.
- **GSD workflow enforcement**: Project-level CLAUDE.md mandates GSD-driven edits — Phase 12 is already inside `/gsd:execute-phase` flow, so direct edits via planner-spawned executor are correct.
- **Pre-push CI** (repo-level): Run inside Docker — `uv run ruff check . && uv run ruff format --check . && uv run mypy src/ --ignore-missing-imports && uv run pytest tests/ -q`. Markdown changes only invoke pytest's skill-reference test materially, but Docker run is the canonical pre-push gate.
- **Commit message format**: Never include "claude code" or "written by claude code" in commit messages (repo-level CLAUDE.md).
- **Phase scope discipline**: D-15-style Phase 13 work (executive summary OUTPUT-01, frontmatter cleanups QUALITY-01..03) MUST NOT be performed in Phase 12 — these are explicitly deferred per REQUIREMENTS.md traceability table.

## Standard Stack

Phase 12 introduces **no new technology**. The "stack" is:

| Item | Version | Purpose | Source |
|------|---------|---------|--------|
| Markdown skill format | n/a | FastMCP skill discovery + body | `src/hpe_networking_mcp/skills/_engine.py` already loads the skill at startup |
| YAML frontmatter | n/a | Tool list + platforms tag | Already present, line 23 lists every tool Phase 12 references |
| `central_recommend_firmware` | catalog (v1.0+) | Fleet-wide AOS10 firmware rec list grouped by model | `platforms/central/tools/firmware.py:223` — accepts optional `serial_number`, `device_type`, `site_id`, `site_name`, `include_up_to_date`; **call with no args** for fleet-wide |
| `central_get_aps` | catalog (v1.0+) | Onboarded AP list — count via `len()` | `platforms/central/tools/monitoring.py:32` — accepts optional filters; **call with no args** for total fleet |
| `central_get_wlan_profiles` | catalog (v1.0+) | Existing Central WLAN profiles for SSID conflict | `platforms/central/tools/wlan_profiles.py:28` — `ssid: str \| None`; **omit for full list** |
| `central_get_roles` | catalog (v1.0+) | Existing Central roles for role-name conflict | `platforms/central/tools/roles.py:32` — `name: str \| None`; **omit for full list** |
| `central_get_named_vlans` | catalog (v1.0+) | Existing Central named VLANs for VLAN-ID conflict | `platforms/central/tools/named_vlans.py:15` — `name: str \| None`; **omit for full list** |
| `aos8_show_command` | catalog (v1.0+) | Fresh `show version` for controller firmware floor | `platforms/aos8/tools/troubleshooting.py` — passthrough CLI executor, takes `command` arg |
| `aos8_get_cluster_state` | catalog (v1.0+) | L2-connected check — Batch 3 context | `platforms/aos8/tools/differentiators.py` |
| `aos8_get_ap_database` | catalog (v1.0+) | AP count + distinct models — Batch 2 context | `platforms/aos8/tools/wlan.py` |
| `aos8_get_bss_table` | catalog (v1.0+) | AOS8 SSID names — Batch 4 context | `platforms/aos8/tools/wlan.py` |
| `aos8_get_effective_config` | catalog (v1.0+) | Role names + VLAN IDs — Batch 1 context | `platforms/aos8/tools/differentiators.py` |
| `pytest tests/unit/test_skill_tool_references.py` | pytest >= 8.4.0 | Validates every skill tool reference | Phase 10 hardened regex; Phase 11 used as commit gate; Phase 12 same |

**Installation:** None — markdown-only edits to an existing file.

**Version verification:** N/A — no library updates. All tools shipped in v1.0 (2026-04-29) and earlier. Frontmatter line 23 is the source of truth.

## Architecture Patterns

### Recommended Edit Plan

The skill file is one document; Phase 12 makes two localized inserts:

```
aos-migration-readiness.md (currently 578 lines)
├── Stage -1 (DETECT-01)              # Phase 10 — UNCHANGED
├── Stage 0 (interview)               # UNCHANGED
├── Stage 1 (live-mode + paste-fallback sub-paths)   # Phase 10 — UNCHANGED
├── Stage 2 (parse paste bundle, with skip clause)   # Phase 11 — UNCHANGED
├── Stage 3 (rules)
│   ├── AOS8 live-mode sub-path       # Phase 11 — UNCHANGED
│   ├── Universal rules (U1..U11)     # UNCHANGED
│   ├── AOS6/8 rules (C1..C10)        # UNCHANGED
│   ├── IAP rules                     # UNCHANGED
│   └── Per-target-mode rules         # UNCHANGED
├── Stage 4 (Central API checks)
│   ├── [NEW] AOS8 live-mode sub-path — Central enrichment (ENRICH-01..04) # Phase 12 EDIT 1: insert above A1
│   └── A1..A13 table                 # UNCHANGED (paste-fallback path)
├── Stage 5 (cutover)
│   ├── [NEW] AOS8 live-mode sub-path — cutover prerequisites (CUTOVER-01..03) # Phase 12 EDIT 2: insert above Phase 0
│   └── Phase 0..8 table              # UNCHANGED (paste-fallback path)
├── Stage 6 (report)                  # UNCHANGED
├── Decision matrix                   # UNCHANGED
└── Output formatting                 # UNCHANGED
```

### Pattern 1: Live/Paste Sub-Path Header (mirror Stage 3 from Phase 11)

**What:** Each sub-path uses an `#####` heading with a conditional clause naming the trigger.
**When to use:** Both Stage 4 and Stage 5 new sub-paths must mirror the existing Stage 3 sub-path header style (skill line ~230) and the Stage 1 sub-path header style (line ~95).
**Example (canonical pattern, Stage 3 line 230):**
```markdown
#### AOS8 live-mode sub-path — rules evaluated from Stage 1 data (used when Stage -1 announced "AOS8 API mode")

When AOS8 live mode is active, RULES-01, RULES-02, and RULES-04 are evaluated directly from the Stage 1 batch data already in context — no re-fetching, no operator paste. RULES-03 result is **pending Stage 4 A11** (ClearPass cross-check). After this sub-path, continue with the universal + AOS6/8 + per-target-mode rule tables below — those apply regardless of source path.
```

Phase 12 Stage 4 should use:
```markdown
##### AOS8 live-mode sub-path — Central enrichment (ENRICH-01..04, used when Stage -1 announced "AOS8 API mode")

When AOS8 live mode is active, the four Central enrichment checks (AP count gap, per-model firmware recommendations, SSID conflicts, role/VLAN conflicts) are evaluated using AOS8 batch data already in context plus a small set of Central tool calls — no operator paste. After this sub-path, continue with the A1–A13 Central API checks below; A4/A7/A8/A9/A13 are superseded by this sub-path's findings when live mode is active.
```

Phase 12 Stage 5 should use:
```markdown
##### AOS8 live-mode sub-path — cutover prerequisites (CUTOVER-01..03, used when Stage -1 announced "AOS8 API mode")

When AOS8 live mode is active, three cutover prerequisites — cluster L2-connected health, controller firmware version floor, and pre-cutover AP-count baseline — are evaluated before the operator walks through the Phase 0–8 cutover sequence below.
```

### Pattern 2: Per-Rule Finding Format (mirror Stage 3 RULES-04 line 268)

**What:** Each finding cites severity, description, source tool call (with args + batch number), and VSG anchor when applicable.
**When to use:** Every emitted finding in both Phase 12 sub-paths.
**Examples** (per D-04, D-08, D-11, D-12, D-14):

```markdown
- **INFO** — Source AP count: 142. Central onboarded: 138. Gap: 4 not yet onboarded. (source: `aos8_get_ap_database()` Batch 2 + `central_get_aps()`)

- **REGRESSION** — SSID "Corp-WiFi" already exists in Central (profile present in `central_get_wlan_profiles()` response). (source: `aos8_get_bss_table()` Batch 4 + `central_get_wlan_profiles()`)

- **REGRESSION** — Role "guest-role" already exists in Central. (source: `aos8_get_effective_config(object_name='user_role', config_path='/md')` Batch 1 + `central_get_roles()`)

- **REGRESSION** — Named VLAN ID 100 already mapped in Central (existing name: "USER-VLAN"). (source: `aos8_get_effective_config()` Batch 1 + `central_get_named_vlans()`)

- **REGRESSION** — Cluster not L2-connected: `<state>`. Resolve before single-controller upgrade. (source: `aos8_get_cluster_state()`, Batch 3)

- **REGRESSION** — Controller firmware 8.8.0.5 is below the 8.10.0.12 / 8.12.0.1 prerequisite. (source: `aos8_show_command(command='show version')`) (VSG §1643-§1649)

- **INFO** — Pre-cutover AP baseline: 142 APs. (source: `aos8_get_ap_database()`, Batch 2) Use this count for post-cutover diff.
```

### Pattern 3: Per-Model Firmware Recommendation Table (D-05, ENRICH-02)

**What:** A compact two-column markdown table the AI emits inline, populated from `central_recommend_firmware()` cross-referenced against distinct AP models in AOS8 Batch 2.
**When to use:** Inside the Stage 4 sub-path, between the AP-count-gap finding and the conflict findings.
**Example skeleton:**
```markdown
**ENRICH-02 — Per-model AOS10 firmware recommendation** (one call: `central_recommend_firmware()`; no filter)

| AP Model | Recommended AOS10 Firmware |
|----------|----------------------------|
| 535      | 10.7.x.y                   |
| 615      | 10.7.x.y                   |
| 655      | 10.7.x.y                   |

(source: distinct models from `aos8_get_ap_database()` Batch 2; recommendations from `central_recommend_firmware()`)
```

The AI extracts model rows by walking `central_recommend_firmware()` response (a `FirmwareRecommendationReport`, see `firmware.py:276`) and matching the device-model field against the distinct `model` set from Batch 2. If Central recommendation list omits a model the AOS8 fleet has, emit one row per missing model with "no recommendation returned — verify with Aruba support."

### Pattern 4: Partial-Failure Fallback (reuse Phase 10 D-07/D-08 pattern)

**What:** If a Central tool call fails or a Stage 1 batch is unavailable, name the exact tool / CLI command needed to recover that single rule — do not reprint the full table.
**When to use:** Per check inside both Phase 12 sub-paths.
**Example:**
```markdown
*If `central_get_wlan_profiles()` fails:* SSID conflict check (ENRICH-03) cannot complete. Fall back to A7 in the table below.

*If Batch 4 was unavailable:* AOS8 SSID list is unknown — operator paste of `show ap essid` required for ENRICH-03.
```

### Anti-Patterns to Avoid

- **Don't iterate `central_recommend_firmware()` per AP model.** D-05 mandates one fleet-wide call. Per-model iteration violates the decision and wastes API budget on large fleets.
- **Don't re-fetch Stage 1 batches.** All AOS8 data needed for ENRICH-01/02/03/04 and CUTOVER-01/03 is already in context. Re-fetching duplicates calls and breaks the "no re-fetching" promise in D-03.
- **Don't reuse `show inventory` for firmware floor.** D-12 explicitly requires a **fresh** `aos8_show_command(command='show version')` because `show inventory` does not reliably surface controller firmware. Pulling firmware from Batch 3 will silently produce wrong findings.
- **Don't downgrade conflict severity.** Existing A7/A8/A9 are INFO; the live-mode sub-path **upgrades** these to REGRESSION per D-07. Do not copy the INFO severity from the table.
- **Don't combine multiple conflicts into one finding.** D-08 requires one finding per conflicting item (matches RULES-04 per-AP listing pattern from Phase 11).
- **Don't add to `tools:` frontmatter.** Every tool name needed by Phase 12 prose is already at line 23 (verified — see Reusable Assets below). Adding duplicate or alternate names will fail the skill regression test.
- **Don't touch Stage 6 / Decision matrix / Output formatting.** Phase 13 owns OUTPUT-01/02. Phase 12 stops at Stage 5.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Per-model firmware lookup | A loop calling `central_recommend_firmware(serial_number=...)` per AP | Single `central_recommend_firmware()` call with no filter | D-05 + tool docstring (`firmware.py:283-292`) — Central returns fleet-wide list grouped by model. Iteration explodes API calls and contradicts the locked decision. |
| Local-user / role / SSID set diff | Custom set-intersection logic in skill prose | Plain narration: "for each item in AOS8 Batch X, check membership in Central response" | The AI handles the set diff at runtime; prose just declares the comparison surface and the per-conflict finding format (D-08). |
| New severity scheme | A new severity tier for "Central conflict" | Use existing REGRESSION (D-07) + INFO (D-04, D-14) | The audit's three-tier model (REGRESSION / DRIFT / INFO) is locked — Phase 12 maps cleanly into it. |
| Custom controller-firmware comparator | A version-string comparison routine described in prose | Inline inequality check against the two known floors `8.10.0.12` / `8.12.0.1` | Two known thresholds, integer-keyed dotted version. The AI does this trivially at runtime. |

**Key insight:** Phase 12 is **prose layout**, not algorithm design. The downstream AI executes the comparisons; the skill's job is to (a) name the data sources, (b) name the conflict surface, (c) lock the severity, (d) lock the finding-line format. Anything more is over-specification and risks brittle prose.

## Runtime State Inventory

This phase is markdown-only; the change is restricted to one file. Inventory:

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | None — verified, this skill produces no persistent state. The audit is read-only and emits a one-shot report. | None |
| Live service config | None — verified, no service registration changes. The skill loads from disk at FastMCP server start (`skills/_engine.py`); no daemon, no scheduler entry. | None |
| OS-registered state | None — verified, no Windows Task Scheduler / launchd / systemd entries reference this skill. | None |
| Secrets/env vars | None — verified, no env-var or SOPS-key names change. AOS8 + Central credentials are already mounted via Docker secrets and unchanged by markdown edits. | None |
| Build artifacts | **One concern**: the skill file ships inside the wheel (`hatchling` build → `src/hpe_networking_mcp/skills/*.md` packaged). After edit, a `pip install -e .` already-installed dev environment picks the change up immediately because the source dir is the install dir. **Docker images** built before the edit will carry the old skill — operators running the published image must `docker compose up -d --build` after the merge to pick up the new skill prose. | Document in PR/release notes: "Rebuild local image to pick up skill change." Same posture as Phase 10 / 11. |

**The canonical question:** *After every file in the repo is updated, what runtime systems still have the old string cached, stored, or registered?* — Answer: published Docker images and pre-built wheels. Both are normal release artifacts and require a rebuild, which is the standard release flow already documented in `hpe-networking-mcp/CLAUDE.md` § "Release Workflow."

## Common Pitfalls

### Pitfall 1: Tool name typo silently fails the regression test
**What goes wrong:** A misspelled tool name in skill prose (e.g., `central_get_wlan_profile` missing the trailing `s`) doesn't match any catalog entry; `tests/unit/test_skill_tool_references.py` flags it.
**Why it happens:** Phase 12 introduces 5 Central tool name references in new prose, plus reuses 5 AOS8 names. Copy-paste from this RESEARCH.md or the canonical references in CONTEXT.md is safer than retyping.
**How to avoid:** After each edit block, run `pytest tests/unit/test_skill_tool_references.py -x`. Phase 11 hardened this regex to catch `aos8_*` typos; Central names are also covered by the existing `_TOOL_REF_PATTERN`.
**Warning signs:** CI failure with "Tool reference X not found in catalog" message.

### Pitfall 2: Pulling controller firmware from Stage 1 Batch 3 instead of fresh `show version`
**What goes wrong:** Batch 3's `show inventory` does not reliably include the running OS firmware string. Reusing it for CUTOVER-02 produces inconclusive or false-negative findings.
**Why it happens:** Batch 3 also collects `aos8_show_command` outputs, so it's tempting to pull `show version` from there if it appears in any past batch.
**How to avoid:** D-12 is explicit — Stage 5 sub-path makes a **fresh** call to `aos8_show_command(command='show version')`. The skill prose must include that exact call (with the `command` kwarg literally `'show version'`).
**Warning signs:** Prose that says "from Stage 1 Batch 3" anywhere near the CUTOVER-02 finding.

### Pitfall 3: One finding for many conflicts
**What goes wrong:** Prose says "Found 3 conflicting SSIDs: A, B, C" — collapses the conflicts into a single finding.
**Why it happens:** It reads more compactly. But it breaks the report's per-item paste-into-customer-doc usability and contradicts D-08.
**How to avoid:** D-08 explicit: one bullet per conflict. Mirror the RULES-04 per-AP enumeration pattern from Phase 11 (skill line ~268).
**Warning signs:** Prose using "Found N conflicts:" or comma-separated lists inside a single finding.

### Pitfall 4: A7/A8/A9 INFO severity carried into the live-mode sub-path
**What goes wrong:** The existing Stage 4 A7/A8/A9 rows say INFO. The live-mode sub-path **upgrades** these to REGRESSION (D-07). Copying severity from the table without re-reading D-07 produces under-flagged findings.
**Why it happens:** Symmetry-bias when paraphrasing existing rules.
**How to avoid:** Hard-code REGRESSION in every conflict finding template the prose introduces.
**Warning signs:** Any conflict finding in the new sub-path emitting INFO.

### Pitfall 5: Iterating `central_recommend_firmware()` per model
**What goes wrong:** Per-model loops violate D-05 and waste API budget.
**Why it happens:** The naive read of "per-model firmware recommendation" suggests one call per model.
**How to avoid:** Skill prose explicitly says "**one call**, fleet-wide, no filter — extract per-model rows from the response."
**Warning signs:** Prose wording like "for each model, call `central_recommend_firmware(...)`."

### Pitfall 6: Frontmatter drift
**What goes wrong:** Adding tool names already present (creates duplicates) or omitting them when editing the line.
**Why it happens:** Reflexively touching `tools:` whenever a phase adds tool references.
**How to avoid:** Verify against current line 23. CONTEXT.md confirms all needed tools are present. **Do not edit line 23 in Phase 12.**
**Warning signs:** A diff that touches line 23.

### Pitfall 7: Bleeding into Phase 13 scope
**What goes wrong:** Updating the Stage 6 executive summary, OUTPUT-01/02 scaffolding, or QUALITY-01..03 frontmatter cleanups. Those are Phase 13.
**Why it happens:** While editing the skill, "one more small fix" temptation.
**How to avoid:** Stage 6 and Decision matrix are explicitly outside Phase 12's diff. Verify before commit.
**Warning signs:** A diff touching anything below Stage 5's last line of new content.

## Code Examples

Verified patterns from the existing skill (canonical templates to mirror):

### Existing Stage 3 sub-path header (line 230 — pattern to mirror)
```markdown
#### AOS8 live-mode sub-path — rules evaluated from Stage 1 data (used when Stage -1 announced "AOS8 API mode")

When AOS8 live mode is active, RULES-01, RULES-02, and RULES-04 are evaluated directly from the Stage 1 batch data already in context — no re-fetching, no operator paste. RULES-03 result is **pending Stage 4 A11** (ClearPass cross-check). After this sub-path, continue with the universal + AOS6/8 + per-target-mode rule tables below — those apply regardless of source path.

Each finding below uses the format **Severity — Description (VSG §anchor) (source: `tool_call(args)`, Batch N)**.
```

### Existing Stage 3 RULES-04 (line 264 — finding-list pattern to mirror for ENRICH-03/04)
```markdown
##### RULES-04 — Static AP IP detection (REGRESSION, VSG §1232-§1234)

From the Batch 2 `aos8_get_ap_database()` response, inspect the `ip_mode` field of each AP entry. For every AP whose `ip_mode` is anything other than DHCP (e.g., `static`), emit one finding:

- **REGRESSION** — AP `<ap_name>` (MAC `<wired_mac>`) has `ip_mode='<value>'` (statically addressed). AOS 10 does not support static AP IPs — APs must be DHCP-addressed before AP convert. (VSG §1232-§1234) (source: `aos8_get_ap_database()`, Batch 2)

If no APs have a non-DHCP `ip_mode`, emit a single INFO confirmation: "AP database shows all APs DHCP-addressed (RULES-04 PASS)."

*If Batch 2 failed:* RULES-04 cannot be evaluated from live data. Mark as **inconclusive — paste required** and consult any pasted `show ap database long` output. Equivalent paste-mode rule is U2 (Universal rules table) below.
```

### Tool docstring confirming `central_recommend_firmware()` returns fleet-wide grouping
```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/firmware.py:277-292
"""Apply an LSR-preferred upgrade policy to every device's firmware.

The tool calls `/network-services/v1/firmware-details`, classifies each
device from the `firmwareClassification` field the API returns (LSR,
SSR, or empty), and recommends accordingly:
  ...
The "newest LSR" target is mined live from the same response — no
hand-maintained train mapping. ..."""
```
This confirms D-05: one call covers the whole fleet; no per-model iteration.

### `central_get_aps` signature (count via len)
```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/monitoring.py:33-44
async def central_get_aps(
    ctx: Context,
    site_id: str | None = None,
    site_name: str | None = None,
    serial_number: str | list[str] | None = None,
    device_name: str | list[str] | None = None,
    status: Literal["ONLINE", "OFFLINE"] | None = None,
    model: str | list[str] | None = None,
    firmware_version: str | list[str] | None = None,
    deployment: Literal["Standalone", "Cluster", "Unspecified"] | None = None,
    sort: str | None = None,
) -> list[dict] | str:
```
All parameters optional; calling with no args returns the full onboarded list. Count = `len(response)` when response is a list.

### `central_get_wlan_profiles` / `central_get_roles` / `central_get_named_vlans` (full-list mode)
```python
# All three follow the same shape:
# Source: platforms/central/tools/{wlan_profiles,roles,named_vlans}.py
async def central_get_wlan_profiles(ctx, ssid: str | None = None) -> dict | list | str: ...
async def central_get_roles(ctx, name: str | None = None) -> dict | list | str: ...
async def central_get_named_vlans(ctx, name: str | None = None) -> dict | list | str: ...
```
Omit the optional `ssid` / `name` arg → returns full list. The skill prose simply tells the AI to call each with no argument.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Stage 4 paste-driven A1–A13 enrichment, all severities INFO/DRIFT | Stage 4 live-mode sub-path with REGRESSION on naming conflicts | Phase 12 (this phase) | Naming conflicts blocked at audit time, not discovered post-cutover |
| Stage 5 cutover prerequisites verified via operator paste of `show lc-cluster group-membership` | Live `aos8_get_cluster_state()` + fresh `aos8_show_command(command='show version')` | Phase 12 | Removes operator-paste step; firmware floor enforced automatically |
| Per-rule paste-mode evaluation (Phase 0 of v1.0) | Two-sub-path pattern (live + paste fallback) | Phase 10 introduced the pattern; Phases 11–12 propagate it through Stages 3–5 | Consistent UX across AOS8 source path; AOS6/IAP paste paths unchanged |

**Deprecated/outdated:** None within Phase 12 scope. All five Central tools are current; `aos8_show_command` is the canonical passthrough for any AOS8 CLI not yet wrapped by a typed tool.

## Open Questions

1. **Does the `FirmwareRecommendationReport` payload from `central_recommend_firmware()` include the AOS8 controllers themselves, or only APs?**
   - What we know: The tool's docstring lists "Unclassified (typically AOS 8)" as a class it handles (`firmware.py:287`), and `device_type` accepts `"ACCESS_POINT"`, `"SWITCH"`, `"GATEWAY"` — no explicit "AOS8 controller" type. With no `device_type` filter, the call returns everything Central knows about.
   - What's unclear: Whether AOS8 Mobility Controllers appear at all in a Central tenant's `/firmware-details` response when they're not yet onboarded as AOS10 gateways.
   - Recommendation: ENRICH-02 prose should restrict the per-model table to AP models only (cross-reference distinct AP models from AOS8 Batch 2). If the response also surfaces controller firmware recommendations, ignore them in this sub-path — CUTOVER-02 owns the controller firmware floor independently and is anchored to the VSG §1643-§1649 threshold, not Central's recommendation engine.

2. **What does `central_get_wlan_profiles()` return when the Central tenant has zero profiles?**
   - What we know: The tool returns `response.get("msg", {})` (`wlan_profiles.py:58`) — could be `{}` (empty dict) or `[]` (empty list) or `""` depending on Central API behavior.
   - What's unclear: Empty-tenant edge case is not exercised in the existing test suite (no `aos8`-paired test fixture).
   - Recommendation: Skill prose should narrate the zero-conflict case as a single INFO bullet ("No SSID conflicts detected against existing Central WLAN profiles") regardless of whether the API returns `{}`, `[]`, or empty string. The AI can disambiguate at runtime.

3. **Is there a live AOS8 + Central paired environment to validate the prose end-to-end?**
   - What we know: Phase 10 and Phase 11 both deferred Scenario A (live AOS8 announcement) to a future session — there's no live AOS8 environment per STATE.md (line 21, 64).
   - What's unclear: Phase 12's live-mode sub-paths have no live exercise path either.
   - Recommendation: Mirror Phase 10 / 11 disposition — accept partial approval at verify time. The pytest skill-reference test plus mechanical review of the prose against this RESEARCH.md and the CONTEXT.md decisions are the gate. A live exercise can be deferred without blocking Phase 12 sign-off.

## Environment Availability

Phase 12 is markdown-only — no new external dependencies introduced. Existing dependencies (already verified at v1.0 release):

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| pytest | `tests/unit/test_skill_tool_references.py` (regression gate) | ✓ (assumed; v1.0 ships green CI) | >= 8.4.0 | — |
| FastMCP skills engine | Skill discovery + load | ✓ (in production) | fastmcp >= 3.1.1 | — |
| Live AOS8 controller | Manual end-to-end validation only (Scenario A — deferred) | ✗ | — | Mechanical prose review against RESEARCH.md + CONTEXT.md (matches Phase 10/11 disposition) |
| Live Central tenant | Manual end-to-end validation only | ✗ in dev | — | Same — mechanical review |

**Missing dependencies with no fallback:** None — the regression test alone gates the phase, and prose is verifiable by inspection.

**Missing dependencies with fallback:** Live AOS8 + Central — fallback is mechanical review (Phase 10/11 precedent).

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest >= 8.4.0 with pytest-asyncio (auto mode) and responses (HTTP mocking) |
| Config file | `hpe-networking-mcp/pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` |
| Full suite command | `cd hpe-networking-mcp && uv run pytest tests/ -q` (or Docker pre-push checklist per repo CLAUDE.md) |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ENRICH-01 | AP-count gap finding mentions `aos8_get_ap_database` and `central_get_aps` tool names | unit (skill-ref) | `uv run pytest tests/unit/test_skill_tool_references.py -x` | ✅ |
| ENRICH-02 | Firmware-rec finding mentions `central_recommend_firmware` tool name | unit (skill-ref) | same | ✅ |
| ENRICH-03 | SSID-conflict finding mentions `central_get_wlan_profiles` and `aos8_get_bss_table` | unit (skill-ref) | same | ✅ |
| ENRICH-04 | Role/VLAN-conflict finding mentions `central_get_roles`, `central_get_named_vlans`, `aos8_get_effective_config` | unit (skill-ref) | same | ✅ |
| CUTOVER-01 | Cluster-health finding mentions `aos8_get_cluster_state` | unit (skill-ref) | same | ✅ |
| CUTOVER-02 | Firmware-floor finding mentions `aos8_show_command` | unit (skill-ref) | same | ✅ |
| CUTOVER-03 | AP-baseline finding mentions `aos8_get_ap_database` | unit (skill-ref) | same | ✅ |
| All | Markdown structure parses (no header-level errors) | manual-only — visual diff in PR review | `git diff src/hpe_networking_mcp/skills/aos-migration-readiness.md` | n/a |
| All (Scenario A) | End-to-end live AOS8 + Central audit produces correct findings | manual-only — deferred per Phase 10/11 precedent (no live env) | n/a | n/a |

### Sampling Rate
- **Per task commit:** `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` (sub-second)
- **Per wave merge:** `cd hpe-networking-mcp && uv run pytest tests/ -q` (full suite — green required)
- **Phase gate:** Full suite green inside Docker per `hpe-networking-mcp/CLAUDE.md` Pre-Push Checklist before `/gsd:verify-work`.

### Wave 0 Gaps
None — `tests/unit/test_skill_tool_references.py` exists and was hardened in Phase 10 with `|aos8` regex and is the canonical regression gate. Every tool name Phase 12 prose introduces is already in the catalog and `tools:` frontmatter, so the test will pass on green prose without further fixture work.

## Sources

### Primary (HIGH confidence)
- `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` (lines 23, 95, 230, 264, 360–395) — frontmatter, sub-path templates, Stage 4 A1–A13 table, Stage 5 Phase 0–8 table
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/firmware.py` (lines 223–292) — `central_recommend_firmware` signature, return type, fleet-wide policy
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/monitoring.py` (lines 32–94) — `central_get_aps` signature
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/wlan_profiles.py` (lines 28–58) — `central_get_wlan_profiles` signature + return shape
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/roles.py` (lines 32–59) — `central_get_roles` signature
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/named_vlans.py` (full file, 43 lines) — `central_get_named_vlans` signature
- `.planning/phases/12-central-enrichment-cutover-validation/12-CONTEXT.md` — locked decisions (D-01..D-14)
- `.planning/REQUIREMENTS.md` §ENRICH-01..04, §CUTOVER-01..03 — acceptance criteria
- `.planning/phases/11-live-vsg-rules/11-RESEARCH.md` — Phase 11 sub-path pattern that Phase 12 mirrors
- `.planning/phases/10-live-detection-collection/10-CONTEXT.md` — partial-failure fallback pattern (D-07/D-08) reused for Stage 4 sub-path
- `.planning/STATE.md` (lines 19–43) — current phase position, prior phase outcomes, Scenario A deferral disposition
- `C:\Dev\adams_mcp\CLAUDE.md` (root) and `C:\Dev\adams_mcp\hpe-networking-mcp\CLAUDE.md` (repo) — project + repo conventions

### Secondary (MEDIUM confidence)
- None — every claim in this RESEARCH.md is anchored to a primary source above.

### Tertiary (LOW confidence)
- None.

## Metadata

**Confidence breakdown:**
- Standard stack: **HIGH** — every tool already in catalog, signatures verified by reading source files
- Architecture: **HIGH** — Phase 11 already proved the sub-path pattern in Stage 3 of the same skill; Phase 12 replicates it in Stages 4 and 5 with no structural divergence
- Pitfalls: **HIGH** — derived from explicit decisions (D-04..D-14) and prior-phase precedent (Phase 10 fallback pattern, Phase 11 frontmatter discipline)
- Open questions: **HIGH** — all three open questions have explicit recommendations grounded in code or precedent

**Research date:** 2026-04-29
**Valid until:** 2026-05-29 (30 days — internal-skill prose with stable tool catalog; no fast-moving external dependencies)
