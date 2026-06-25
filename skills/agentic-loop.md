---
title: "Skill: agentic-loop"
type: skill
created: 2026-05-31
last_updated: 2026-06-25
sources:
  - skills/agentic-loop/SKILL.md
  - sources/session_2026-05-31_prompting-doc-alignment.md
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
tags: [skill, agentic-loop, multi-agent, orchestration, context-window, delegation, artifact-chain, loop-state]
---

# Skill: agentic-loop

The multi-agent orchestration discipline skill. Sits *above* `/workflow` — it uses `/workflow` as a subroutine for each PR in a loop. Covers autonomous multi-PR sessions where the user has waived per-step confirmation.

Source: `skills/agentic-loop/SKILL.md`

> Substantially upgraded by the 2026-06-24 spec arc (PRs #12–#18). See [[session_2026-06-25_agentic-loop-upgrade-arc]] and the [[spec-plan-progress-artifact-chain]] design page. This page reflects the post-arc skill.

## When to load

Load immediately — before `/workflow`, `/prep`, `/push` — when the user authorises a sequence of agent-driven work. Trigger phrases (verified: SKILL.md description block):

- "TeamCreate", "spawn a team", "no human gates", "self-merge", "crack on", "without the human", "no per-PR confirmation", "agentic loop", "multi-PR"
- Any authorisation of 3+ PRs in one instruction
- Autonomous merge + deploy + verify chains (even single-PR) when the user has waived per-step confirmation

Do NOT load for standard single-PR `/workflow` runs where the user is present at each gate.

## Phase structure (post-arc)

The skill defines a sequence of decimal-numbered phases (decimals are the established insertion pattern: -2, -1, 0, 0.5, 2.5, 2.6, 2.7, 2.8, 3a). Phases 4–6 repeat per work-unit inside the loop. Full ordering (verified: SKILL.md headings):
`-2, -1, 0, 0.5, 1, 2, 2.5, 2.6, 2.7, 2.8, 3, 3a, 4, 4b, 5, 6, 7&8, 9, 10, 11, 12, 13`.

| Phase | Name | Core action | Added by |
|---|---|---|---|
| -2 | Stub `progress.json` first | Literal first action; write the stub at the helper-resolved path | C1 |
| -1 | Sharpen the authorising prompt | Run `/coderails:improve-prompt`, ask once | (pre-arc) |
| 0 | Read the authorisation envelope | `<thinking>`: verbatim quote, envelope class, in/out-of-scope | (pre-arc) |
| 0.5 | Orchestrator operating rules | Stop-ceremony: labels + DNV + `LOOP-STOP` together | C2 |
| 1 | State the plan in bullets | Ask once to confirm | (pre-arc) |
| 2 | Pre-flight via spawned agents | Delegate planning/premortem/wiki-query to sonnet agents | (pre-arc) |
| 2.5 | Resolve design forks up front | Ask once; record decision + flip-condition | (pre-arc) |
| 2.6 | Resolve disposition before replacement | clean-break vs preserve-compat; named blocker required | A |
| 2.7 | Commit resolved design to `spec.md` | In-line write; ≥3-unit guard | E |
| 2.8 | Write `plan.md` via `coderails:writing-plans` | ≥3-unit guard; the static SSOT Phase 3 consumes | E |
| 3 | Delegate all impl to sonnet agents | TeamCreate at ≥3 sequential units / dependency chain | (pre-arc) + A + D |
| 3a | Single sonnet agent for impl + verify | The TeamCreate-is-overkill case | (pre-arc) + A + D |
| 4 | Spawn workers in waves | Check artifacts, never idle pings | (pre-arc) |
| 4b | PR review = the six toolkit agents + `/security-review` | Clean-break compat hunt is a MERGE-BLOCKER | A |
| 5 | Disprove the premise before each fix | Reproduce via SOT before spawning | (pre-arc) |
| 6 | Match confirmation to envelope | Don't ask inside authorised scope | (pre-arc) |
| 7&8 | Stack-specific deploy/push tactics | Collapsed to a generic stub | B |
| 9 | Cluster wiki ingest | Run ingest+lint once at loop end | (pre-arc) |
| 10 | v2/v3 names when respawning | Versioned names identify the live agent | (pre-arc) |
| 11 | Agent prompts include confidence labels | Propagate the labelling standard | (pre-arc) |
| 12 | Status reports are claims, not evidence | Re-check artifact at moment of action | (pre-arc) |
| 13 | Confirm the factory ran (terminal self-audit) | Disposition + `loop_stop_counts` KPIs | A + C2 |

## The spec → plan → progress artifact chain (Spec E)

The loop now writes three durable artifacts, all in the loop-state dir `~/.claude/agentic-loop/<cwd-slug>/` (resolved by `agentic_loop_path.sh`, outside the repo, uncommitted):

- `spec.md` (Phase 2.7) — the *already-resolved* design: envelope, design fork + flip-condition, disposition + named blockers, success criteria. A loop cannot brainstorm with itself, so this commits resolved design, it does not re-open it.
- `plan.md` (Phase 2.8, via `coderails:writing-plans`) — the durable decomposition. **The static SSOT** Phase 3 builds its task list from, and re-reads for *scope* after a compaction.
- `progress.json` (Phase -2 onward) — **the dynamic cursor** against `plan.md`.

Both `spec.md`/`plan.md` fire ONLY at the complexity guard: ≥3 work-units or a cross-unit dependency (Phase 3's own TeamCreate threshold, pulled one phase earlier). Sub-threshold loops skip them — ceremony on trivial work trains the loop to skip it on real work. See [[spec-plan-progress-artifact-chain]] and [[writing-plans]].

## The LOOP-STOP stop-ceremony contract (Spec C2)

When the loop is active and incomplete, the orchestrator cannot stop without a declaration in its stopping turn:

```
LOOP-STOP: <hard-stop|approval-gate|awaiting-input|complete> — <reason>
```

Phase 0.5 bundles this with the confidence-label and DNV requirements into one stop-ceremony, so the orchestrator emits all three together rather than thrashing one Stop hook while satisfying another. The four categories map onto the Stop-conditions section. **Declaring `complete` is atomically the Phase 13 teardown** that sets `progress.json status: complete` — a text-only `complete` leaves both loop-state hooks treating the loop as active. Enforced mechanically by [[loop_stall_guard]].

## The clean-migration disposition fork (Spec A)

Phase 2.6 forces a disposition decision before replacement work: **clean-break** (remove the old path; no shims) vs **preserve-compat** (keep it behind a shim, with a named blocker and a mandatory removal ticket). clean-break is the stated default; preserve-compat requires a *specific named blocker* (anti-laundering — a generic "safer" is rejected). The decision is propagated verbatim into the worker prompt (Phase 3), and the **independent code-simplifier reviewer is the load-bearing gate** (Phase 4b) — it hunts relabelled compat (fallback/adapter/guard/bridge) and whether an old path still executes; findings are MERGE-BLOCKERS. The worker's own assertion (Phase 3a) is a smell test, not the gate. Phase 13 counts `disposition-violations`, distinguishing "0 violations" from "no record found" (the latter = audit failure).

## progress.json schema growth across the arc

| Field | Added by | Purpose |
|---|---|---|
| `disposition`, `named_blocker`, `removal_ticket` | A | per-work-unit clean-break record |
| `schema_version`, `session_id`, `status`, `created`, `last_updated`, `completed_marker` | C1 | lifecycle + presence/ownership |
| `loop_stop_counts` (`{hard-stop, approval-gate, awaiting-input, complete}`) | C2 | per-category stall metric |

`status` ∈ `initialising` | `in-progress` | `complete`. The path is resolved by `agentic_loop_path.sh` — **the model never computes it** (a cwd-slug cannot be reproduced by hand).

## Construction discipline (Spec D)

Phase 3 and Phase 3a reference `coderails:test-driven-development` (code-guarded: "if the change adds or alters a function, method, or branch that can carry a test"). Vendored as a coderails-owned skill so the plugin keeps zero cross-plugin dependency (REVERSED from Spec A's "reference, not vendor" note). The reference sits near the TOP of the Phase 3a prompt-contract list (Phase 9's placement lesson: scope-shaping instructions get shortcut when buried low). `subagent-driven-development` was NOT vendored — the agentic-loop already embodies that orchestration pattern. See [[test-driven-development]].

## When to delegate vs. work directly

(verified: SKILL.md, added 2026-05-31). Use subagents when tasks run in parallel, need isolated context, or are genuinely independent. Work directly on single-file edits, sequential steps, and work needing shared context. TeamCreate is for ≥3 sequential units or cross-step dependency chains — not "any multi-step task." This is the same threshold the Phase 2.7/2.8 complexity guard reuses.

## Context-window persistence

(verified: SKILL.md). Do not stop work early because the context window is filling — context compacts and the session continues. Before compaction, checkpoint: commit in-progress work to git, update `progress.json`, record the loop's phase position. Git is the authoritative checkpoint. (Post-arc: the orchestrator also re-reads `plan.md` for *scope* after compaction — stated in Phase 2.8, deliberately NOT in this section, which is a Stop-hook no-touch region describing `progress.json` alone.)

## Stop conditions

The loop runs autonomously until ANY of (verified: SKILL.md): verification failure that can't auto-recover; premise disproven (Phase 5); genuinely ambiguous decision outside the envelope; destructive/irreversible op not authorised; all authorised work complete. On stop: report state with confidence labels, emit the `LOOP-STOP` declaration, propose the next move, then wait. Model an approval-gate as "pause-then-proceed," never "do not start."

## Slimming (Spec B)

The skill was cut 454→434 lines: Phases 7&8 (corporate docker/Teleport residue) collapsed to one generic stub; 16 war-stories compressed to one-clause `Past failure:` tags; verbose "why" paragraphs trimmed. **Six no-touch regions** (the frontmatter description, Phase -2 stub block, Phase 0.5 LOOP-STOP bullet, Phase 13 KPI bullet, Stop-conditions "Declaring the stop" block, and the `## Context-window persistence` section) are byte-stable — they teach the exact behaviour the Stop hooks check, so editing them would turn a hook from a safety net into a stall generator. The byte-diff against `origin/main` is the primary verification gate; token greps are necessary-not-sufficient. (PR #18 separately dropped `tsh ssh` from the Phase 4/12 illustrative examples — tool-agnostic cleanup, no contract change.)

## Key architectural decisions encoded

- **Pre-flight + worker agents use `model: sonnet`** — orchestration pattern; cost control. No escalation path; D's TDD and E's planning skills carry no model guidance that could escalate.
- **Wiki ingest clusters, not per-PR** — Phase 9; fragmented ingests produce fragmented wiki context.
- **Scope-shaping instructions go high in worker prompts** — Phase 9 lesson, reused by D's TDD placement.
- **Artifact verification not idle pings** — Phase 4/12.
- **The model never computes a hook-derived value** — path (C1) and LOOP-STOP tag format (C2) both come from the hook's block message.

## Cross-references

- [[spec-plan-progress-artifact-chain]] — the design page for the artifact chain + two-hook guard
- [[loop_state_guard]] — C1 hook: presence/ownership of `progress.json`
- [[loop_stall_guard]] — C2 hook: the `LOOP-STOP` declaration enforcement
- [[test-driven-development]] — vendored construction skill (Phase 3/3a)
- [[writing-plans]] — vendored plan skill (Phase 2.8)
- [[enforcement-model]] — hooks (mechanical) vs. skill prose (advisory)
- [[discipline-loop]] — how the discipline hooks compose
- [[session_2026-06-25_agentic-loop-upgrade-arc]] — the PR #12–#18 source record
- [[session_2026-05-31_prompting-doc-alignment]] — earlier source for delegate/context guidance
