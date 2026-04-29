# Phase 12: Central Enrichment & Cutover Validation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-29
**Phase:** 12-central-enrichment-cutover-validation
**Areas discussed:** Stage 4 sub-path placement, Conflict detection severity, Firmware call strategy, Stage 5 live-mode structure, ENRICH coverage scope, Conflict listing granularity, AP count snapshot placement, CUTOVER-02 data source

---

## Stage 4 Sub-Path Placement

| Option | Description | Selected |
|--------|-------------|----------|
| Sub-path block above table | Self-contained `##### AOS8 live-mode sub-path` above A1–A13 table. Existing table stays as paste fallback. Consistent with Stage 3 pattern. | ✓ |
| Annotate rows inline | Modify A4/A7/A8/A9/A13 rows directly with conditional live-mode override notes. | |
| Separate Stage 4b section | New `### Stage 4b — AOS8 live enrichment` section after Stage 4. | |

**User's choice:** Sub-path block above table
**Notes:** Consistent with Stage 3 pattern; keeps paste fallback intact.

---

## Conflict Detection Severity (ENRICH-03 / ENRICH-04)

| Option | Description | Selected |
|--------|-------------|----------|
| REGRESSION | Naming conflicts block a clean migration — counts as a blocker in the executive summary verdict. | ✓ |
| DRIFT | Needs attention but not a hard blocker; operator can resolve during migration window. | |
| INFO | Surface for awareness only — no severity escalation from current A7/A8/A9 rows. | |

**User's choice:** REGRESSION
**Notes:** Upgrades severity from the current INFO on A7/A8/A9.

---

## `central_recommend_firmware()` Call Strategy (ENRICH-02)

| Option | Description | Selected |
|--------|-------------|----------|
| Once, extract per-model | Call once with no filter; AI extracts per-model rows from fleet response cross-referenced against AOS8 AP models. | ✓ |
| Once per distinct AP model | Call once per distinct AP model from AOS8 inventory — N calls for N models. | |

**User's choice:** Once, extract per-model
**Notes:** One call for any fleet size; Central response already grouped by model.

---

## Stage 5 Live-Mode Structure (CUTOVER-01..03)

| Option | Description | Selected |
|--------|-------------|----------|
| Sub-path block before table | `##### AOS8 live-mode sub-path` before Phase 0–8 cutover table. Existing table stays intact. Mirrors Stage 3 pattern. | ✓ |
| Annotate Phase rows inline | Modify Phase 0 and Phase 1 rows directly with conditional live-mode override notes. | |

**User's choice:** Sub-path block before table
**Notes:** Mirrors Stage 3 and Stage 4 patterns exactly.

---

## ENRICH Coverage Scope

| Option | Description | Selected |
|--------|-------------|----------|
| All three (A7 + A8 + A9) | Check SSID names (Batch 4), role names and VLAN IDs (Batch 1 effective config) against all three Central tools. | ✓ |
| Only confirmed Stage 1 data | Only emit findings where Stage 1 data is confirmed present — avoids partial comparisons. | |

**User's choice:** All three (A7 + A8 + A9)
**Notes:** Comprehensive; all data points are available from Stage 1 batches.

---

## Conflict Listing Granularity

| Option | Description | Selected |
|--------|-------------|----------|
| List each item individually | One finding per conflict with name/ID and source tool — matches RULES-04 per-AP pattern. | ✓ |
| Summarize by count | "3 SSID conflicts, 2 role conflicts — see full list below." Shorter but less actionable. | |

**User's choice:** List each item individually
**Notes:** Matches RULES-04 style; gives SE actionable specifics to resolve each conflict.

---

## AP Count Snapshot Placement (CUTOVER-03)

| Option | Description | Selected |
|--------|-------------|----------|
| Stage 5 sub-path as INFO finding | Lives alongside cluster health and firmware check in Stage 5 live-mode block. | ✓ |
| Stage 4 A4 enrichment | Merged with ENRICH-01 AP count gap — both from `aos8_get_ap_database()`. | |

**User's choice:** Stage 5 INFO finding
**Notes:** Keeps cutover snapshot in the cutover readiness context; clean separation from enrichment gap.

---

## CUTOVER-02 Firmware Version Source

| Option | Description | Selected |
|--------|-------------|----------|
| Fresh show version call in Stage 5 | `aos8_show_command(command='show version')` called fresh in Stage 5 sub-path. Unambiguous firmware version. | ✓ |
| Reuse show inventory from Batch 3 | Extract firmware version from Stage 1 Batch 3 `show inventory` context if available. | |
| You decide | Leave choice to Claude when writing prose — whichever tool most reliably surfaces firmware version. | |

**User's choice:** Fresh show version call in Stage 5
**Notes:** `show version` is the canonical AOS8 firmware version source; avoids relying on `show inventory` format variability.

---

## Claude's Discretion

- Exact wording for Stage 4 and Stage 5 sub-path headers and intro lines
- Whether Stage 4 sub-path emits a brief "enrichment complete" summary
- Partial failure handling in Stage 4 (Central tool failures)
- Format of the per-model firmware recommendation table

## Deferred Ideas

None — discussion stayed within phase scope.
