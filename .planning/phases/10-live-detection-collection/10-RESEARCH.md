# Phase 10: Live Detection & Collection - Research

**Researched:** 2026-04-29
**Domain:** FastMCP skill (markdown) authoring — live tool calls + paste fallback
**Confidence:** HIGH

## Summary

Phase 10 is **pure markdown editing** to a single file: `src/hpe_networking_mcp/skills/aos-migration-readiness.md`. No Python code is changed; every required AOS8 tool already shipped in v1.0 (Phases 1-9). The work is to (1) prepend a session-start detection step using `health()`, (2) replace the existing 16-row AOS8 paste table in Stage 1 with API-driven prose narrating four COLLECT batches, (3) add per-batch hybrid fallback to paste, and (4) update the skill's YAML frontmatter so the skill regression test (`tests/unit/test_skill_tool_references.py`) keeps passing.

The risks are not technical — they are editorial and locational: AOS6/IAP paste sections must be left untouched, tool names referenced in skill prose must match the catalog exactly (one signature mismatch was discovered: three tools live in `health.py`, not `wlan.py`/`clients.py` as CONTEXT.md implies), and the skill regression test currently uses a regex that does NOT match `aos8_*` names — meaning Phase 10's frontmatter additions will pass the test for a non-obvious reason (regex blindness), and Phase 13's QUALITY-03 work must extend the regex.

**Primary recommendation:** Treat this as a surgical markdown edit — preserve every AOS6/IAP line, replace the AOS8 Stage 1 table with a four-batch API procedure, add a Stage -1 detection step before Stage 0, and update the `tools:` and `platforms:` frontmatter. Verify all referenced tool names against actual decorator declarations in the codebase (not against CONTEXT.md, which has one location error).

## User Constraints (from CONTEXT.md)

### Locked Decisions

**Detection (DETECT-01):**
- **D-01:** Detection happens **before Stage 0**. Skill calls `health()` at session start. If AOS8 returns reachable, announce "AOS8 API mode — live data" before Stage 0 interview begins.
- **D-02:** If AOS8 NOT reachable, proceed **silently** into existing Stage 0 interview — no announcement, paste-driven flow unchanged.
- **D-03:** Detection mechanism: call `health()` (or `health(platform="aos8")`) — consistent with `infrastructure-health-check.md` pattern.

**Stage 1 AOS8 Paste Section (COLLECT-01..04):**
- **D-04:** The existing 16-command AOS8 paste table in Stage 1 is **removed entirely**. Stage 1 for AOS8 source becomes API-only. Commands remain documented in v1.0 TOOLS.md / INSTRUCTIONS.md.
- **D-05:** Stage 0 paste paths for AOS6 and IAP are **fully preserved** — those sections are not touched.

**Live Collection Narration:**
- **D-06:** Collection is narrated in **4 grouped batches** matching the COLLECT requirements:
  - Batch 1 (COLLECT-01): MD hierarchy + per-node effective config — `aos8_get_md_hierarchy()` + `aos8_get_effective_config()`
  - Batch 2 (COLLECT-02): Full AP inventory — `aos8_get_ap_database()`
  - Batch 3 (COLLECT-03): Cluster state + running-config + local-user db — `aos8_get_cluster_state()` + `aos8_show_command()` passthrough
  - Batch 4 (COLLECT-04): Client baseline + BSS/SSID table + active AP RF state + AP wired ports — `aos8_get_clients()` / `aos8_get_bss_table()` / `aos8_get_active_aps()` / `aos8_get_ap_wired_ports()`

**Partial Collection Policy:**
- **D-07:** If some AOS8 tools fail mid-collection, skill uses **hybrid approach** — use what API collected, request paste for failed data points only.
- **D-08:** Paste request is **exact commands only** — tell operator the exact CLI command(s) to run. No reprint of the full table, no VSG anchors.
- **D-09:** Hybrid fallback is **per-batch granularity** — failure in Batch 3 doesn't invalidate data already collected in Batches 1-2.

