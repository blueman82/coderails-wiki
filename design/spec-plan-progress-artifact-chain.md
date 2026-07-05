---
title: "Spec→Plan→Progress artifact chain + two-hook loop-state guard"
type: design
created: 2026-06-25
last_updated: 2026-07-05
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_49_gate-function-rename.md
  - sources/session_2026-07-03_ai-docs-refresh-and-cc-mechanics-probes.md
  - sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md
tags: [design, agentic-loop, artifact-chain, loop-state, hooks, progress-json, hook-owned-counter]
---

# Spec→Plan→Progress artifact chain + two-hook loop-state guard

The agentic-loop upgrade arc (PRs #12–#18) gave the loop two compounding architectural guarantees: a full **spec → plan → progress** durable artifact chain, and a **two-hook loop-state guard** that mechanically enforces the chain's cursor. This page captures the relationships; the per-component detail lives on the individual pages.

## Context — the loop was calibrated one direction only

Before this arc, the agentic-loop skill guarded almost exclusively against **over-asking** (stalls, re-asks, holding at removed gates). It was blind to **over-production** (building more than asked) and had no durable record of *what* it was building. It reimplemented only the third of superpowers' three build artifacts — it tracked *position* (`progress.json`) but had no durable *design* (spec) or *decomposition* (plan).

## The artifact chain

| Artifact | Captures | Written by | Lives where |
|---|---|---|---|
| `spec.md` | What we decided and why (envelope, design fork, disposition) | Phase 2.7 (in-line loop write) | loop-state dir, uncommitted |
| `plan.md` | The durable decomposition (tasks with files, interfaces, verify-criteria) | Phase 2.8 via `coderails:writing-plans` | loop-state dir, uncommitted |
| `progress.json` | Where we are against the plan (the cursor) | Phase -2 stub → enriched per phase | loop-state dir, session-owned |

**`plan.md` is the static SSOT; `progress.json` is the dynamic cursor against it** (verified: Spec E design + Phase 2.8). A position artifact is only as good as the static SSOT it indexes — before E, "work-unit 2 is in-progress" pointed at ephemeral Phase 1 bullets that compacted away.

### Why spec is in-line but plan is a skill

A loop cannot brainstorm with itself. By Phase 2.7 the design is already resolved (envelope at Phase 0, design fork at Phase 2.5, disposition at Phase 2.6). The spec phase **commits already-resolved design to disk** — it does not re-open it, so it is an in-line write. The plan is genuine decomposition work, delegated to `coderails:writing-plans` the way Phase 3/3a delegate construction to `coderails:test-driven-development`.

### Where the artifacts live (anti-pollution invariant)

All three sit in `~/.claude/agentic-loop/<cwd-slug>/`, resolved by `agentic_loop_path.sh` — **outside the code repo, cwd-keyed, uncommitted**. This honours Phase 2.5's anti-pollution rule: loop state never lands on local `main` or in a worker's base. They are loop *state* keyed to one orchestrator's run, not shareable design records.

**Tradeoff (acknowledged, deliberate):** a teammate or different machine picking up the branch sees no `spec.md`/`plan.md` — they are machine-local loop state, exactly like `progress.json`. The shareable record is the committed spec under `docs/` (for a real product) or `coderails:handoff` for ad-hoc work. (The arc's own spec docs under `coderails/docs/superpowers/specs/` are committed because coderails *is* the product — a different thing from a loop running in some other project.)

### Complexity guard

Phases 2.7/2.8 fire ONLY at **≥3 work-units or a cross-unit dependency** — the *exact* threshold Phase 3 already uses to choose `TeamCreate` over a single Agent. Reusing Phase 3's own line (rather than inventing "≥2") means the guard is not a new judgement call, and avoids forcing formal docs onto single-agent work. Ceremony that fires on trivial work trains the loop to skip it on real work.

## The two-hook loop-state guard

`progress.json` was an unenforced convention. Two hooks make it reliable, sharing `hooks/scripts/lib/loop_state_common.sh` so they can never disagree on what "an active loop" means.

| Hook | Spec | Enforces | Boundary |
|---|---|---|---|
| [[loop_state_guard]] | C1 | presence + ownership | cannot force content accuracy |
| [[loop_stall_guard]] | C2 | `LOOP-STOP` declaration presence + valid category | cannot force the reason to be truthful |

**Stop hook order**: `check_confidence_labels` → `check_verify_loop` → `loop_state_guard` (C1) → `loop_stall_guard` (C2). C1 (presence is the more fundamental fix) speaks before C2.

### Shared lib — single source of truth

`loop_state_common.sh` (Spec C2) is sourced by both guards. It holds:
- `LOOP_STOP_VOCAB="hard-stop|approval-gate|awaiting-input|complete"` — defined once; C2 builds both its match regex and its block message from it, so the message can never advertise a category the regex rejects.
- The loop-active detection (structured `jq` over the transcript for a `coderails:agentic-loop` Skill `tool_use`, with the transcript-flush retry) and the `progress.json` state read.
- Named `als_gate_*` functions for Gates 1–4 shared between C1 and C2 (extracted PR #49; formerly byte-identical in both scripts): `als_gate_no_transcript`, `als_gate_stop_hook_active`, `als_gate_not_a_loop`, and `als_load_progress`. Each guard adds its own Gates 5–6 for guard-specific enforcement.

C1 (#13) was created with its detection inline; C2 (#14) extracted that into the shared lib and refactored C1 to source it — a behaviour-preserving extraction gated by C1's existing 8/8 test suite passing unchanged.

### The shared off-switch

Both hooks treat a loop as over only when `progress.json status == "complete"` AND not re-armed (re-armed = `invocation_count > completed_marker`). This is why **declaring `LOOP-STOP: complete` MUST atomically set `status: complete`** — a text-only `complete` satisfies one turn's gate but leaves both hooks treating the loop as active forever, a hang that looks like a stuck hook.

## Two recurring design lessons across the arc

1. **The model must never derive a value the hook will re-derive.** C1's path-deadlock (a model can't reproduce a cwd-slug) → the hook is the sole path authority and puts the resolved path in its block message. C2 applies the same lesson to the `LOOP-STOP` tag format (copy-paste template in the block message). The model copies; it never computes.
2. **A declaration is auditable; prose is not.** C2 rejected text-heuristic stall detection (can't separate a real question from a stall) in favour of a vocab-checked declaration. The honest boundary is uniform across C1/C2/D/E: the hooks/seams force a *declaration* or *reference*, they cannot force *honesty*. Measurement (Phase 13 KPIs) is the counter-pressure, not a tighter check.

## Where it is enforced

- **Mechanical (hooks):** `loop_state_guard.sh` (presence/ownership), `loop_stall_guard.sh` (declaration). Registered in `hooks/hooks.json` Stop array; armed by `install.sh`'s explicit chmod list (both new scripts AND both new lib scripts must be in it — it is a hardcoded list, not a glob).
- **Advisory (skill prose):** the artifact chain itself (Phases 2.7/2.8) and the construction seam (Phase 3/3a → `coderails:test-driven-development`) are skill references — advisory by design. A mechanical "no dispatch without a plan" gate is PreToolUse territory, explicitly deferred.

### Rejected: migrating these two guards into skill-frontmatter hooks

Considered (2026-07-03) and rejected: moving `loop_state_guard`/`loop_stall_guard` out of `hooks/hooks.json` and into `skills/agentic-loop/SKILL.md` frontmatter hooks instead. Feasible mechanically, but there is no documented or observed guarantee that skill-frontmatter (`PostToolUse`) hooks survive a post-compaction session restart — empirical probing that same session found such hooks are session-scoped (persist across turn boundaries within a session, but a separate session's calls trigger nothing). These two guards are safety-critical and need deterministic scope, so they stay as `hooks.json`-wired Stop hooks. See [[session_2026-07-03_ai-docs-refresh-and-cc-mechanics-probes]].

## See also

- [[agentic-loop]] — the skill that produces and consumes the chain
- [[loop_state_guard]] · [[loop_stall_guard]] — the two hooks
- [[writing-plans]] · [[test-driven-development]] — the two vendored skills (E→D tie)
- [[enforcement-model]] — hooks (mechanical) vs. commands/skills (advisory)
- [[discipline-loop]] — the broader hook composition
