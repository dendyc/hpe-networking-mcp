# Phase 4: Differentiator Tools - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-28
**Phase:** 04-differentiator-tools
**Areas discussed:** File structure, DIFF-09 depth, Plan count & sequencing, DIFF-02/03 API uncertainty

---

## File Structure

| Option | Description | Selected |
|--------|-------------|----------|
| One `differentiators.py` | All 9 DIFF tools in a single new file, ~250–300 lines | ✓ |
| Two files by domain | hierarchy.py + infrastructure.py | |
| Distribute into existing files | DIFF tools spread across health.py, troubleshooting.py, etc. | |

**User's choice:** One `differentiators.py`
**Notes:** Fits under 500-line limit; simpler to navigate; avoids splitting 9 tools across two files.

---

| Option | Description | Selected |
|--------|-------------|----------|
| New "differentiators" key | TOOLS["differentiators"] = [9 tool names] | ✓ |
| Bucket under existing keys | e.g., DIFF-04 under "health", DIFF-08 under "troubleshooting" | |

**User's choice:** New "differentiators" key
**Notes:** Keeps category semantically accurate and consistent with existing 5 category keys.

---

## DIFF-09 Depth

| Option | Description | Selected |
|--------|-------------|----------|
| Multi-call aggregation | 3–4 targeted API calls merged into one response dict | ✓ |
| Single show command | One show command, return cleaned output | |
| Delegate to Phase 6 prompt | Thin tool; prompt synthesizes in Phase 6 | |

**User's choice:** Multi-call aggregation
**Notes:** Mirrors Central's get_site_health pattern.

---

**Data points for aggregation (multiselect — all selected):**

| Data Point | Selected |
|------------|----------|
| AP counts (up/down) | ✓ |
| Client count | ✓ |
| Alarm summary (by severity) | ✓ |
| Firmware version | ✓ |

---

## Plan Count & Sequencing

| Option | Description | Selected |
|--------|-------------|----------|
| 3 plans: Wave 0 + impl + wire | Wave 0 scaffold → implement all 9 → wire __init__.py | ✓ |
| 2 plans: combined scaffold+impl + wire | Tests and impl together in Plan 1 | |
| 4 plans: Wave 0 + 2 impl groups + wire | Split impl into two groups of tools | |

**User's choice:** 3 plans
**Notes:** Preserves TDD red-baseline discipline; right-sized for 9 tools vs the 7 plans used in Phase 3's 26-tool scope.

---

## DIFF-02/03 API Uncertainty

| Option | Description | Selected |
|--------|-------------|----------|
| Show command fallback | Use show API if no config-object endpoint exists | ✓ |
| Stub with NotImplemented error | Return error if no clean endpoint found | |
| Drop the tool from scope | Remove from Phase 4 if no viable path | |

**User's choice:** Show command fallback
**Notes:** Researcher to find best show command for each and document in plan.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Parameterized object_name | Tool accepts object_name param for any config object | ✓ |
| Fixed SSID/VAP scope only | DIFF-02 only resolves SSID profiles and virtual APs | |
| Full config dump | Return entire effective running-config for the scope | |

**User's choice:** Parameterized object_name
**Notes:** Mirrors aos8_show_command's flexibility. Researcher confirms which objects support inheritance resolution.

---

## Claude's Discretion

- Exact AOS8 REST endpoint paths or show commands for DIFF-01..09
- JSON fixture file structure for test mocks
- Whether DIFF-05..08 use show commands or config-object endpoints
- Which DIFF tools need `config_path` parameter (researcher to confirm per tool)

## Deferred Ideas

None — discussion stayed within phase scope.