### Claude's Discretion
- Exact wording for the "AOS8 API mode — live data" announcement (tone, length)
- Exact wording for per-batch narration lines
- How the compiled Stage 1 inventory is presented after collection completes (table format, data structure)
- Whether collection batches run sequentially or the skill indicates parallel intent

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DETECT-01 | Skill detects AOS8 connectivity at session start and announces API mode (live data) vs paste-mode fallback | `health()` is already in skill's `tools:` frontmatter; pattern proven in `infrastructure-health-check.md` Step 1; `health(platform="aos8")` form supported (per `_probe_*` aggregator pattern in v1.0). |
| COLLECT-01 | MD hierarchy + per-node effective config via `aos8_get_md_hierarchy()` + `aos8_get_effective_config()` | Both tools confirmed at `platforms/aos8/tools/differentiators.py` lines 49 + 72. `aos8_get_effective_config(object_name, config_path="/md")` requires a per-call `object_name` argument — skill prose must guide the AI to iterate over object types or pick a representative one. |
| COLLECT-02 | Full AP inventory (model, MAC, serial, IP mode, AP group) via `aos8_get_ap_database()` | Tool confirmed at `platforms/aos8/tools/health.py:46` (NOTE: lives in `health.py`, **not** `wlan.py` as CONTEXT.md states). Signature: `aos8_get_ap_database(ctx, config_path="/md")`. |
| COLLECT-03 | Cluster state + running-config + local-user db via `aos8_get_cluster_state()` + `aos8_show_command()` passthrough | `aos8_get_cluster_state` at `differentiators.py:157`; `aos8_show_command(ctx, command)` at `troubleshooting.py:82` accepts arbitrary `show ...` strings (e.g., `show running-config`, `show local-user db`, `show inventory`, `show port status`, `show trunk`). |
| COLLECT-04 | Client baseline / BSS / active APs / AP wired ports via `aos8_get_clients()` / `aos8_get_bss_table()` / `aos8_get_active_aps()` / `aos8_get_ap_wired_ports()` | Locations: `aos8_get_clients` at `clients.py:31`; `aos8_get_bss_table` at `health.py:117` (NOTE: in `health.py`, not `wlan.py`); `aos8_get_active_aps` at `health.py:66` (NOTE: in `health.py`, not `clients.py`); `aos8_get_ap_wired_ports` at `differentiators.py:201` — requires a per-call `ap_name` argument (skill must iterate the AP list from Batch 2). |

## Standard Stack

This is a markdown-only phase. No package installation or version bumps. The "stack" is FastMCP's skill discovery + the existing AOS8 platform module shipped in v1.0.

| Asset | Location | Purpose |
|-------|----------|---------|
| Skill engine | `src/hpe_networking_mcp/skills/_engine.py` | Loads `*.md` skills with YAML frontmatter at startup, exposes via `skills_list()` / `skills_load()` |
| Skill file (target) | `src/hpe_networking_mcp/skills/aos-migration-readiness.md` | The single file edited in Phase 10 |
| Skill regression test | `tests/unit/test_skill_tool_references.py` | Validates every platform-prefixed tool name in skill bodies/frontmatter resolves to a real registered tool |
| Established detection pattern | `src/hpe_networking_mcp/skills/infrastructure-health-check.md` | The Step 1 `health()` probe + per-platform branching is the canonical reference |
| AOS8 tool catalog | `src/hpe_networking_mcp/platforms/aos8/tools/` | All required tools shipped in v1.0; no new tools needed |

**No installations:** Phase 10 changes no `pyproject.toml`, `uv.lock`, or Docker artifacts.

## Architecture Patterns

### Pattern 1: Frontmatter as the Tool Contract

Every tool referenced in the skill body must appear in the `tools:` list in YAML frontmatter. The skill engine and the regression test both rely on this.

**Current frontmatter** (line 23 of `aos-migration-readiness.md`):

```yaml
tools: [health, central_get_scope_tree, central_get_devices, central_get_aps, central_get_sites, central_get_site_name_id_mapping, central_recommend_firmware, central_get_config_assignments, central_get_server_groups, central_get_wlan_profiles, central_get_roles, central_get_named_vlans, clearpass_get_network_devices, clearpass_get_device_groups, clearpass_get_server_certificates, clearpass_get_local_users, greenlake_get_subscriptions, greenlake_get_workspace, greenlake_get_devices]
```

