# Phase 6: Guided Prompts & Documentation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-28
**Phase:** 06-guided-prompts-documentation
**Areas discussed:** Prompt detail level, Prompt coverage, INSTRUCTIONS.md scope, Plan sequencing

---

## Prompt Detail Level

| Option | Description | Selected |
|--------|-------------|----------|
| Fully explicit | Numbered steps with specific `aos8_*` tool names and key params — matches Central's style | ✓ |
| Hybrid | Explicit tool names, no param examples — shorter prompts | |
| High-level workflow | Workflow intent without specific tool names | |

**User's choice:** Fully explicit
**Notes:** Matches Central's approach; guides AI models most reliably.

| Option | Description | Selected |
|--------|-------------|----------|
| Summary + next steps | End each prompt with findings summary and recommended operator actions | ✓ |
| Summary only | Just summarize findings | |
| You decide | Claude's discretion per prompt | |

**User's choice:** Summary + next steps

---

## Prompt Coverage

| Option | Description | Selected |
|--------|-------------|----------|
| All 9 | Implement all PROMPT-01..09 per requirements | ✓ |
| Core 7 (skip 07 + 08) | Defer aos8_client_flood and aos8_compare_md_config | |
| You decide | Claude's discretion based on confidence | |

**User's choice:** All 9

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — contextual prompts take params | mac_address, ap_name, config_path params where useful | ✓ |
| No — all parameterless | Generic workflows, operator provides specifics conversationally | |
| You decide per prompt | Claude decides which benefit from params | |

**User's choice:** Yes — contextual prompts take parameters

---

## INSTRUCTIONS.md Scope

| Option | Description | Selected |
|--------|-------------|----------|
| 4 required topics only | config_path, write_memory, show_command, Conductor-vs-standalone | |
| 4 topics + guided prompt index | Same 4 plus a section listing all 9 prompts with purpose and params | ✓ |
| Full operator guide | 4 topics + Docker setup, secrets, examples, troubleshooting | |

**User's choice:** 4 topics + guided prompt index
**Notes:** The guided prompt index acts as a quick-reference card for operators.

---

## Plan Sequencing

| Option | Description | Selected |
|--------|-------------|----------|
| 2-plan structure | Plan 06-01: prompts + smoke test. Plan 06-02: all docs | ✓ |
| 3-plan TDD structure | Wave 0 red tests → implementation → docs | |
| 1-plan structure | All prompts + all docs in one plan | |

**User's choice:** 2-plan structure

| Option | Description | Selected |
|--------|-------------|----------|
| Registration smoke test only | 9 prompts register and return non-empty strings | ✓ |
| Per-prompt content assertions | Test that returned strings contain specific tool names | |

**User's choice:** Registration smoke test only
**Notes:** Content assertions are brittle; smoke test proves wiring.

---

## Claude's Discretion

- Exact parameter names and defaults for parameterized prompts
- Precise version number for pyproject.toml bump
- Order of sections in INSTRUCTIONS.md
- Whether README.md already has an AOS8 row stub

## Deferred Ideas

None.
