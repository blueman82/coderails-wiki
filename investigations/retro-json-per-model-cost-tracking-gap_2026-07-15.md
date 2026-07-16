---
title: "retro.json / Phase 13 / dashboard: what a per-model cost-tracking plan would assume but isn't enforced"
type: investigation
created: 2026-07-15
last_updated: 2026-07-15
sources:
  - skills/agentic-loop.md
  - hooks/loop_stall_guard.md
  - design/spec-plan-progress-artifact-chain.md
  - design/loop-progress-fields.md
  - skills/dashboard.md
  - sources/pr_118-123_self-improving-loops.md
  - sources/pr_169_model-routing-step.md
tags: [investigation, retro-json, phase-13, dashboard, cost-tracking, token-usage, model-routing, gap-analysis, resolved]
---

# retro.json / Phase 13 / dashboard: what a per-model cost-tracking plan would assume but isn't enforced

> **RESOLVED same day, 2026-07-15.** The plan this investigation was filed
> ahead of shipped as PRs #184/#185/#186 within hours — see
> [[pr_184_185_186_loop-cost-tracking]] for the source record. The headline
> finding below ("no cost or per-model tracking anywhere") is no longer
> true. This page is kept as the point-in-time gap analysis and design-input
> record per the wiki's investigation-page convention (point-in-time,
> superseded rather than deleted) — the cross-PR constraints section below
> held up as the actual design contract the shipped cluster followed. The
> one constraint that did **not** get closed: #4, model-identity-at-spawn
> attribution — the miner reads `message.model` per-transcript-line (a
> stronger per-message join than this page anticipated), but still has no
> reconciliation against Phase 2.8's role-assignment intent.

Query-driven wiki investigation (2026-07-15): what does the wiki know about
`retro.json`, agentic-loop Phase 13 teardown, the dashboard's data sources, and
existing cost/token/model tracking — filed because a plan to add **per-model
cost tracking to retro.json + dashboard rollups** is being considered, and the
gap between what such a plan would assume and what the code actually enforces
is large enough to be worth a durable record.

## Headline finding

**There is no cost or per-model tracking anywhere in coderails today.** Not in
`retro.json`'s schema, not in `progress.json`, not in the dashboard. The wiki
had zero pages under any cost/pricing/budget/spend keyword before this one.
There IS a token-usage collector (`skills/dashboard/app/src/lib/collect/usage.ts`)
feeding the SYSTEM VITALS panel — but it was **completely undocumented in the
wiki** until this investigation (found only by grepping the actual TypeScript
source, not by any wiki page). That collector's shape is the single most
important existing artifact a cost-tracking plan needs to know about, and it
does not do what "per-model cost tracking" would need.

## What retro.json actually is (verbatim schema, from `skills/agentic-loop/SKILL.md:527`)

Fields: `session_id`, `created`, `loop_ordinal`, `envelope` (verbatim from
`progress.json.authorising_prompt_raw`), `loop_stop_counts` (verbatim,
hook-owned), `decisions_absorbed` (verbatim array), `disposition_record`,
`evals` (`result`/`amendments`/unresolved P1s), `artifacts` (PRs + verifying
check), `hook_blocks` (mined from `discipline.log` by `dc_mine_hook_blocks`),
`review_themes`, `raw_notes`. **No `verdict` field** — this is structural, not
an oversight: "the retro records what happened, it does not grade it." See
[[agentic-loop]] and [[pr_118-123_self-improving-loops]].

No field carries tokens, cost, or a model identifier. `decisions_absorbed`
carries model **routing** (see below) but as a single free-text bullet, not
structured usage data.

## Phase 13 teardown order (write contract, gated)

1. Assemble `retro.json` (`schema_version` 1) beside `progress.json`.
2. Update the repo-keyed `standing-orders.md` overlay.
3. Write feedback auto-memories.
4. Only then declare `complete`.