**Phase 10 addition (8 AOS8 tools):**

```yaml
tools: [health, ..., aos8_get_md_hierarchy, aos8_get_effective_config, aos8_get_ap_database, aos8_get_cluster_state, aos8_show_command, aos8_get_clients, aos8_get_bss_table, aos8_get_active_aps, aos8_get_ap_wired_ports]
```

`health` is already present — no change needed for DETECT-01.

### Pattern 2: Detect-and-Branch via `health()`

Reference: `infrastructure-health-check.md` Step 1 (lines 31-40). The pattern is:

1. Call `health()` (no platform arg) at session start.
2. Check the per-platform `status` key.
3. Branch prose conditionally — if `aos8.status == "ok"`, announce live mode and use API path; if `unavailable` or `degraded`, fall through silently to paste path.

The skill is **prose-driven** — the LLM reads the markdown and decides what to do. The skill must be unambiguous about "if AOS8 is reachable, do X; otherwise, proceed exactly as before."

### Pattern 3: Conditional Branching Inside Stage 1

Stage 1 currently has three top-level conditional sections: `If source = aos8`, `If source = aos6`, `If source = iap` (lines 80-134 of the skill). Phase 10 replaces only the `aos8` block. The structural pattern is:

```markdown
#### If source = `aos8` (anchor: VSG §1671-§1873)

**If AOS8 API mode was announced at session start (live data path):**
[four batch narration with tool calls + per-batch hybrid fallback]

**Otherwise (paste fallback — AOS8 not reachable, treated like AOS6/IAP source):**
[existing 16-row table — KEPT as the fallback path]
```

**Open question for planner:** D-04 says the 16-command paste table is "removed entirely" from Stage 1. But D-02 says paste-driven flow is unchanged when AOS8 is not reachable. Resolution: when AOS8 is unreachable, the existing paste flow must still work — so the table cannot be deleted outright. The most likely interpretation is: the table is removed from the live-mode path but a fallback paste section remains for the unreachable case (or the skill instructs the AI to use the AOS6 procedure as a baseline, since they overlap heavily). **This contradiction in CONTEXT.md should be flagged in PLAN-CHECK.** The planner should pick one interpretation and document it.

### Pattern 4: Per-Batch Hybrid Fallback

For each batch, the skill prose should explicitly state the fallback. Example shape:

```markdown
**Batch 2 (COLLECT-02) — AP inventory**
Call `aos8_get_ap_database()`. Expected: list of APs with model, MAC, serial, IP mode, AP group, ap-name.
On error: ask the operator to run `show ap database long` and paste the output. Continue with Batches 3-4; do not abort.
```

The fallback request is **exact CLI command text**, no VSG anchors, no full-table reprint (per D-08).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Detect AOS8 reachability | A new tool, custom probe, or `aos8_health` reference | `health()` (already in frontmatter) | The aggregator probes every enabled platform including AOS8 in one call; matches the established `infrastructure-health-check.md` pattern. |
| Iterate node hierarchy | A custom traversal helper | `aos8_get_md_hierarchy()` returns the tree in one call; the AI can iterate output | The tool already returns the hierarchy; node-by-node `effective_config` calls follow naturally. |
| Run arbitrary `show` commands | A new wrapper or paste fallback for each one | `aos8_show_command(command="show running-config")` | Already a generic CLI passthrough; covers running-config, local-user db, inventory, port status, trunk in one tool. |
| New AOS8 tool entry in `tools:` frontmatter | Inventing a tool name not in the catalog | Verify each name against `@tool(name=...)` decorators in `platforms/aos8/tools/*.py` | The regression test enforces this; CI fails otherwise. |

**Key insight:** Every required tool exists in v1.0. The phase is *purely composition* — wiring catalog tools into skill prose.

## Common Pitfalls

### Pitfall 1: Tool location drift — `aos8_get_ap_database`, `aos8_get_bss_table`, `aos8_get_active_aps` are in `health.py`, NOT where CONTEXT.md says

