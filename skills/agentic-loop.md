---
title: "Skill: agentic-loop"
type: skill
created: 2026-05-31
last_updated: 2026-07-12
sources:
  - skills/agentic-loop/SKILL.md
  - sources/session_2026-05-31_prompting-doc-alignment.md
  - sources/session_2026-06-01_agentic-loop-delegate-all-impl.md
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_19-30_self-containment-and-hardening.md
  - sources/pr_39_agentic-loop-slim-v2.md
  - sources/pr_41_phase25-brainstorming-xref.md
  - sources/pr_64_loop-review-via-skill.md
  - sources/pr_77_agentic-loop-sync-docs-step.md
  - sources/pr_81-83_review-artifact-seam.md
  - sources/pr_86_agentic-loop-hardening.md
  - sources/pr_87_agentic-loop-path-session-keying.md
  - sources/pr_89-91_skills-doc-frontmatter-injection.md
  - sources/pr_1-4_task-evals-feature.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_134_agentic-loop-retry-until-green.md
  - sources/pr_144-149_agentic-loop-hardening-from-loop-engineering.md
tags: [skill, agentic-loop, multi-agent, orchestration, context-window, delegation, artifact-chain, loop-state, post-review, review-artifact, self-attestation, enforcement-ceiling, session-keying, frontmatter, description-cap, task-evals, work-units, teamcreate-purge, sdd-ledger, retry-until-green, hard-stops, decisions-absorbed, grade-loop, review-tier-ladder]
---

# Skill: agentic-loop

The multi-agent orchestration discipline skill. Sits *above* `/workflow` — it uses `/workflow` as a subroutine for each PR in a loop. Covers autonomous multi-PR sessions where the user has waived per-step confirmation.

Source: `skills/agentic-loop/SKILL.md`