The `complete` declaration is gated: [[loop_stall_guard]]'s
`als_gate_retro_on_complete` (PR #119) blocks a `LOOP-STOP: complete`
declaration unless a parseable `retro.json` (`schema_version` 1) sits beside
`progress.json`. This is presence-and-parse only — provenance and content
fidelity are not checkable, the same honest boundary every other coderails
gate uses. Fail-open on absent `jq`; fail-closed when `ALS_PATH` is unset.
Category is lowercase-normalised (a case-sensitivity bypass was the Critical
caught in review — `Complete`/`COMPLETE` had skipped the gate entirely before
the fix). Block-before-bump: a blocked `complete` never increments
`loop_stop_counts`.

**Any new retro.json field a cost-tracking plan adds inherits this gate for
free (schema_version 1, presence+parse) but gets zero enforcement on
correctness** — a hallucinated or stale cost figure would satisfy the gate
exactly as well as a real one.

## Model routing exists, but is prose, not data (PR #169, 2026-07-14)

Phase 2.8 assigns a **capability role** (`fast-mechanical`/`default`/
`frontier` → currently haiku/sonnet/fable) to every build task before Phase 3.
*(Point-in-time note: accurate when filed; PR #192 (2026-07-16) re-pointed `frontier` to
opus-at-`xhigh` with fable as a justified escalation — see [[pr_192_frontier-opus-effort-routing]].)*
Critically:

- The assignment is recorded **once**, in `progress.json.decisions_absorbed`,
  as a single `{phase: "2.8", decision: "<task id: role, ...>"}` string
  covering every task — not a structured per-task, per-model machine-readable
  record.
- **No hook enforces which model a spawned worker actually runs at.** Verified
  directly: `hooks/hooks.json`'s only `PreToolUse` matchers are `Bash` and
  `Write|Edit|MultiEdit` — none gate `Agent`/`Task` spawn calls. `AGENTS.md`
  states this is deliberate: "routing exists for cost and latency, not
  correctness" — PR gates (review, evals, hook-seam) are model-independent by
  design, so nothing breaks if a role assignment is silently ignored by a
  worker or a spawn call.
- This means: **a plan assuming "the model that ran task X is knowable from
  progress.json/retro.json" is assuming something the system does not
  guarantee.** The stamp records *intent* (Phase 2.8's assignment), not
  *fact* (which model actually executed). There is no reconciliation step
  anywhere that checks a worker's declared role against what it was actually
  spawned with.
- See [[agentic-loop]] "Model-role routing is advisory, not hook-enforced" and
  [[pr_169_model-routing-step]].

## The one existing usage collector — undocumented in the wiki, found only in source

`skills/dashboard/app/src/lib/collect/usage.ts` (`collectUsage`), consumed by
`collect/health.ts` to build the SYSTEM VITALS panel's two token tiles
(`usage5h`, `usageWeek`):

- Reads **every** `*.jsonl` transcript under `~/.claude/projects/` (recursively,
  machine-wide — not scoped to a repo, loop, session, or PR), filtered only by
  file mtime within a 7-day window as a cheap prefilter.
- Parses each line for `type === "assistant"` with a numeric `usage.input_tokens`
  / `usage.output_tokens`; sums `input_tokens + cache_creation_input_tokens +
  cache_read_input_tokens` into `inputTokens`, and separately tracks
  `cacheReadTokens` (called out because on real transcripts cache re-reads are
  ~99% of the input total — the tile note surfaces this so the headline number
  isn't misread as fresh consumption).
- De-dupes by `message.id` (streaming re-emits the same id with an identical
  cumulative snapshot; only the first occurrence is kept).
- Produces exactly two rolling windows: last 5 hours, last 7 days. **No
  per-model breakout** — the transcript JSONL line does carry a model field in
  real Claude Code output, but `usage.ts` never reads it. **No per-session,
  per-loop, per-PR, or per-repo grouping** — it is a single flat sum across
  every local project transcript on the machine. **No dollar conversion** —
  output is a raw token count formatted as `"1.2M tok"`, nothing else.
- Degrades to `null`/"unavailable: no local usage source" if the base dir is
  unreadable; never throws.

This is the **only** place tokens are counted anywhere in the codebase. A
per-model cost-tracking plan would almost certainly want to build on or
replace this collector, but it currently answers a different question
("how many tokens has this machine burned recently, machine-wide") than "how
much did loop/PR X cost, broken out by model."

`RunRecord` (`skills/dashboard/app/src/lib/runlog.ts`, the dashboard's
per-button-run JSONL ledger) has no token, cost, or model field either — it
tracks `runId`/`button`/`argv`/`cwd`/`profile`/`startedAt`/`endedAt`/
`exitCode`/`signal`/`outputPath`. Runs and usage are two entirely separate
data paths that never join today.

## Dashboard data sources and aggregation, generally

Six panels (SYSTEM VITALS, DIRECTIVES, COMMAND DECK, PR GATES, bottom-centre
hero stat, ASSISTANT.LINK) all read state the kernel already produces — no new
services, no telemetry leaving the machine. Collectors read: `~/.claude/projects/*`
session dirs (usage + session collectors), agentic-loop `progress.json` (via
`agentic_loop_path.sh`), `gh`-polled PR state (marker-grammar parsed, shared
with `review-artifact.sh`/`eval-artifact.sh`), hook logs (`discipline.log`),
and wiki/memory mtimes. `DIRECTIVES` now renders one card per **live** loop
(PR #168, `LOOP_LIVE_WINDOW_MS = 60min`), reading `progress.json`'s
`decisions_absorbed` for its per-card decisions sub-list (last 5, newest-first)
— this is the same field a cost-tracking plan might be tempted to overload for
model-routing facts; it is orchestrator-owned prose, not built for numeric
rollups. See [[dashboard]].

Any "dashboard rollup" of retro.json-sourced cost data would be a **seventh**
collector-then-render seam, following the same pattern as `collectLoops`
(`decisions_absorbed`) or `readEvalsFrozen` (evals `grading` stamp) — both of
which are read-only, degrade-don't-throw, and independently must stay in
lockstep with their bash-side SSOT (see [[task-evals-gate]]'s "one schema,
three seams" framing). A new retro.json cost field would become a **fourth**
seam needing the same lockstep discipline, with no existing mechanism to
enforce it beyond code review.

## Cross-PR constraints a per-model cost-tracking plan must respect

1. **`loop_stop_counts` is HOOK-OWNED, sole writer, and copied verbatim into
   retro.json** (PR #98, reinforced by PR #147's conditional carry-forward
   rule). Any new cost field must pick an ownership model explicitly —
   orchestrator-owned like `decisions_absorbed`, or hook-owned like
   `loop_stop_counts` — and the wiki's `[[loop-progress-fields]]` page is the
   established pattern for documenting exactly that split. Silently defaulting
   to "orchestrator writes it" repeats the two-writer race PR #98 fixed for
   `loop_stop_counts` if a hook (e.g. a future usage-mining hook) also tries
   to write it.
2. **retro.json has no verdict field by design.** A raw per-model cost number
   is fine (matches "raw, unscored"); a self-issued efficiency verdict
   ("model X was wasteful") would violate the explicit "record what happened,
   don't grade it" rule that also killed rollback-on-regression in the v2
   promotion pipeline design (PR #121).
3. **The retro gate is presence+parse only.** Adding a cost field does not
   get correctness checking for free — a worker or orchestrator could write a
   fabricated number and the gate would not notice, same honest boundary as
   every other field.
4. **No hook enforces actual model identity at spawn time.** Any cost
   attribution keyed off Phase 2.8's role assignment is attributing cost to
   *declared intent*, not *verified fact* — the same gap `pr_169` documents
   for correctness. If the actual token-usage collector (`usage.ts`) doesn't
   record which model produced which tokens, and the routing decision doesn't
   record which model a worker actually ran, there is currently **no join
   key** connecting "these N tokens" to "this Phase-2.8 role/model" at all —
   this would need new instrumentation on both sides, not just a schema
   addition to retro.json.
5. **`usage.ts` is machine-wide and session-scoped, not loop-scoped.** It has
   no concept of "this loop's tokens" vs. "some other unrelated session
   running concurrently on the same machine." A loop-scoped cost rollup would
   need to correlate transcript timestamps against a loop's start/end window
   (itself not currently a tracked field — `progress.json` doesn't record a
   loop start time as of this writing) and filter to the orchestrator's own
   session id plus any spawned worker session ids, none of which `usage.ts`
   currently threads through.
6. **`docs/coderails/specs/` and `docs/coderails/plans/` are gitignored, not
   committed** (PR #138, 2026-07-11) — a plan for this feature will not
   itself land as a permanent repo artifact; only this wiki page and the
   eventual PR are durable records.

## Superseded / stale things to watch for

- Pre-2026-07-14 dashboard descriptions citing "seven panels" or a
  "wiki + memory mtime feed" are stale by construction — PR #166 removed the
  Memory.Trail panel clean-break; six panels is current.
- Pre-PR #169 "model: sonnet" blanket assertions in `agentic-loop/SKILL.md`
  are superseded by the capability-role vocabulary
  (`fast-mechanical`/`default`/`frontier`); a cost-tracking plan should key
  off roles, not literal model-name strings, to avoid the same staleness PR
  #169 fixed ("a named-tier table went stale within a day of Fable 5's
  release").
- The v2 loop-retro-promotion pipeline (PR #121) is dormant by design
  (graduation predicate: ≥10 retros + one lifecycle + one clean decay per repo
  key) — do not assume any mined/aggregated cross-loop lesson store already
  exists to piggyback cost-rollup mining onto; it ships inert.

## See also

- [[agentic-loop]] — Phase 13 teardown contract, Phase 2.8 model-role routing
- [[loop_stall_guard]] — hosts the retro gate (`als_gate_retro_on_complete`, PR #119)
- [[spec-plan-progress-artifact-chain]] — the artifact family retro.json joined
- [[loop-progress-fields]] — the established pattern for documenting a
  progress.json/retro.json field's ownership model (orchestrator vs. hook)
- [[dashboard]] — panel/collector architecture; SYSTEM VITALS is where
  `usage.ts` surfaces today
- [[task-evals-gate]] — "one schema, three seams" framing a new cost seam
  would extend to four
- [[pr_118-123_self-improving-loops]] — retro.json's origin, the no-verdict
  design decision, the dormant v2 pipeline
- [[pr_169_model-routing-step]] — capability-role vocabulary, the advisory
  (non-hook-enforced) nature of model routing
- [[pr_184_185_186_loop-cost-tracking]] — the resolution: the three-PR
  cluster that shipped the miner, the `retro.json` schema_version 2 write
  contract, and the dashboard rollup this page found missing
