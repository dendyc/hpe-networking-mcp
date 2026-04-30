# Milestones

## v1.1 AOS8-Powered Migration Readiness (Shipped: 2026-04-30)

**Phases completed:** 4 phases (10-13), 6 plans  
**Timeline:** 2026-04-29 → 2026-04-30 (2 days)  
**Test suite:** 790 unit tests passing (790/790)  
**Scope:** Skill-only milestone — no Python code changes  

**Key accomplishments:**

- CI enforces `aos8_*` tool name correctness — skill-tool-reference regex extended with `|aos8` alternation, 9-tool parametrized catalog assertion added (Phase 10-01)
- AOS8 `aos-migration-readiness` skill rewritten with session-start `health()` detection gate + four-batch live API collection path (9 AOS8 tools replacing 16 CLI paste commands); AOS6/IAP paste flows preserved byte-for-byte (Phase 10-02)
- AOS8 live-mode rule evaluation (RULES-01..04) layered into Stage 3 — VRRP VIP check, ARM/radio profile detection, static AP IP detection as REGRESSION/DRIFT findings; RULES-03 local-user cross-check deferred to Stage 4 A11 (Phase 11-01)
- Stage 4 AOS8 live-mode Central enrichment sub-path (ENRICH-01..04) — AP count gap, per-model AOS10 firmware recommendations, SSID/role/named-VLAN naming conflicts surfaced as REGRESSION findings without operator paste (Phase 12-01)
- Stage 5 AOS8 live-mode cutover prerequisites sub-path (CUTOVER-01..03) — cluster L2-connected health, fresh `show version` firmware floor check (8.10.0.12 / 8.12.0.1), pre-cutover AP-count baseline (Phase 12-02)
- Executive summary paragraph (GO/BLOCKED/PARTIAL verdict + REGRESSION/DRIFT/INFO counts + SE-ready sentence template), output hygiene rules (4 prohibitions), AOS8 PARTIAL decision-matrix row, `platforms:[central, aos8]` frontmatter — regression test 1/1 + full suite 790/790 (Phase 13-01)

**Known Tech Debt (carried forward):**
- DETECT-01 runtime behavior (Scenario A — AOS8 live-mode announcement) unverified — no live AOS8 environment; prose is mechanically correct
- `aos-migration-readiness.md` is ~693 lines, exceeding the 500-line soft limit — refactoring deferred to v1.2
- Nyquist VALIDATION.md status=draft for Phases 11, 12, 13 — markdown-only phases, no new unit tests added

**Archive:**

- [v1.1-ROADMAP.md](milestones/v1.1-ROADMAP.md)
- [v1.1-REQUIREMENTS.md](milestones/v1.1-REQUIREMENTS.md)
- [v1.1-MILESTONE-AUDIT.md](milestones/v1.1-MILESTONE-AUDIT.md)

---

## v1.0 AOS8 MVP (Shipped: 2026-04-29)

**Phases completed:** 9 phases, 26 plans  
**Timeline:** 2026-04-27 → 2026-04-29 (2 days)  
**Test suite:** 766 unit tests passing across 7 platforms  
**Code:** ~148K total Python LOC (project); AOS8 module adds ~3K LOC  

**Key accomplishments:**

- Delivered AOS8/Mobility Conductor as the 7th platform module — secrets, write-gate, Docker Compose wiring, graceful auto-disable when unconfigured (Phase 1)
- `AOS8Client` with UIDARUBA token reuse, asyncio.Lock-serialized refresh, lazy login, token masking in all log paths (Phase 2)
- 26 read tools across 5 categories: controller/AP inventory, client lookup by MAC/IP/username, alerts/audit trail, WLAN/VAP/role config, troubleshooting suite (ping, traceroute, show_command passthrough) (Phase 3)
- 9 AOS8-specific differentiator tools: MD hierarchy tree, effective config post-inheritance, pending changes, RF neighbors, cluster state, air monitors, AP wired ports, IPsec tunnels, unified MD health check (Phases 4/7/8)
- 12 gated write tools behind `ENABLE_AOS8_WRITE_TOOLS` with elicitation confirmation: full SSID/VAP/AP-group/role/VLAN/AAA/ACL/netdest lifecycle + client disconnect + AP reboot + explicit write_memory (Phase 5)
- 9 guided workflow prompts (triage_client, triage_ap, health_check, audit_change, rf_analysis, wlan_review, client_flood, compare_md_config, pre_change_check) + full operator documentation (INSTRUCTIONS.md, README, TOOLS.md, CHANGELOG) (Phase 6)
- 766-test suite with UIDARUBA token-leak detection, DIFF production bug fix (raw httpx.Response → parsed JSON), Phase 4 planning closure, and non-regression across all 6 existing platforms (Phases 7–9)

**Known Tech Debt (carried forward):**

- `docker-compose.yml` `ENABLE_AOS8_WRITE_TOOLS` defaults to `:-true` (permissive, consistent with all other platforms but contradicts documented default-off)
- `test_aos8_init.py` `EXPECTED_CATEGORY_COUNTS` missing `troubleshooting: 7` key — global count of 47 still guards regressions
- REQUIREMENTS.md footer count "71/71" is stale (actual: 82); Nyquist VALIDATION.md incomplete for 8 of 9 phases

**Archive:**

- [v1.0-ROADMAP.md](milestones/v1.0-ROADMAP.md)
- [v1.0-REQUIREMENTS.md](milestones/v1.0-REQUIREMENTS.md)
- [v1.0-MILESTONE-AUDIT.md](milestones/v1.0-MILESTONE-AUDIT.md)

---