> Substantially upgraded by the 2026-06-24 spec arc (PRs #12–#18), then hardened again by PR #86 (2026-07-01, 7 review-driven decisions), then had its single remaining documented-but-unfixed limitation closed by PR #87 (2026-07-01, same day), then gained a loop-scope eval gate (Phase 2.7c) and a TeamCreate→spawned-team terminology purge via [[pr_1-4_task-evals-feature]] (2026-07-06). See [[session_2026-06-25_agentic-loop-upgrade-arc]], the [[spec-plan-progress-artifact-chain]] design page, [[pr_86_agentic-loop-hardening]], [[pr_87_agentic-loop-path-session-keying]], and [[task-evals-gate]]. This page reflects the post-arc, post-hardening, post-session-keying-fix, post-task-evals skill.

## When to load

Load immediately — before `/workflow`, `/prep`, `/push` — when the user authorises a sequence of agent-driven work. Trigger phrases (verified: SKILL.md description block):

- "TeamCreate", "spawn a team", "no human gates", "self-merge", "crack on", "without the human", "no per-PR confirmation", "agentic loop", "multi-PR"
- Any authorisation of 3+ PRs in one instruction
- Autonomous merge + deploy + verify chains (even single-PR) when the user has waived per-step confirmation

Do NOT load for standard single-PR `/workflow` runs where the user is present at each gate.

**"TeamCreate" survives here only as a quoted user-phrasing trigger** (PR #4 of [[pr_1-4_task-evals-feature]]). The actual `TeamCreate`/`TeamDelete` tools no longer exist (removed in Claude Code v2.1.178) — the mechanism the skill now instructs elsewhere is named-teammate `Agent` spawns coordinated through a shared task list (`TaskCreate`/`TaskUpdate` with `blockedBy` dependencies) and `SendMessage`. The trigger line is retained verbatim here specifically because it describes what a *user might still type*, not what the orchestrator invokes — every other reference to the old tool name across the skill body was rewritten to describe the real mechanism.

## Phase structure (post-arc, post-hardening)

The skill defines a sequence of decimal-numbered phases (decimals are the established insertion pattern: -2, -1, 0, 0.5, 2.5, 2.6, 2.7, 3a). Phases 4–6 repeat per work-unit inside the loop. Full ordering (verified: SKILL.md headings, post PR #86):
`-2, -1, 0, 0.5, 1, 2, 2.5, 2.6, 2.7 (2.7a/2.7b), 3, 3a, 4, 4b, 5, 6, 7&8, 9, 10, 11, 12, 13`.

**Stage-map (added PR #86, §3.1 of [[pr_86_agentic-loop-hardening|the hardening spec]]).** A cold reader hit 19+ ungrouped phases with no shape to hold in mind. `SKILL.md` now opens `## The phases` with a 5-row grouping table before the phase-by-phase detail (verified: SKILL.md):

| Stage | Phases |
|---|---|
| Setup | -2, -1, 0, 0.5 |
| Pre-flight | 1, 2, 2.5, 2.6, 2.7 |
| Build | 3, 3a, 4 |
| Review & Ship | 4b, 5, 6, 7&8 |
| Wrap-up | 9, 10, 11, 12, 13 |

Full renumbering was rejected — a 9-reference audit plus `docs/coderails-review.md`'s line-number
citations (see the caveat below) made the churn disproportionate to the benefit.

| Phase | Name | Core action | Added by |
|---|---|---|---|
| -2 | Stub `progress.json` first | Literal first action; write the stub at the helper-resolved path | C1 |
| -1 | Sharpen the authorising prompt | Run `/coderails:improve-prompt`, ask once | (pre-arc) |
| 0 | Read the authorisation envelope | `<thinking>`: verbatim quote, envelope class, in/out-of-scope, **+ explicit yes/no on clean-break auto-demote authority (quoted, not inferred) — PR #86** | (pre-arc) + PR #86 |
| 0.5 | Orchestrator operating rules | Stop-ceremony: labels + DNV + `LOOP-STOP` together | C2 |
| 1 | State the plan in bullets | Ask once to confirm | (pre-arc) |
| 2 | Pre-flight via spawned agents | Delegate planning/premortem/wiki-query to sonnet agents | (pre-arc) |
| 2.5 | Resolve design forks up front | Ask once; record decision + flip-condition. Design agent applies [[brainstorming]]'s quality discipline (YAGNI, design-for-isolation, weigh viable approaches) **without** brainstorming's human-approval gates — see PR #41 note below. | (pre-arc) + PR #41 |
| 2.6 | Resolve disposition before replacement | clean-break vs preserve-compat; named blocker required | A |
| 2.7 | Commit resolved design to `spec.md` **and** `plan.md` **and** loop-scope `evals.json` (sub-steps 2.7a/2.7b/2.7c) | ≥3-unit guard stated once; 2.7a = `spec.md` write, 2.7b = `plan.md` via `coderails:writing-plans`, 2.7c = freeze loop-scope evals via `coderails:task-evals` | E, merged PR #86, +2.7c task-evals cluster |
| 3 | Delegate all impl to sonnet agents | Spawn a named team at ≥3 sequential units / dependency chain | (pre-arc) + A + D |
| 3a | Single sonnet agent for impl + verify | The spawned-team-is-overkill case | (pre-arc) + A + D |
| 4 | Spawn workers in waves | Check artifacts, never idle pings | (pre-arc) |
| 4b | PR review = invoke `/pr-review-toolkit:review-pr <PR#>` Skill; then invoke `/coderails:post-review <PR#>` | Skill required for enforce_pr_workflow gate evidence; post-review creates the SHA-bound artifact `/merge` gate-checks; clean-break compat hunt is a MERGE-BLOCKER the orchestrator **cannot self-demote** (PR #86) | A + PR #64 + PR #83 + PR #86 |
| 5 | Disprove the premise before each fix | Reproduce via SOT before spawning | (pre-arc) |
| 6 | Match confirmation to envelope | Don't ask inside authorised scope | (pre-arc) |
| 7&8 | Stack-specific deploy/push tactics | Collapsed to a generic stub | B |
| 9 | Cluster wiki ingest + docs-drift check | Run wiki ingest+lint once at loop end; then run `/sync-docs` (delegated agent) to audit in-tree docs drift | (pre-arc) + PR #77 |
| 10 | v2/v3 names when respawning | Versioned names identify the live agent | (pre-arc) |
| 11 | Agent prompts include confidence labels | Propagate the labelling standard | (pre-arc) |
| 12 | Status reports are claims, not evidence | Re-check artifact at moment of action | (pre-arc) |
| 13 | Confirm the factory ran (terminal self-audit) | Raw `loop_stop_counts` + unscored "decisions absorbed" list — **no numeric scorecard** (PR #86 dropped it); + unscored loop-scope eval result (task-evals cluster) | A + C2, rewritten PR #86, +eval bullet task-evals cluster |

**2.7/2.8 merge note:** Phase 2.8 no longer exists as a separate phase — its content (writing
`plan.md`) is now sub-step 2.7b under Phase 2.7. Phase 2.5 and 2.6 were deliberately left unmerged
(they fire unconditionally, and each has 6 inbound cross-references that a merge risked breaking
for no corresponding duplication removed — see [[pr_86_agentic-loop-hardening]]).

## The spec → plan → progress artifact chain (Spec E, merged PR #86)

The loop now writes three durable artifacts, all in the loop-state dir `~/.claude/agentic-loop/<cwd-slug>/` (resolved by `agentic_loop_path.sh`, outside the repo, uncommitted):

- `spec.md` (Phase 2.7, sub-step **2.7a** since PR #86) — the *already-resolved* design: envelope, design fork + flip-condition, disposition + named blockers, success criteria. A loop cannot brainstorm with itself, so this commits resolved design, it does not re-open it.
- `plan.md` (Phase 2.7, sub-step **2.7b** since PR #86, via `coderails:writing-plans`) — the durable decomposition. **The static SSOT** Phase 3 builds its task list from, and re-reads for *scope* after a compaction.
- `progress.json` (Phase -2 onward) — **the dynamic cursor** against `plan.md`.

Both sub-steps fire ONLY at the complexity guard: ≥3 work-units or a cross-unit dependency (Phase 3's own spawned-team threshold, pulled one phase earlier). Sub-threshold loops skip both entirely — ceremony on trivial work trains the loop to skip it on real work. **PR #86 merged what was Phase 2.7 and Phase 2.8 into one Phase 2.7** with the two sub-steps above, because both phases independently restated the identical complexity guard as their opening sentence — a verified duplication (same condition, gating two adjacent phases that were really one logical step: commit the resolved design to durable state, across two files). See [[spec-plan-progress-artifact-chain]], [[writing-plans]], and [[pr_86_agentic-loop-hardening]].

## Phase 2.7c — freeze loop-scope evals via task-evals (added by [[pr_1-4_task-evals-feature]], 2026-07-06)

A third sub-step joins 2.7a/2.7b, in the same loop-state dir alongside `spec.md`/`plan.md` (never committed): invoke `/coderails:task-evals` (scope: `loop`) to freeze the loop's end-state success evals into `evals.json`. Two independent triggers, stated explicitly because they don't collapse into one: (1) reaching Phase 2.7 at all is already tier-2-eligible on work-unit count alone, since 2.7 itself only fires at ≥3 units; (2) an irreversible-surface trigger (publish, deploy, migration, data deletion, external send) can independently apply even to a <3-unit loop that reached 2.7 via the cross-unit-dependency clause rather than the unit-count clause.

The same invocation also produces per-work-unit **pr-scope** eval refs (one per unit). These travel into worker prompts under the identical rule Phase 3 already applies to disposition: **a ref recorded only in `progress.json`/`plan.md` and absent from the worker's own prompt does not exist for that worker.** [[loop_state_guard]] is the hook that later enforces this artifact's presence at loop completion — see [[task-evals-gate]] for the full dual-scope architecture.

**Grading is a separate, neutral step from freezing** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #144]], 2026-07-12): freezing this file here (Phase 2.7c) only defines the evals; computing and stamping `result` at loop end is `post_evals.sh grade-loop`'s job (Phase 13), never the orchestrator's own. `grade-loop` also stamps a `grading` object (`by` + a checksum) that [[loop_state_guard]] checks before accepting a `GO`/`TIER0` verdict — an ungraded or hand-graded file now reads `UNSTAMPED` and blocks. See [[task-evals-gate]] for the full mechanism.

**The eval author's goal-state anchor at loop scope is `progress.json`'s `authorising_prompt_raw`** ([[pr_155-158_ceremony_noise_envelope_anchoring|PR #158]], 2026-07-13, extending [[task-evals]]'s rule 4 oracle-independence discipline): Phase 2.7c now cross-references this explicitly rather than leaving it implicit. This is a precedence rule, not a content denial — `spec.md`'s Phase 2.7a restatement and `plan.md`'s per-task restatement remain useful sources of constraints and concrete assertable surfaces, they just never override the envelope field as the anchor. See "Envelope anchoring for loop-scope evals" below for the full wiring across Phases -2/-1/2.7c/13.

## Envelope anchoring for loop-scope evals (PR #158, 2026-07-13)

`progress.json`'s `authorising_prompt_raw` field — the post-Phase-0 envelope, recorded verbatim — is now the **canonical, single-string anchor** [[task-evals]]'s rule 4 (oracle independence) points a loop-scope eval author at, closing an ambiguity: before this PR, an eval author had no stated rule for which of several plausible "goal state" sources to derive evals from (the original prompt, `spec.md`'s Phase 2.7a restatement, `plan.md`'s per-task restatement) — a paraphrase drifting during normal plan-writing could silently become the oracle, which is exactly the implementation-adjacent-oracle risk rule 4 already warned against for the non-loop case.

Four touch points wire this through the skill:

1. **Phase -2 stub schema comment** — `authorising_prompt_raw`'s inline comment now notes Phase -1 may update it if an improved prompt is adopted.
2. **Phase -2 mid-loop re-stub rule** — the existing rule carrying `loop_stop_counts` forward verbatim on a mid-loop re-stub (recovery after a restart) now also covers `authorising_prompt_raw`: "a re-stub refilled from conversation memory instead of the prior file's value would silently drift the eval author's canonical anchor." Same memory-vs-durable-record concern the `decisions_absorbed` carry-forward rule already applies elsewhere in this skill.
3. **Phase -1 (sharpen the authorising prompt)** — on adopting an improved envelope (outcome A: improved prompt adopted, or B: user tweak applied), the orchestrator updates `progress.json.authorising_prompt_raw` to the adopted text. Outcome C (proceed with the original prompt unchanged) needs no update since the Phase -2 stub already wrote it verbatim.
4. **Phase 13 teardown enrichment** — the `retro.json` assembly step's `envelope` field is now specified as sourced from `progress.json`'s `authorising_prompt_raw` specifically, not "verbatim from `progress.json`" generically.

The "Enrich at Phase 0" bullet in the `progress.json` lifecycle section (below) is worded consistently: "record the envelope verbatim in `authorising_prompt_raw`."

See [[pr_155-158_ceremony_noise_envelope_anchoring]] and [[task-evals-gate]] for the full mechanism.

## Orchestrator discipline demotes to warn inside an active loop (PR #155, 2026-07-13)

Phase 0.5 states the orchestrator is subject to the same discipline it imposes on workers. As of PR #155, the two discipline Stop hooks — `check_confidence_labels.sh` and `check_verify_loop.sh` — **demote** a would-be block to a model-visible `additionalContext` warn on the `Stop` event, when the session is inside an active, incomplete loop (the `als_loop_active_incomplete` predicate in `loop_state_common.sh`). Outside an active loop, and for worker output (`SubagentStop`), both hooks still block outright — this is not a general relaxation of the discipline, only a change to how the orchestrator's own in-loop Stop turns are enforced. "The discipline itself hasn't changed, the warn is the correction signal the orchestrator acts on next turn."

**Motivating incident:** loop f87a0a2e forced ~69 turn regenerations in one crack-on loop (49 confidence-label blocks + 20 verify-loop blocks) — each block costing a full model regeneration cycle for discipline that was correct in substance but delivered at a cost mismatched to an autonomous loop's cadence. Even at warn-level a missed warn is still a cost, "just paid as a drifted transcript instead of a forced regeneration" — the skill's framing is explicit that this is a cost-shape change, not a removal of the underlying requirement.

Full mechanism (the predicate's truth table, lazy evaluation, fail-toward-blocking `jq` emission, and the `would_block=1 warned=1 blocked=0` log-line accounting that future retro mining should key on `blocked=1` counts rather than `would_block=1` counts) is documented on [[check_confidence_labels]], [[check_verify_loop]], [[discipline-loop]], and [[pr_155-158_ceremony_noise_envelope_anchoring]] — not duplicated here.

## The LOOP-STOP stop-ceremony contract (Spec C2)

When the loop is active and incomplete, the orchestrator cannot stop without a declaration in its stopping turn:

```
LOOP-STOP: <hard-stop|approval-gate|awaiting-input|complete> — <reason>
```

Phase 0.5 bundles this with the confidence-label and DNV requirements into one stop-ceremony, so the orchestrator emits all three together rather than thrashing one Stop hook while satisfying another. The four categories map onto the Stop-conditions section. **Declaring `complete` is atomically the Phase 13 teardown** that sets `progress.json status: complete` — a text-only `complete` leaves both loop-state hooks treating the loop as active. Enforced mechanically by [[loop_stall_guard]].

**The declaration must be the FINAL line of the turn** ([[pr_155-158_ceremony_noise_envelope_anchoring|PR #157]], 2026-07-13 — a prose clarification, not a new rule): "that ending-line position is the contract this skill defines and the hook's category accounting assumes: when a turn carries more than one LOOP-STOP-shaped line (e.g. a quoted example), `loop_stall_guard` counts only the last one, so the last line must be the declaration that reflects the turn's actual outcome." `loop_stall_guard`'s tail-1 read only decides *which* category among multiple matching lines gets counted — it does not reject a declaration placed earlier in the turn outright; the skill text is what establishes ending-line placement as the expected authoring contract, and the hook's tail-1 behaviour is why getting that placement right matters (a misplaced declaration risks the wrong category being counted, not an outright rejection).

**Warn-era bundling (PR #157).** Before PR #155 demoted the confidence-label and verify-loop Stop hooks to warn-level inside an active loop (see below), Phase 0.5 framed the bundle as avoiding "thrashing one Stop hook while satisfying another" — both hooks still hard-blocked. After #155, bundling matters *more*, not less: those two hooks no longer block the orchestrator's in-loop Stop turns, so nothing else forces the confidence labels and DNV tags into the transcript except the orchestrator's own discipline — "the bundle is what keeps them present for post-hoc audit, and one composed ending beats clearing one stop hook only to trip another (`loop_stall_guard` still blocks)."

## The clean-migration disposition fork (Spec A)

Phase 2.6 forces a disposition decision before replacement work: **clean-break** (remove the old path; no shims) vs **preserve-compat** (keep it behind a shim, with a named blocker and a mandatory removal ticket). clean-break is the stated default; preserve-compat requires a *specific named blocker* (anti-laundering — a generic "safer" is rejected). The decision is propagated verbatim into the worker prompt (Phase 3), and the **independent code-simplifier reviewer is the load-bearing gate** (Phase 4b) — it hunts relabelled compat (fallback/adapter/guard/bridge) and whether an old path still executes; findings are MERGE-BLOCKERS. The worker's own assertion (Phase 3a) is a smell test, not the gate. Phase 13 counts `disposition-violations`, distinguishing "0 violations" from "no record found" (the latter = audit failure).

**The disposition decision itself is now also appended to `progress.json`'s `decisions_absorbed` array** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]], 2026-07-12): `{phase: "2.6", decision: "<clean-break or preserve-compat, with named_blocker if applicable>"}`. See "Phase 13 ... `decisions_absorbed` trace" below for why this array exists and how it's consumed.

## Phase 4b self-attestation loophole closed (PR #86)

Before PR #86, the orchestrator could unilaterally demote a clean-break compat `MERGE-BLOCKER` —
raised by the independent `code-simplifier` reviewer — to a logged note, by writing free-text
"reviewed, not compat — `<reason>`". **The party doing the demoting was the same party whose worker
had just shipped the compat path and had the motive to keep the shortcut.** No counter-check existed
on the override text (verified: [[pr_86_agentic-loop-hardening]] §3.3 of the design spec).

An alternative that kept self-demotion but required the override to cite a re-checkable file:line
fact, logged for a later independent auditor, was rejected: a fabricated-but-plausible citation
costs the same effort as a true one, and nothing re-runs the check at override time — that only
makes bad overrides auditable *after the fact*, not prevented, when outright prevention was
available at small cost (an occasional hard-stop).

**Decision:** the orchestrator's self-demote power is removed entirely. On a clean-break unit, an
independent reviewer's compat finding is a `MERGE-BLOCKER` the orchestrator **cannot** downgrade
unilaterally. Its only two moves:
- (a) actually fix it — remove the compat path, or
- (b) declare a hard-stop and hand it to a human, logged with who/when/SHA/reason.

**The one carve-out:** a fully-unattended envelope that genuinely cannot tolerate ever hard-stopping
at this gate must have auto-demote authority granted explicitly **at Phase 0 envelope-authorisation
time** — never something the orchestrator grants itself mid-run. Phase 0's `<thinking>` template now
has a dedicated line forcing an explicit yes/no answer with a verbatim quote of the authorising
clause (added by a post-merge Important-severity review finding — quoting prevents the orchestrator
from inferring the carve-out from a general "fully autonomous" classification instead of an actual
grant).

This is the same enforcement-boundary pattern documented in [[enforcement-model]]: a check performed
by the party with motive to pass it is not a check. See [[pr_86_agentic-loop-hardening]] for the
full decision record and the 2 post-merge Critical stale-cross-reference fixes this change required.

## progress.json schema growth across the arc

| Field | Added by | Purpose |
|---|---|---|
| `disposition`, `named_blocker`, `removal_ticket` | A | per-work-unit clean-break record |
| `schema_version`, `session_id`, `status`, `created`, `last_updated`, `completed_marker` | C1 | lifecycle + presence/ownership |
| `loop_stop_counts` (`{hard-stop, approval-gate, awaiting-input, complete}`) | C2 | per-category counts; Phase 13 reports these raw since PR #86 (no scorecard — see below) |
| `work_units` (JSON object keyed by unit id, each carrying at least a `status`) | task-evals cluster | [[loop_state_guard]]'s eval gate reads `.work_units \| length` off this field to decide whether the ≥3-unit eval threshold applies; fails open (no block) when the field is absent — so it must stay populated whenever the loop tracks ≥1 work-unit |

`status` ∈ `initialising` | `in-progress` | `complete`. The path is resolved by `agentic_loop_path.sh` — **the model never computes it** (a cwd-slug cannot be reproduced by hand).

## Construction discipline (Spec D + PR #24)

Phase 3 and Phase 3a reference `coderails:test-driven-development` (code-guarded: "if the change adds or alters a function, method, or branch that can carry a test"). Vendored as a coderails-owned skill so the plugin keeps zero cross-plugin dependency (REVERSED from Spec A's "reference, not vendor" note). The reference sits near the TOP of the Phase 3a prompt-contract list (Phase 9's placement lesson: scope-shaping instructions get shortcut when buried low). PR #24 additive-wired `coderails:subagent-driven-development` into worker-prompt construction (Phase 3), replacing the former superpowers cross-plugin reference. Also fixed dead `/claude-guardrails:assumptions` and `/claude-guardrails:notchecked` references → `coderails:assumptions` and `coderails:notchecked` respectively. The six C1/C2 no-touch regions were kept byte-identical. (verified — PR #24) See [[test-driven-development]] and [[subagent-driven-development]].

## Delegation rung: single agent vs. spawned team

Updated 2026-06-01 (verified: SKILL.md Phase 3 and Phase 3a); terminology updated by [[pr_1-4_task-evals-feature]] (PR #4, 2026-07-06) — the `TeamCreate`/`TeamDelete` tools this section used to name no longer exist (removed in Claude Code v2.1.178).

**Main context is a pure orchestrator that NEVER implements.** Every code change — even a single-file edit — is delegated to a spawned sonnet agent. The two reasons, stated in the skill: keep main context clean (opus context is scarce), and keep cost down (sonnet does the typing, not opus). A file edit done directly in main context is the exception that needs a justification, not the default.

The delegation decision is a two-rung ladder:

**Rung 1 — Single sonnet `Agent` (default for 1–2 self-contained units):**
- A bug fix, one PR, a single-file change, a tight sequence with shared context
- One `model: sonnet` agent owns both implementation AND verification before reporting back
- Why one agent does both: verification output is the dense kind you delegated to keep out of main context; if main context re-verified every small change, it refills. Agent self-verifies; main context spot-checks only at dependency boundaries (Phase 12)
- See Phase 3a for the prompt contract

**Rung 2 — Spawn a team (for ≥3 PRs or cross-step dependency chains):**
- Spawn each worker as a **named teammate** via the `Agent` tool; build a shared task list with explicit `blockedBy` dependencies via `TaskCreate`/`TaskUpdate`; coordinate between teammates with `SendMessage`
- If the user has explicitly asked for a spawned team in their prompt, it is non-negotiable — spawn named teammates even if a flat sequence of solo `Agent` calls would technically work

The old guidance "work directly when: single-file edits / sequential steps" was removed in plugin commit `3c33f99` because it contradicted the main-context-is-pure-orchestrator rule. The correct frame is which rung, not whether to delegate. (This is the same two-rung threshold the Phase 2.7/2.8 complexity guard reuses for the spec/plan artifacts.)

## Context-window persistence

(verified: SKILL.md). Do not stop work early because the context window is filling — context compacts and the session continues. Before compaction, checkpoint: commit in-progress work to git, update `progress.json`, record the loop's phase position. Git is the authoritative checkpoint. (Post-arc: the orchestrator also re-reads `plan.md` for *scope* after compaction — stated in Phase 2.7b (formerly 2.8), deliberately NOT in this section, which is a Stop-hook no-touch region describing `progress.json` alone.)

One sentence was added to this section by [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #16]] (WU4), noting that `sdd-ledger.md` is a sibling file in the same session-keyed loop-state dir as `progress.json`, written by [[subagent-driven-development]]'s own `sdd-workspace` helper — not by this skill. This is documentation of an adjacent fact, not a change to the no-touch region's own instructions to the model.

**Concurrent loops in one directory — fixed by PR #87 (was: single-loop-per-directory invariant, PR #86).** `progress.json` is now keyed on project working directory **and session_id** (Phase -2), not cwd alone — two `agentic-loop` sessions running concurrently in the same checkout each get their own file automatically; no race, no last-writer-wins (verified: [[pr_87_agentic-loop-path-session-keying]], `hooks/scripts/lib/agentic_loop_path.sh`). This relies on a platform guarantee that can't be verified from repo source alone: Claude Code's `session_id` stays stable for the life of one continuous conversation — including across that conversation's own compaction/restart — while differing between genuinely separate conversations. `loop_state_guard.sh`'s session-mismatch check is retained but now narrower: it no longer catches the routine cross-session race (that no longer happens by construction), only a rarer case where a file's content disagrees with its own session-scoped path — a corruption signal (copied, hand-edited, or otherwise tampered content), not the routine race it used to catch.

Git worktrees ([[using-git-worktrees]]) are **no longer required** to avoid this specific collision, but remain the right tool when concurrent loops must not see each other's working-tree changes — a different concern (working-tree isolation) than progress.json ownership.

**The PR #87 fix itself shipped a residual bug, caught by review in the same PR.** The initial commit fixed only `agentic_loop_path.sh`'s own session_id fallback; the two guard scripts' `jq` extraction still fell back to a shared `"?"` sentinel on a missing/null session_id, which would make multiple sessions hitting that edge case collide again. A follow-up commit in the same PR (`8cc9d63`) closed this with a `als_sanitise_session_id()` helper generating a unique PID+timestamp fallback, applied consistently across all three call sites. See [[pr_87_agentic-loop-path-session-keying]] for the full account — instructive because a fix for a race condition itself had a residual race-like edge case, found by independent review rather than the original author.

Historical note: the original PR #86 design session verified live that a leftover `completed` `progress.json` from an earlier finished loop blocked a new session's Stop hook until manually re-stubbed — the trigger case that led to documenting (not yet fixing) the invariant at the time.

## Stop conditions

The loop runs autonomously until ANY of (verified: SKILL.md): verification failure that survives the bounded retry-until-green cycle below; premise disproven (Phase 5); genuinely ambiguous decision outside the envelope; destructive/irreversible op not authorised; all authorised work complete. On stop: report state with confidence labels, emit the `LOOP-STOP` declaration, propose the next move, then wait. Model an approval-gate as "pause-then-proceed," never "do not start."

**Retry-until-green, added by [[pr_134_agentic-loop-retry-until-green|PR #134]] (2026-07-10), extracted to a sibling file `retry-until-green.md`.** A single failing test/lint/verification check is not, by itself, grounds to stop and ask — diagnose, fix, re-verify in a bounded cycle (default 5 distinct attempts; re-trying an identical fix without a new diagnosis doesn't count as a new attempt). Only escalates to the verification-failure hard-stop once the bound is exhausted and the failure still reproduces. This narrows how often that hard-stop fires; it does not remove it, and does not touch the other three. Born from an explicit owner request for zero-question autonomy — the orchestrator refused to remove the 4 hard-stops entirely (named the runaway-loop risk: pushing through a broken suite or force-pushing with nobody watching) and negotiated this narrower version instead. Three vendored skills are now wired into the cycle: `coderails:systematic-debugging` when a failure's cause isn't obvious (the Phase 5 tie-in the "Slimming v2" section below already anticipated, now made concrete rather than implicit), `coderails:dispatching-parallel-agents` when a verification failure turns out to be 2+ independent broken things, and `coderails:disconfirm` (new) as a pre-implementation check on a diagnosed fix.

## Slimming (Spec B)

The skill was cut 454→434 lines: Phases 7&8 (corporate docker/Teleport residue) collapsed to one generic stub; 16 war-stories compressed to one-clause `Past failure:` tags; verbose "why" paragraphs trimmed. **Six no-touch regions** (the frontmatter description, Phase -2 stub block, Phase 0.5 LOOP-STOP bullet, Phase 13 KPI bullet, Stop-conditions "Declaring the stop" block, and the `## Context-window persistence` section) are byte-stable — they teach the exact behaviour the Stop hooks check, so editing them would turn a hook from a safety net into a stall generator. The byte-diff against `origin/main` is the primary verification gate; token greps are necessary-not-sufficient. (PR #18 separately dropped `tsh ssh` from the Phase 4/12 illustrative examples — tool-agnostic cleanup, no contract change.)

**Update (PR #86):** the Phase 13 KPI no-touch region above described the *pre-hardening* scorecard
("human turns approaching zero" target). PR #86 deliberately rewrote that region as part of its §3.4
decision (see the Phase 13 section below) — this is an intentional supersession of the slimming-era
no-touch content, not a violation of the no-touch convention, because the content itself was the
finding being fixed. The other five no-touch regions are untouched by PR #86.

## Slimming v2 — reference-not-embody is exhausted (PR #39)

A follow-up tried to shrink the skill further by replacing inline restatements with one-line references to the vendored `coderails:` skills. **The realistic delta was zero.** [[pr_39_agentic-loop-slim-v2|PR #39]] normalised 13 bare skill refs to the fully-qualified `/coderails:` form (so they resolve as invocable references) — and that was the entire actionable surface. Two independent passes (main context + a blind Explore agent) classified all 22 phases against a four-part test — **generic + fully-covered + worker-facing + no-autonomy-delta** — and both converged on **zero** replaceable passages. The embodied skills are autonomy *supersets, not duplicates*: every phase fails part 3 (orchestrator-facing) or part 4 (superset of the vendored skill), or is frozen. The last near-miss, Phase 5, is KEEP — its "reproduce before fixing" core is `coderails:systematic-debugging`'s Iron Law, but it is wrapped in an autonomy delta (the source-of-truth caching warning + the STOP-and-report hard-stop tie-in). **Do not re-attempt the slim:** further cuts would only delete autonomy deltas the constraints exist to preserve. See [[pr_39_agentic-loop-slim-v2]].

## Phase 2.5 — brainstorming design-quality by reference (PR #41)

Phase 2.5 spawns a design agent that applies [[brainstorming]]'s design-quality discipline — weigh viable approaches, YAGNI, prefer designs whose units stay small and independently testable (design-for-isolation) — **without** invoking the brainstorming skill itself. (verified: SKILL.md line 163)

The rationale: brainstorming is human-gated by construction — its approval steps require a human in the loop. An autonomous loop can't pause at those gates; it would stall. So Phase 2.5 reuses brainstorming's *thinking* by reference, not its control flow. This is the same reference-not-embody pattern established by the Slimming v2 work: the loop borrows the design principles without being able to invoke the gated skill. (verified: SKILL.md Phase 2.7 "a loop cannot brainstorm with itself")

The non-obvious/durable point: if anyone asks "why doesn't the autonomous loop just invoke brainstorming?" — the answer is that brainstorming blocks on a human at its approval gates. The loop reuses its design *quality criteria* at Phase 2.5 rather than calling the gated skill. See [[pr_41_phase25-brainstorming-xref]] and [[brainstorming]].

**Under full-autonomous envelopes, the auto-adopted design fork is now also recorded** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]], 2026-07-12): `{phase: "2.5", decision: "<chosen shape + flip-condition>"}` appended to `progress.json`'s `decisions_absorbed` array, alongside the pre-existing `progress.json` note of the chosen shape.

## Phase 4b — review-pr Skill + post-review artifact (PR #64 + PR #83)

Phase 4b previously described hand-rolling six toolkit reviewer agents as parallel `Agent` or `Task` spawns. PR #64 changed this: **Phase 4b now invokes `/pr-review-toolkit:review-pr <PR#>` as a Skill, passing the PR number as the argument.** PR #83 extended Phase 4b with a second step: **`/coderails:post-review <PR#>` runs after `review-pr`** to create the durable SHA-bound artifact.

**Why both steps are required:**

1. **`/pr-review-toolkit:review-pr` as a Skill** — `enforce_pr_workflow.sh` only recognises Skill invocations (with the PR number in args) as valid merge-gate evidence. A manually-spawned agent fanout leaves no evidence the gate can see. The merge will block. (verified: PR #64; source: [[pr_64_loop-review-via-skill]])

2. **`/coderails:post-review <PR#>`** — `merge.sh` gate-checks for a coderails review comment on the PR matching the current head SHA. Without this artifact, `/merge` blocks regardless of whether `review-pr` ran. This is the new fail-closed SHA-bound gate. (verified: PR #83; source: [[pr_81-83_review-artifact-seam]])

The orchestrator's job at Phase 4b:
1. Invoke `/pr-review-toolkit:review-pr <PR#>` as a Skill.
2. Collect the returned aggregated findings (Critical / Important / Suggestion).
3. Feed any MERGE-BLOCKER to a fix agent (Phase 5/10) before proceeding.
4. Invoke `/coderails:post-review <PR#>` — posts the SHA-bound review artifact to GitHub.

The "do not substitute the generic trio" warning (architect-review + debugger + ai-engineer) still applies — those are design stress-test agents, not the PR-review step.

See [[enforce_pr_workflow]] for how it recognises Skill invocations. See [[review-artifact-seam]] for the artifact gate design. See [[skills-hooks-seam]] for the general seam convention.

**Review tier ladder** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #146]], 2026-07-12): regardless of the PR's own eval-artifact tier (Phase 4b's eval gate is a separate, orthogonal check), every tier invokes `/pr-review-toolkit:review-pr <PR#>` plus `/coderails:post-review <PR#>` — the toolkit self-scales its reviewer fan-out by change shape. Only at **tier 0** may the separate native `/security-review` pass be skipped, and only after checking the actual diff file list (`gh pr diff <PR#> --name-only` or `git diff origin/main...HEAD --name-only`): any path under `hooks/` or `scripts/`, or any auth/exec/network-fetch touch, **forces** the security pass regardless of the declared tier. The override keys off the diff, never the self-assigned tier label — reusing a self-assigned tier alone to gate a security control would be the same self-exemption shape the task-evals tier rules exist to resist (see [[task-evals]]'s tier rules). Tier 1/2 PRs run the full Phase 4b unchanged, security pass included. Concrete example named inline: the `silent-failure-hunter` toolkit reviewer caught a real swallowed-crash-payload bug on the one-function PR #142 fix — why the toolkit review itself is never skippable, only the separate `/security-review` pass is.

## Phase 9 — cluster wiki ingest + in-tree docs-drift check (PR #77)

Phase 9 now has two steps at the loop boundary:

1. **Wiki ingest + lint** (the existing cluster step) — updates the external knowledge base ([[wiki-ingest]] + [[wiki-lint]]). Runs once per loop, not per PR.
2. **[[sync-docs|`/sync-docs`]] docs-drift check** (added PR #77) — audits the repo's own in-tree docs (README.md, AGENTS.md, docs/REFERENCE.md, etc.) for drift against the just-merged code. Note: this is a user-level skill (`~/.claude/skills/sync-docs`), not part of the coderails plugin itself — see [[sync-docs]] for the location detail.

**The critical distinction:** wiki ingest is external-KB maintenance; `/sync-docs` is in-tree-docs maintenance. These are complementary, not redundant. Both run once at the loop boundary; wiki ingest runs first.

**Serena (`--semantic`) is optional.** Run `/sync-docs` even without Serena — omit `--semantic` for the traditional file-comparison audit, which still catches real drift (stale command names, removed flags, changed config keys). Do not skip `/sync-docs` because Serena is absent.

**Both steps are delegated to spawned agents** — same as Phase 3/3a implementation delegation. Keeps orchestrator context clean.

**Findings triage for `/sync-docs`:** Fix only drift the loop's own PRs introduced. Pre-existing drift is surfaced to the user, not silently absorbed. Folding unrelated doc fixes into the loop is scope creep. (This is the same triage discipline as Phase 5's finding-triage for code.) (verified: PR #77 diff)

This step is advisory — no hook enforces it. See [[enforcement-model]].

## Phase 13 — terminal self-audit rewritten to drop the gameable scorecard (PR #86)

Before PR #86, Phase 13 reported "human turns approaching zero" as a target metric. The problem
(§3.4 of [[pr_86_agentic-loop-hardening|the hardening spec]]): that reading is identical whether the
orchestrator was well-calibrated (correctly absorbing every in-scope decision) or silently suppressed
asks it should have made — the metric can't tell the two apart.

An alternative — spawn a fresh independent agent to audit the same `progress.json` fields and issue
a "calibrated zero" vs "suppressed zero" verdict — was rejected: that auditor's only inputs are still
orchestrator-authored records, so it grades homework against homework. Worse, a clean automated
verdict can look *more* trustworthy than an honest raw list while being equally gameable — the safe
strategy under that scheme becomes padding the record with trivial non-decisions to look thorough
(a Goodhart's-law failure mode).

**Decision:** drop the numeric pass/fail scorecard entirely. Phase 13 now reports two raw, unscored
facts:
- **`LOOP-STOP` category counts** — already artifact-backed from `progress.json`'s `loop_stop_counts`,
  hard to fake. Reported with no verdict attached.
- **"Decisions absorbed"** — a flat, unscored list of in-scope decisions the loop made autonomously
  without asking. No self-justification text, no automated "this looks calibrated" stamp.

The human is the only party positioned to judge "should I have been asked about that?" — hand them
the raw list rather than have the process pre-grade itself. A clean-looking scorecard is more
dangerous than an honest unscored list because it's more likely to be trusted uncritically; the two
raw facts can only be gamed by omission, which a human spots more easily in a flat list than in a
fabricated scorecard pass.

Phase 13 still separately reports **artifacts produced** (verified, not claimed) and **disposition
violations** (unchanged from Spec A) — only the human-turns scorecard was removed.

**Loop-scope eval result bullet (added by [[pr_1-4_task-evals-feature]], 2026-07-06).** Phase 13 gains a fourth raw fact, reported unscored alongside the three above: the loop's final `evals.json` `result` (`GO` / `NO-GO` / a justified tier-0 exemption), plus any `amendments` (post-freeze eval edits with recorded reasons). Same framing as the disposition-violation bullet: **"no `evals.json` record found" for a ≥3-work-unit loop is an audit failure, not a pass** — explicitly distinguished from a genuine `GO`, mirroring how this section already distinguishes "0 disposition violations" from "no disposition record found." No self-issued verdict is layered on top of the raw result. As of [[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #144]] (2026-07-12), this `result` is graded via `post_evals.sh grade-loop` — never hand-written into `evals.json` by the orchestrator — and reported as such.

**"Decisions absorbed" is now a durable trace, not a memory reconstruction** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]], 2026-07-12). Before this PR, the bullet above was assembled from conversation memory at teardown time — exactly the kind of after-the-fact self-report Phase 13 otherwise exists to avoid. `progress.json` now carries a `decisions_absorbed` array of `{phase, decision}` objects, appended chronologically (oldest-first) at the phase boundary where each in-scope autonomous decision is made: Phase 2.5 (design-fork auto-adopted), Phase 2.6 (disposition defaulted), Phase 5 (a consciously-absorbed `/coderails:disconfirm` skip), Phase 6 (a notable in-scope action taken without a check-in). Phase 13's "Decisions absorbed" report bullet is now specified as **copied verbatim** from this array — never reconstructed from memory. The `retro.json` teardown artifact (below) carries the same array verbatim into its own field, by the same rule.

**`loop_stop_counts` carry-forward is now conditional, not unconditionally verbatim** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]], 2026-07-12): on any wholesale `progress.json` rewrite, the orchestrator must re-read the existing file first, then carry `loop_stop_counts` forward **verbatim** on a genuine mid-loop recovery rewrite, but **reset to `{}`** when the prior file's `status` was `"complete"` — i.e. a fresh re-arm after a prior loop already finished. Before this fix, a brand-new loop starting in the same repo/session-key slot could wrongly inherit an already-finished prior loop's stop-counts.

**Post-merge fix:** the initial hardening pass left two stale cross-references (`SKILL.md:435` and
`:480`) that still described the pre-rewrite scorecard shape after the rest of Phase 13 changed —
caught and fixed by the post-merge review, inside the same merge commit. See
[[pr_86_agentic-loop-hardening]].

## Phase 13 now writes a retro + maintains a standing-orders overlay (PR #118-123)

The self-improving-loops cluster gave Phase 13 a **write contract**, and gave the loop a **read-back** at Phase 2. Teardown order: (1) assemble `retro.json` (`schema_version` 1) beside `progress.json` — raw, unscored, no verdict field; `loop_stop_counts` copied verbatim (hook-owned), `hook_blocks` mined by `dc_mine_hook_blocks` from `discipline.log`, disposition record (distinguishing "0 violations" from "no record"), evals result, artifacts; (2) update the repo-keyed, machine-owned `standing-orders.md` overlay — additive-or-recurrence-only, decay at `loops_since_recurrence >= 5` by MOVING to a tombstone file (never delete); (3) write feedback auto-memories; (4) only then declare `complete`. **The `complete` declaration is now gated** — [[loop_stall_guard]] blocks it unless the retro.json exists and parses (the feature gates its own loop's completion). Phase 2 pre-flight reads the last N=5 retros + the overlay and carries lessons **verbatim** into worker prompts (additive-only: may add cautions/assertions/premortem entries, never relax a gate or skip a phase). A **dormant v2 pipeline** ([[pr_118-123_self-improving-loops]]) will, once a graduation predicate fires (≥10 retros + one lifecycle + one clean decay), promote repo-agnostic lessons into `learned-failure-modes.md` through the full gate chain. rollback-on-regression was explicitly cut — the feedback path is additive-or-recurrence-only, never a self-issued performance verdict. See [[pr_118-123_self-improving-loops]].

**Concrete example (2026-07-10 loop, [[pr_130-136_dashboard-right-rail-ux]]):** two new SO entries were appended to the repo-keyed `standing-orders.md` overlay by this loop's own Phase 13 teardown — SO-2 (warn every still-running parallel worker about base-staleness proactively, once one sibling PR hits it, rather than after each one independently discovers it) and SO-3 (verify a worker-dispatch's branch is genuinely isolated from the orchestrator's own working docs before dispatching, not after a precondition check catches contamination). Both were live near-misses this same loop hit and fixed, not hypothetical.

## `model: sonnet` is advisory, not hook-enforced (PR #86, documentation only)

`AGENTS.md`'s "Enforcement ceilings" list gained a new bullet (no code change — a documentation
decision, §3.5 of the hardening spec): no hook gates `Agent`/`Task` spawn calls on the requested
model, even though `SKILL.md` asserts `model: sonnet` for workers roughly 6 times (Phases 2, 2.5, 3,
3a, 10). This is deliberate, not a gap to close:

- The rule's purpose is **cost control, not correctness** — an opus worker still produces a valid,
  fully-gated PR; nothing load-bearing breaks if a worker runs on the wrong model.
- Phase 2.5 sanctions a **legitimate opus-escalation exception** ("escalate the synthesis to opus
  only if the tradeoff is genuinely close") that a blunt model-gate hook cannot distinguish from a
  disallowed worker spawn without a self-reported carve-out flag — which reintroduces the same
  trust-the-agent problem the hook was meant to remove, just one level down.

See [[enforcement-model]] for the general hooks-vs-advisory distinction this decision extends.

## Key architectural decisions encoded

- **Pre-flight + worker agents use `model: sonnet`** — orchestration pattern; cost control. No escalation path except Phase 2.5's sanctioned opus-escalation exception; documented as a deliberate advisory (not hook-enforced) ceiling as of PR #86.
- **Wiki ingest clusters, not per-PR** — Phase 9; fragmented ingests produce fragmented wiki context.
- **`/sync-docs` is the in-tree complement to wiki ingest** — Phase 9 (PR #77); the two were previously conflated as "docs", now separated at the loop boundary.
- **Scope-shaping instructions go high in worker prompts** — Phase 9 lesson, reused by D's TDD placement.
- **Artifact verification not idle pings** — Phase 4/12.
- **The model never computes a hook-derived value** — path (C1) and LOOP-STOP tag format (C2) both come from the hook's block message.
- **Phase 4b requires BOTH review-pr AND post-review** — PR #83: `enforce_pr_workflow` gates on the review-pr Skill event; `/merge` gates on the SHA-bound artifact from post-review. Missing either one blocks the merge. (verified: PR #64 + PR #83)
- **The orchestrator cannot self-demote an independent reviewer's finding** — PR #86 Phase 4b: the party with motive to keep a shortcut must never be the party grading whether the shortcut is acceptable. Fix-it or hard-stop; the only override path is a pre-granted, quoted Phase 0 carve-out.
- **Self-audits report raw facts, not self-issued verdicts** — PR #86 Phase 13: a scorecard graded by the graded party (or an auditor reading only that party's own records) is gameable and a clean-looking verdict is more dangerous than an honest unscored list.

## Description trimmed to the skill-listing truncation cap (PR #89)

`SKILL.md`'s frontmatter `description` was trimmed from 1533 to 1134 characters (PR #89,
merged 2026-07-03). Claude Code truncates skill-listing descriptions at 1,536 characters —
the original text sat close enough to that cap that a review round flagged it. The trim
preserved every trigger phrase verbatim ("TeamCreate", "spawn a team", "no human gates",
"self-merge", "crack on", "without the human", "no per-PR confirmation", "agentic loop",
"multi-PR") and the 3+ PRs / autonomous-chain triggers; only the explanatory prose describing
*how* the phased method works was compressed. One clause was cut and then **restored** after
review: the "sonnet" delegation rule ("every code change ... goes to a sonnet agent that does
the implementation AND verifies its own artifact") — an initial trim draft dropped it, and
review caught that this is load-bearing trigger-adjacent content (it's the rule Phase 3/3a
enforce), not prose padding. (verified: [[pr_89-91_skills-doc-frontmatter-injection]], `git diff` on `skills/agentic-loop/SKILL.md`)

No phase content, trigger phrase, or behavioural rule changed — this is a frontmatter-only
edit to the listing description Claude Code shows before the skill body loads.

## Cross-references

- [[spec-plan-progress-artifact-chain]] — the design page for the artifact chain + two-hook guard
- [[review-artifact-seam]] — design page for the SHA-bound PR comment gate (PR #82/83)
- [[post-review]] — the `/coderails:post-review` command Phase 4b now invokes after review-pr
- [[loop_state_guard]] — C1 hook: presence/ownership of `progress.json`
- [[loop_stall_guard]] — C2 hook: the `LOOP-STOP` declaration enforcement
- [[test-driven-development]] — vendored construction skill (Phase 3/3a)
- [[writing-plans]] — vendored plan skill (Phase 2.7b, formerly Phase 2.8)
- [[using-git-worktrees]] — working-tree isolation between concurrent loops; no longer required to avoid the progress.json collision (fixed by PR #87), still useful for its original purpose
- [[pr_87_agentic-loop-path-session-keying]] — PR #87 source record: progress.json keyed on cwd+session_id, fixing the race PR #86 had only documented; includes the same-PR Critical fix for a residual fixed-sentinel fallback collision
- [[pr_89-91_skills-doc-frontmatter-injection]] — PR #89 source record: `description` trimmed to the skill-listing truncation cap, sonnet-delegation clause restored after review
- [[subagent-driven-development]] — vendored execution skill (Phase 3 worker-prompt, PR #24)
- [[enforcement-model]] — hooks (mechanical) vs. skill prose (advisory)
- [[discipline-loop]] — how the discipline hooks compose
- [[self-containment]] — the broader initiative PR #24 is part of
- [[session_2026-06-01_agentic-loop-delegate-all-impl]] — source for the delegate-all-impl-to-sonnet change (plugin commit 3c33f99)
- [[session_2026-06-25_agentic-loop-upgrade-arc]] — the PR #12–#18 source record
- [[session_2026-05-31_prompting-doc-alignment]] — earlier source for delegate/context guidance
- [[pr_41_phase25-brainstorming-xref]] — PR #41 source record: Phase 2.5 references brainstorming's design-quality discipline without its gates
- [[pr_64_loop-review-via-skill]] — PR #64 source record: Phase 4b changed to invoke review-pr Skill (with PR number) instead of hand-rolling agents
- [[pr_77_agentic-loop-sync-docs-step]] — PR #77 source record: Phase 9 gains `/sync-docs` in-tree docs-drift check at the loop boundary (distinct from wiki ingest)
- [[sync-docs]] — the skill's own home page (what it does, its location outside the coderails repo)
- [[pr_81-83_review-artifact-seam]] — PR #83 source record: Phase 4b extended with `/coderails:post-review` for loop symmetry
- [[pr_86_agentic-loop-hardening]] — PR #86 source record: 7 hardening decisions (stage-map, Phase 2.7/2.8 merge, Phase 4b self-demote removal, Phase 13 scorecard drop, `model: sonnet` advisory doc, single-loop invariant **documented not fixed — see PR #87 for the fix**, stale memory-citation cleanup) + 2 post-merge Critical stale-cross-reference fixes + 1 Important Phase 0 carve-out fix
- [[unregistered_loop_guard]] — new sibling Stop hook (PR #17): nudges when a session looks like an unregistered instance of this skill (dispatch-shaped, no `progress.json`, no Skill invocation)
- [[loop-progress-fields]] — consolidating page for `progress.json`'s `work_units` and `loop_stop_counts` fields this skill reads/writes
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #16 source record: the `sdd-ledger.md` sibling-file sentence in Context-window persistence; PR #17 source record: the unregistered-loop nudge hook this skill's non-invocation is detected against
- [[dashboard]] / [[pr_25_observability-dashboard]] — PR #25 (2026-07-06): the observability-dashboard skill built under a registered loop (loop-scope `evals.json` GO 4/4, Phase 2.7c). Its own frozen pr-scope Tier-2 eval suite caught two production bugs no review round found — the first concrete demonstration of the task-evals gate paying for itself in this loop
- [[verify-merged-pr]] — re-derives a "PR #N is merged" claim (state + content on origin/main + sibling PRs) before an orchestrator relies on a worker's or subagent's merge report; a process-learning discipline this skill's own multi-PR loop shape makes directly applicable, since a headless builder or teammate reporting "done — merged" is exactly the trust boundary this skill's delegation model creates
- [[pr_134_agentic-loop-retry-until-green]] — PR #134 source record: retry-until-green + systematic-debugging/dispatching-parallel-agents/disconfirm wiring + the file-size extraction to `retry-until-green.md`
- [[pr_130-136_dashboard-right-rail-ux]] — the loop PR #134 shipped inside; six independent dashboard UX fixes plus this skill's own mid-loop hardening
- [[pr_138_remove-specs-plans-tracking]] — a separate, later PR in the same session: `docs/coderails/specs/`/`docs/coderails/plans/` (referenced by [[brainstorming]] and [[writing-plans]], both of which this skill invokes via its own phases) are no longer tracked in the repo
- [[pr_144-149_agentic-loop-hardening-from-loop-engineering]] — PRs #144-149 (2026-07-12) source record: `grade-loop` neutral grading (Phase 2.7c/13), rule 6 "Strongest surface" ([[task-evals]]), the Phase 4b review tier ladder, the `decisions_absorbed` durable trace (Phases 2.5/2.6/5/6/13) + `loop_stop_counts` reset-on-rearm, and the dashboard's loop-decisions tile
- [[task-evals-gate]] — the `grading`/`UNSTAMPED` mechanism this skill's Phase 2.7c/13 now names explicitly
- [[dashboard]] — the RailLeft Directives card that now surfaces this skill's `decisions_absorbed` trace