**What goes wrong:** CONTEXT.md (lines 73-74) lists `aos8_get_ap_database` and `aos8_get_bss_table` under `wlan.py`, and `aos8_get_active_aps` under `clients.py`. Searching the codebase confirms all three are actually in `platforms/aos8/tools/health.py`.
**Why it happens:** The v1.0 file organization grouped them by *operational use* (health monitoring), not by topic-name keyword.
**How to avoid:** When PLAN.md or task descriptions reference tool locations, verify against the actual `@tool(name=...)` decorator location, not the CONTEXT.md description.
**Warning signs:** Implementation tasks that "edit `wlan.py` to expose tool X" — if X is `aos8_get_ap_database`, the task is wrong.

### Pitfall 2: Skill regression test regex doesn't catch `aos8_*` names

**What goes wrong:** `tests/unit/test_skill_tool_references.py` line 39 uses regex `\b(mist|central|greenlake|clearpass|apstra|axis)_[a-z_][a-z_0-9]+\b` — note `aos8` is **not** in the alternation. Phase 10 adds `aos8_*` references to the skill body and frontmatter, but the test will silently *not enforce* that those names resolve to real tools.
**Why it happens:** The test was written before AOS8 platform existed; nobody updated the regex when v1.0 shipped.
**How to avoid:** Phase 10 is technically not blocked by this — but the planner must (1) note that name-correctness is on the author, not enforced by CI for AOS8 names, and (2) call out for Phase 13 (QUALITY-03) that the regex must be extended to include `aos8`.
**Warning signs:** A typo'd `aos8_get_ap_databse` in the skill prose passes CI but fails at runtime with "tool not found."

### Pitfall 3: The "remove entirely" / "preserved" contradiction (D-04 vs D-02)

**What goes wrong:** D-04 says "16-command AOS8 paste table is removed entirely," but D-02 says "if AOS8 is NOT reachable, the skill proceeds silently into the existing Stage 0 interview — paste-driven flow is unchanged."
**Why it happens:** D-04 is talking about Stage 1 in live-mode; D-02 is talking about overall flow when AOS8 is unreachable. The two are not contradictory if the AOS8-not-reachable case falls back to the AOS6 procedure (which structurally overlaps for cluster + non-Conductor environments). But this resolution is implicit and must be made explicit.
**How to avoid:** PLAN-CHECK should force the planner to declare: when AOS8 is unreachable AND `source=aos8` is selected in Stage 0, what data-collection prose runs? Either (a) a vestigial paste table is retained inside the AOS8 Stage 1 section, or (b) the skill explicitly redirects to the AOS6 paste procedure for that combination, or (c) the skill demands the operator either fix AOS8 reachability or pick a different source.
**Warning signs:** Plan tasks that delete the entire `If source = aos8` block without specifying the unreachable fallback path.

### Pitfall 4: `aos8_get_effective_config` requires `object_name` per call

**What goes wrong:** `aos8_get_effective_config(ctx, object_name, config_path="/md")` does not return a "full effective config dump" — it returns one named object's resolved config (e.g., `ssid_prof`, `virtual_ap`, `ap_sys_prof`). Skill prose that says "call `aos8_get_effective_config()` to get the full per-node config" will produce an LLM error or an empty-arg call.
**Why it happens:** The original CLI command `show configuration effective detail <node>` dumps everything; the API tool is object-scoped.
**How to avoid:** Skill prose for Batch 1 must instruct the AI to either (a) iterate a known list of object names that matter for migration readiness (e.g., `ap_sys_prof`, `virtual_ap`, `ssid_prof`, `aaa_server_group`, `aaa_prof`), or (b) call `aos8_show_command(command="show configuration effective detail")` as a passthrough alternative.
**Warning signs:** Task wording like "fetch effective config" without an explicit object-iteration plan.

### Pitfall 5: `aos8_get_ap_wired_ports` requires `ap_name` per call

**What goes wrong:** Like `effective_config`, this tool is per-AP, not bulk. Batch 4 prose must direct the AI to iterate the AP list collected in Batch 2 and call once per AP — or pick a representative subset for the report (which may be acceptable for a baseline).
**How to avoid:** Skill prose explicitly mentions iteration or sampling.

### Pitfall 6: Tools must be in `lifespan_context["aos8_client"]` — silent skip if AOS8 not initialized

