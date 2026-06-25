---
title: "Session 2026-06-25 — Agentic-loop upgrade arc (PRs #12–#18)"
type: source
created: 2026-06-25
last_updated: 2026-06-25
sources: []
tags: [agentic-loop, hooks, skills, progress-json, tdd, writing-plans, upgrade-arc]
---

# Session 2026-06-25 — Agentic-loop upgrade arc (PRs #12–#18)

A sequenced six-spec hardening of the agentic-loop skill, plus a tool-agnostic cleanup. All merged to `main` before 2026-06-25. The arc is the first body of work to touch every layer of the loop simultaneously: the skill prose (phases), the Stop hooks (mechanical enforcement), the shared hook lib, and two new coderails-owned skills.

## PRs

| PR | Merge SHA | Spec | Description |
|---|---|---|---|
| #12 | `bf6695c` (squash) | A | Clean-migration discipline |
| #13 | `07a9897` | C1 | `progress.json` lifecycle + presence/ownership guard |
| #14 | `50a0b10` | C2 | Declaration-based anti-stall guard |
| #15 | `bcf9305` | B | Slim the skill |
| #16 | `35dea0b` | D | Vendored TDD construction seam |
| #17 | `2d05d14` | E | Spec→plan artifact discipline |
| #18 | `8242f7f` (squash) | — | Tool-agnostic prod-log examples (drop `tsh ssh`) |

## Primary spec docs (design authority)

All under `coderails/docs/superpowers/specs/`:
- `2026-06-24-clean-migration-discipline-design.md` (Spec A)
- `2026-06-24-c1-progress-json-lifecycle-design.md` (Spec C1)
- `2026-06-24-c2-anti-stall-guard-design.md` (Spec C2)
- `2026-06-24-b-slim-skill-design.md` (Spec B)
- `2026-06-24-d-construction-seam-design.md` (Spec D)
- `2026-06-24-e-plan-artifact-design.md` (Spec E)

Plans under `coderails/docs/superpowers/plans/` (one per spec).

## Files created by the arc

| File | Spec |
|---|---|
| `hooks/scripts/lib/agentic_loop_path.sh` | C1 |
| `hooks/scripts/lib/loop_state_common.sh` | C2 |
| `hooks/scripts/loop_state_guard.sh` | C1 |
| `hooks/scripts/loop_stall_guard.sh` | C2 |
| `skills/test-driven-development/SKILL.md` | D |
| `skills/test-driven-development/testing-anti-patterns.md` | D |
| `skills/writing-plans/SKILL.md` | E |
| `skills/writing-plans/plan-anti-patterns.md` | E |

## Files substantially modified

| File | Specs |
|---|---|
| `skills/agentic-loop/SKILL.md` | A, C1, C2, B, D, E |
| `hooks/hooks.json` | C1, C2 |
| `install.sh` | C1, C2 |

## Key decisions

**Spec A** — Disposition fork (clean-break vs preserve-compat) must be resolved explicitly before replacement work starts, not defaulted silently. The independent code-simplifier reviewer (not the worker) is the enforcement gate. A "0 violations" metric must be distinguished from "no record found" (the latter = audit failure).

**Spec C1** — The model must NEVER compute the `progress.json` path. The hook is the sole path authority via `agentic_loop_path.sh`. Structured jq `tool_use` detection (never text grep) prevents maintainer self-tripping.

**Spec C2** — Text-heuristic stall detection was rejected because it cannot separate a real question from an unauthorised yield. Declaration-based design: the model emits a vocab-checked `LOOP-STOP: <category>` tag; the hook checks presence + category only. Shared lib avoids any drift in "active loop" definition between C1 and C2.

**Spec B** — Phases 7/8 (corporate stack residue) collapsed to a generic stub. 16 war stories compressed to one-clause `Past failure:` tags. Six no-touch regions pinned; the byte-diff is the primary gate (token greps are necessary-not-sufficient).

**Spec D** — REVERSED Spec A's "reference not vendor" position. The dep on `superpowers:test-driven-development` would break coderails' self-contained-zip property. Vendor as `coderails:test-driven-development`. `subagent-driven-development` was NOT vendored — the agentic-loop already embodies that pattern.

**Spec E** — Adds the two missing durable artifacts (`spec.md`, `plan.md`) so the loop has the full spec→plan→progress chain. Artifacts live in the loop-state dir (next to `progress.json`, outside the repo, uncommitted). Complexity guard: only fires at ≥3 work-units or a cross-unit dependency (Phase 3's own TeamCreate threshold, pulled one phase earlier).

**PR #18** — Dropped `tsh ssh` (Teleport) from Phase 4/12 illustrative examples. Tool-agnostic cleanup, no design change.

## Through-line

Two compounding guarantees now underpin the loop:
1. **Full artifact chain:** spec→plan→progress. C1 made progress.json reliable; E added the first two as durable docs.
2. **Two-hook loop-state guard:** C1 (presence/ownership) + C2 (declaration-based anti-stall), sharing `lib/loop_state_common.sh`. Stop order: confidence → verify → loop_state_guard → loop_stall_guard.

## Wiki pages updated / created

- Updated: [[agentic-loop]]
- Created: [[loop_state_guard]], [[loop_stall_guard]]
- Created: [[test-driven-development]] (skill), [[writing-plans]] (skill)
- Created: [[spec-plan-progress-artifact-chain]] (design)