**What goes wrong:** Each AOS8 tool reads `ctx.lifespan_context["aos8_client"]`. If AOS8 isn't initialized (no secrets), `client` is `None` and the tool errors. This is the same condition `health()` reports as `unavailable` — so the detection step naturally guards the live path.
**How to avoid:** Don't try to call AOS8 tools in the unreachable branch. The `health()` guard at session start is sufficient.

### Pitfall 7: AOS6/IAP regression risk

**What goes wrong:** Editing the file globally (search-and-replace, big block reorganization) accidentally drops or modifies AOS6/IAP paste tables.
**How to avoid:** Constrain the edit to the `If source = aos8` section only (lines ~80-104 of the current skill). Leave AOS6 (lines 105-118) and IAP (lines 119-134) untouched. Verify with a focused diff before commit.
**Warning signs:** PR diff that touches AOS6 or IAP lines.

## Code Examples

### Example: Detection step (model after `infrastructure-health-check.md`)

```markdown
### Stage -1 — Session-start AOS8 detection

Before beginning the Stage 0 operator interview, call `health()` once.

If `aos8.status == "ok"`:
> Announce: "AOS8 API mode — live data. Stage 1 collection will run via API; no CLI paste required for the AOS8 source path."

Otherwise: proceed silently to Stage 0 with no announcement and no behavior change. The paste-driven flow is unchanged for AOS6, IAP, or unreachable-AOS8 sessions.
```

### Example: Batch narration (Batch 2 — COLLECT-02)

```markdown
**Batch 2 — Full AP inventory (COLLECT-02)**

Call `aos8_get_ap_database()`. Expected output: list of APs with `model`, `mac`, `serial`, `ip_mode` (DHCP / static), `group`, `ap_name`. This replaces `show ap database long`.

On tool error: tell the operator: "API call to retrieve the AP database failed. Please run `show ap database long` on the Mobility Conductor and paste the output." Then proceed to Batch 3.
```

### Example: Frontmatter update

```yaml
platforms: [central, aos8]
tools: [health, ..., aos8_get_md_hierarchy, aos8_get_effective_config, aos8_get_ap_database, aos8_get_cluster_state, aos8_show_command, aos8_get_clients, aos8_get_bss_table, aos8_get_active_aps, aos8_get_ap_wired_ports]
```

(Note: `platforms:` update to include `aos8` is technically QUALITY-02 in Phase 13, but doing it now costs nothing and keeps frontmatter consistent. Planner should decide whether to do it now or defer per scope.)

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Operator pastes 16 CLI command outputs (~5-15 min collection) | API mode: skill calls 8 AOS8 tools (~30 sec, no paste needed) | v1.1 (this milestone) | Drops the data-collection burden from the operator and makes the workflow chat-native. |

**Deprecated/outdated:**
- The 16-row AOS8 paste table is being retired from the live path. The CLI commands themselves remain documented in v1.0 `TOOLS.md` / `INSTRUCTIONS.md` for any operator who prefers the paste flow or for unreachable-AOS8 sessions.

## Open Questions

1. **D-04 vs D-02 fallback path**
   - What we know: When AOS8 is unreachable AND source=aos8, the operator-paste flow must still work somehow.
   - What's unclear: Whether the skill keeps a vestigial paste table inside the `If source = aos8` section, or redirects to the AOS6 path, or stops with an error.
   - Recommendation: Planner picks one and documents it in PLAN.md. Suggested resolution: keep a brief paste fallback section labeled "AOS8 source — paste fallback (AOS8 not reachable)" with the existing 16-command table as the operator's reference. This honors both D-04 (the live path is API-only) and D-02 (paste flow unchanged when unreachable).

2. **Should `platforms: [central]` become `platforms: [central, aos8]` in Phase 10 or wait for Phase 13 (QUALITY-02)?**
   - Recommendation: Defer to Phase 13 to keep Phase 10 scope tight. The phase boundary in REQUIREMENTS.md explicitly assigns QUALITY-02 to Phase 13.

3. **Should `aos8_get_effective_config` be replaced by `aos8_show_command(command="show configuration effective detail <node>")` for Batch 1?**
   - What we know: The API tool is object-scoped (one object per call); the show-command passthrough returns a full text dump like the CLI.
   - What's unclear: Whether the downstream parsing in Stage 2/3 expects structured JSON (favoring `aos8_get_effective_config`) or text (favoring `aos8_show_command`).
   - Recommendation: Planner should specify both — call `aos8_get_effective_config` per object (for structured parsing in Phase 11 RULES) and optionally a `aos8_show_command` dump for human-readable inclusion. Or pick one and document the trade.

## Environment Availability

This phase has no external dependencies beyond the existing v1.0 codebase and test framework.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python 3.12 | All work | ✓ | 3.12+ | — |
| uv | Local test runs | ✓ | Latest | — |
| pytest 8.4+ | Skill regression test | ✓ | Per `pyproject.toml` | — |
| AOS8 tool modules | Tool name validation | ✓ | v1.0 (shipped 2026-04-29) | — |

**Missing dependencies with no fallback:** None.
**Missing dependencies with fallback:** None.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest 8.4+ with pytest-asyncio (asyncio_mode=auto) |
| Config file | `pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `uv run pytest tests/unit/test_skill_tool_references.py -x` |
| Full suite command | `uv run pytest tests/ -q` |
| Lint | `uv run ruff check .` (no Python changes here, but skill engine reads the file at startup) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DETECT-01 | Skill announces API mode at session start when AOS8 reachable | manual / smoke | Manual: load skill via `skills_load("aos-migration-readiness")` with AOS8 secrets configured; verify announcement appears | manual-only — LLM-prose behavior, not unit-testable |
| DETECT-01 (negative) | Skill is silent and falls through to paste when AOS8 unreachable | manual / smoke | Manual: same load with AOS8 secrets unset; verify no announcement | manual-only |
| COLLECT-01..04 | Required tool names appear in skill body and resolve to real tools | unit | `uv run pytest tests/unit/test_skill_tool_references.py -x` | ✅ exists; ⚠ regex doesn't match `aos8_*` (Pitfall 2). Phase 10 should add a parametrized assertion that each new `aos8_*` name is in the catalog, OR document the gap for Phase 13. |
| COLLECT-01..04 | Tool names in `tools:` frontmatter are valid catalog entries | unit | Same as above (the test inspects frontmatter too) | ✅ — but note same regex limitation |
| AOS6/IAP regression | Existing AOS6 + IAP paste paths unchanged | unit + manual | `uv run pytest tests/unit/test_skill_tool_references.py` (catches frontmatter regressions) + manual diff review | ✅ |
| Skill loads without parse errors | YAML frontmatter + body still parse cleanly | unit | Existing skill engine smoke test in `tests/unit/test_skills_*.py` if present, or run server startup against the file | check for existing test in Wave 0 |

### Sampling Rate
- **Per task commit:** `uv run pytest tests/unit/test_skill_tool_references.py -x && uv run ruff check .`
- **Per wave merge:** `uv run pytest tests/ -q`
- **Phase gate:** Full suite green + manual smoke test of skill load with AOS8 reachable + with AOS8 unreachable, before `/gsd:verify-work`.

### Wave 0 Gaps
- [ ] **Extend `tests/unit/test_skill_tool_references.py` regex to include `aos8`** — strictly required for COLLECT requirements to be CI-enforced. (Alternatively defer to Phase 13 QUALITY-03 and document the risk in Phase 10 PLAN.md.)
- [ ] **Confirm existence of skill-load smoke test** — if no test currently asserts `aos-migration-readiness.md` parses cleanly via the skill engine, add one. Search `tests/unit/test_skills_*.py` and `tests/unit/test_skill_engine*.py`.
- [ ] **Decide on a parametrized "AOS8 tool name in catalog" assertion** specific to this skill — belt-and-suspenders alongside the regex fix.

*(If Phase 13 owns the regex fix and the planner accepts that risk for Phase 10, the only Wave 0 task is confirming the skill-load smoke test exists or adding one.)*

## Project Constraints (from CLAUDE.md)

The project's `CLAUDE.md` (and the `hpe-networking-mcp/CLAUDE.md` sub-doc) directives applicable to this phase:

| Directive | Applies How |
|-----------|-------------|
| **Max 500 lines/file** | The current skill file is ~488 lines. Phase 10 adds a Stage -1 detection block (~10 lines), an API-mode Batch section (~30-40 lines), and may keep a fallback paste table. **Risk: file may exceed 500 lines.** Planner must check projected line count; if over, consider splitting the AOS8 fallback paste table into a separate referenced file or trimming verbose VSG anchors. |
| **No PUT/PATCH/DELETE; all mutations via POST** | N/A — Phase 10 is read-only (`@tool` annotations are `READ_ONLY`). |
| **UIDARUBA token reuse across calls** | Already handled by the v1.0 AOS8 client; skill prose does not need to address. |
| **Compatibility: don't break any existing platform module or test** | Skill regression test must pass. AOS6/IAP sections must be untouched. |
| **Style: max 500 lines/file, ruff lint + format** | Applies to the markdown file's line count; ruff doesn't lint markdown but the line-count rule is the project convention. |
| **Documentation Checklist (per `hpe-networking-mcp/CLAUDE.md`)** | Phase 10 changes a skill — TOOLS.md / INSTRUCTIONS.md may need a brief mention of "AOS8 source path is now API-driven in `aos-migration-readiness` skill." Planner decides scope; minimum is CHANGELOG.md entry. |
| **Branch strategy: `feature/*`, PR to main** | Per `.planning/config.json`: `branching_strategy: "none"` — overrides the CLAUDE.md branch-per-phase rule for this project. Single trunk OK. |
| **Pre-Push Checklist (run in Docker)** | Lint + format + mypy + pytest before commit. Phase 10 has no Python changes so mypy is a no-op, but the pytest run still needs to pass. |
| **No "claude code" in commit messages** | Standard convention. |

**Skill authoring constraints (project-specific):**
- Frontmatter `tools:` must list every platform-prefixed tool name in the skill body.
- Skill files live only in `src/hpe_networking_mcp/skills/`; the skill engine loads on startup.
- Skills are read by an LLM, not executed — prose ambiguity is a real bug class.

## Sources

### Primary (HIGH confidence)
- `src/hpe_networking_mcp/skills/aos-migration-readiness.md` (full file, 488 lines) — the file being edited
- `src/hpe_networking_mcp/skills/infrastructure-health-check.md` — canonical detect-and-branch pattern
- `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` — verified locations for `aos8_get_md_hierarchy` (line 49), `aos8_get_effective_config` (line 72), `aos8_get_cluster_state` (line 157), `aos8_get_ap_wired_ports` (line 201)
- `src/hpe_networking_mcp/platforms/aos8/tools/health.py` — verified locations for `aos8_get_ap_database` (line 46), `aos8_get_active_aps` (line 66), `aos8_get_bss_table` (line 117)
- `src/hpe_networking_mcp/platforms/aos8/tools/clients.py` — verified location for `aos8_get_clients` (line 31)
- `src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py` — verified location for `aos8_show_command` (line 82)
- `tests/unit/test_skill_tool_references.py` — regression test mechanics (and the regex gap)
- `.planning/phases/10-live-detection-collection/10-CONTEXT.md` — locked phase decisions
- `.planning/REQUIREMENTS.md` — DETECT-01, COLLECT-01..04 acceptance criteria
- `CLAUDE.md` (project root) — project conventions
- `hpe-networking-mcp/CLAUDE.md` — sub-project conventions (file size, doc checklist, pre-push)

### Secondary (MEDIUM confidence)
- *(none — all findings grounded in repo source)*

### Tertiary (LOW confidence)
- *(none)*

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — every required tool verified by file-line read
- Architecture (detect-and-branch): HIGH — pattern is one of the codebase's established skill idioms
- Pitfalls: HIGH — Pitfalls 1, 2, 3 are confirmed by direct source inspection (tool location vs CONTEXT.md mismatch, regex gap in regression test, D-04/D-02 contradiction in CONTEXT.md)

**Research date:** 2026-04-29
**Valid until:** 2026-05-29 (30 days — codebase stable, milestone v1.0 just shipped)
