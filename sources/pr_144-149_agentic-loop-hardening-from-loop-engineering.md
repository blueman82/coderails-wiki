---
title: "PRs #144-149 — agentic-loop hardening from loop-engineering diagnosis"
type: source
origin: PR #144, #145, #146, #147, #148, #149 (merged 2026-07-12)
created: 2026-07-12
last_updated: 2026-07-12
sources: []
tags: [source, agentic-loop, task-evals, enforce_pr_workflow, loop_state_guard, dashboard, decisions-absorbed, grade-loop, merge-gate]
---

# PRs #144-149 — agentic-loop hardening from loop-engineering diagnosis

Five independent PRs closing gaps a "loop engineering" diagnosis surfaced against the agentic-loop skill: orchestrator-graded loop evals could pass the gate with no independent stamp (#144), the anti-gaming rule set didn't force a user-facing eval when the goal state was user-facing (#145), a hand-rolled `scripts/merge.sh` invocation bypassed the review-pr merge gate entirely (#146), the Phase 13 terminal self-audit's "decisions absorbed" list was reconstructed from conversation memory rather than a durable trace (#147), and the dashboard had no surface for that trace at all (#148, with a same-day follow-up #149 tightening the display reader to match the bash SSOT). Shipped as a 5-PR loop, all merged to `origin/main` (tip `8eac114`).

## The five PRs

| PR | Change | Merge commit | Files touched |
|---|---|---|---|
| #144 | `grade-loop` neutral grading subcommand + UNSTAMPED demotion | 038dfec | `hooks/scripts/lib/loop_state_common.sh`, `hooks/scripts/loop_state_guard.sh`, `scripts/lib/eval-artifact.sh`, `scripts/post_evals.sh`, `skills/task-evals/SKILL.md`, `skills/agentic-loop/SKILL.md` |
| #145 | Rule 6 "Strongest surface" | f86f558 | `skills/task-evals/SKILL.md` |
| #146 | `merge.sh` recognized as the merge-gate subcommand; decoy-number + `--dry-run` bypass fixes | 8eac114 | `hooks/scripts/enforce_pr_workflow.sh`, `skills/agentic-loop/SKILL.md` |
| #147 | `decisions_absorbed` trace in `progress.json`; `loop_stop_counts` reset-on-rearm | f2ed234→(chain, see below) | `skills/agentic-loop/SKILL.md` |
| #148 + #149 | Dashboard loop-decisions tile; `readEvalsFrozen` aligned with bash SSOT | 27c0c0f, 05ee6cc | `skills/dashboard/app/src/lib/collect/sessions.ts`, `skills/dashboard/app/src/components/RailLeft.tsx`, associated test files |

## #144 — `grade-loop`: closing the self-graded-loop-evals gap

Before this PR, a loop-scope `evals.json` could be hand-graded `GO` by the orchestrator itself and the `loop_state_guard` Stop hook would accept it — the loop-scope gate checked structural shape and the `result` field, but nothing verified *who* computed `result`. `scripts/post_evals.sh` gained a `grade-loop` subcommand: it validates the loop-variant structure, computes `result` via the existing `eval_artifact::compute_go` SSOT (the same pure `jq` P0-pass predicate the pr-scope gate already used), and atomically stamps `result`, `graded_at`, and a new `grading` object (`{by: "post_evals.sh grade-loop", checksum: "<sha256 over statuses+result>"}`).

`als_read_loop_evals_result` (the loop-scope reader in `loop_state_common.sh`) now demotes an otherwise-valid `GO`/`TIER0` verdict to a new terminal category, **`UNSTAMPED`**, when the `.grading` object is absent or its checksum doesn't match a fresh recomputation against the file's own `.result`. `loop_state_guard.sh` blocks on `UNSTAMPED` with a message pointing at `grade-loop`. The stamp is explicitly scoped as **provenance, not proof against forgery** — it catches accidental drift (a status hand-edited after grading without re-stamping), not a deliberately fabricated checksum; the source comments state this ceiling directly rather than overclaiming.

A side effect worth naming precisely: because `grade-loop` always writes `.result` (even a tier-0 exemption with an empty `.evals` array computes `GO` — vacuously true under `compute_go`'s "every P0 passes" predicate with zero P0 evals), a **stamped** tier-0 artifact now reads as `GO`, not `TIER0` — the reader's `elif` chain checks `.result` before `.tier`, so `TIER0` as a terminal outcome now only occurs for an **unstamped** tier-0 file. The test suite (`loop_state_guard_evals.test.sh`) was rewritten to stamp fixtures via the real `post_evals::grade_loop` function rather than hand-writing `result`/`grading` fields, so "properly stamped" test fixtures are byte-identical to what the writer produces.

`skills/task-evals/SKILL.md`'s schema example gained the `grading` field (documented as optional/additive — does not bump `schema_version` past 1) and the Verifier-agent-contract section was rewritten: the orchestrator folds verifier statuses into `evals.json` but never computes or writes `result` itself at either scope — `post_evals.sh` (pr scope) or `post_evals.sh grade-loop` (loop scope) is the sole neutral assembly step. `skills/agentic-loop/SKILL.md` Phase 2.7c and Phase 13 were updated to name `grade-loop` as the grading mechanism.

## #145 — Rule 6, "Strongest surface"

The task-evals anti-gaming rule set grew from five rules to six. New **rule 6**: if the task's goal state names something a human sees or interacts with — a UI, CLI output, a rendered artifact, a served endpoint — at least one P0 eval must exercise that surface directly (browser, CLI invocation, HTTP request), never only code-greps of merged state. At pr scope pre-merge this means the locally-run artifact; at loop scope, the deployed surface. This is a **writer-side generation rule enforced by generation discipline and review, not a script** — no automated check can detect "this goal state is user-facing," so unlike rules 2-5 (which the pr/loop gates can partially verify structurally) rule 6 has no corresponding gate change.

The rule's exemplar is named inline in the skill: the run-output noise-strip loop (PRs #139-141, per `dashboard ASK-button UX loop` project history) where merged-state greps passed while the live streaming window still leaked the injected skill-preamble text — only an in-browser eval driving the actual streaming lifecycle caught it. The section title changed from "the five anti-gaming rules" to "the six anti-gaming rules" throughout `skills/task-evals/SKILL.md`.

## #146 — `merge.sh` recognized as a gated merge subcommand

`enforce_pr_workflow.sh`'s `gate_in_scope` function classified commands by matching against a fixed set of prefixes (`gh pr create`, `gh pr merge`, `git merge`, `git push`) — `scripts/merge.sh <N>` (the repo's own sanctioned merge wrapper, which calls `gh pr merge` internally) matched none of them, so a hand-rolled `scripts/merge.sh <N>` invocation sailed past the review-pr and eval-artifact gates entirely. This is a real, previously-ungated bypass, not a regression — `merge.sh` was never covered by this hook before this PR.

**Fix, landed across two commits in the same PR:**

1. **The matcher** — a new `elif` arm recognizes `merge.sh` (optionally path-prefixed, optionally `bash`/`sh`-prefixed, optionally quoted) as `subcommand="merge"`, reusing the identical review-pr + eval-artifact gating `gh pr merge` already gets. Word-boundary anchored (mirrors the `git merge-base` fix, PR #42) — a name merely *containing* `merge.sh` (`auto_merge.sh`, `some-merge.shim`) must not match.
2. **Decoy-number hijack, found in review and fixed before merge** — the original `pr_num` extraction for both the `gh pr merge` and `merge.sh` forms scanned the raw `$cmd` string end-to-end, so a PR number merely *mentioned* in an earlier, non-executed segment of a shell chain (e.g. `echo "run merge.sh 999 first" && scripts/merge.sh 140`) could donate its number to the later, real invocation — letting an unreviewed PR (140) merge under a reviewed decoy's number (999). Fixed by capturing the matched segment (`matched_seg`, same idiom as the existing `push_target_dir` capture) in `gate_in_scope` and scoping both `pr_num` extraction blocks to it instead of the raw command.
3. **`--dry-run` passthrough bypass, also found in review** — `gate_safe_passthrough`'s global `--dry-run`/`--help` exemption assumed those flags are always inert (true for `gh pr merge --dry-run`, which `gh` itself rejects, and `gh pr create --help`, which just prints usage). But `scripts/merge.sh`'s arg parser (`merge::main`) reads only `$1` as the PR number and silently ignores every trailing token — so `scripts/merge.sh 140 --dry-run` would have stood aside from the gate entirely and then performed a **real** merge. Fixed by excluding `merge.sh` invocations from that passthrough.

**Documented, tested, non-regression residuals** (the hook's pre-existing "not every shell form is parsed" posture, now extended to `merge.sh`): `bash -x scripts/merge.sh 140` and `command bash scripts/merge.sh 140` are not gated — these are the same class of unmatched invocation forms the hook already didn't parse for `gh pr merge` before this PR.

`skills/agentic-loop/SKILL.md`'s Phase 4b gained a **review tier ladder**: regardless of a PR's own eval-artifact tier, `/pr-review-toolkit:review-pr` + `/coderails:post-review` always run; only at tier 0 may the separate `/security-review` pass be skipped, and only after checking the actual diff file list — any path under `hooks/`/`scripts/`, or any auth/exec/network-fetch touch, forces the security pass regardless of the declared tier. The override keys off the diff, never the self-assigned tier label, for the same reason the task-evals tier rules resist self-exemption: reusing a self-assigned label to gate a security control would be the identical loophole shape one level up. A concrete example is cited inline: the `silent-failure-hunter` toolkit reviewer caught a real swallowed-crash-payload bug on the one-function PR #142 fix, which is why the toolkit review itself is never skippable — only the separate native `/security-review` pass is tier-0-skippable.

## #147 — `decisions_absorbed` trace

Phase 13's "decisions absorbed" self-audit bullet (added by the earlier PR #86 hardening) was, before this PR, reconstructed from conversation memory at teardown time — exactly the kind of after-the-fact self-report the section otherwise exists to avoid. This PR gives it a durable, append-only trace: `progress.json` gains a `decisions_absorbed` array of `{phase, decision}` objects, appended chronologically (oldest-first) at the phase boundary where each in-scope autonomous decision is made:

- **Phase 2.5** — a design-fork auto-adopted under full-autonomous envelopes: `{phase: "2.5", decision: "<chosen shape + flip-condition>"}`.
- **Phase 2.6** — a disposition (`clean-break`/`preserve-compat`) defaulted without asking: `{phase: "2.6", decision: "<disposition, with named_blocker if applicable>"}`.
- **Phase 5** — a consciously-absorbed `/coderails:disconfirm` skip (the diagnosis was already verified, e.g. during brainstorming, so there's no fresh diagnosis left to disconfirm).
- **Phase 6** — a notable in-scope action taken without a check-in.

Phase 13's own "Decisions absorbed" report bullet is now specified as **copied verbatim** from this array, chronological, never reconstructed from memory. The `retro.json` teardown artifact (assembled at Phase 13 before the `complete` declaration) carries the same array verbatim into its own `decisions_absorbed` field, by the same rule.

**Companion fix, same PR: `loop_stop_counts` carry-forward made conditional.** The pre-existing rule ("on any wholesale `progress.json` rewrite, re-read the existing file first and carry `loop_stop_counts` forward verbatim") is now conditional on the prior file's `status`: **verbatim** on a genuine mid-loop recovery rewrite, but **reset to `{}`** when the prior file's `status` was `"complete"` — i.e. on a fresh re-arm after a loop has already finished. Before this fix, a brand-new loop starting in the same repo/session-key slot could inherit a completed prior loop's stop-counts, which is a wrong carry-forward (those counts describe a different, finished loop, not the one starting now).

## #148 + #149 — dashboard loop-decisions tile

The dashboard's `collectLoops` (in `skills/dashboard/app/src/lib/collect/sessions.ts`) gained a `decisions: string[]` field on `LoopInfo`, populated by a new `readDecisions()` helper: reads `progress.json`'s `decisions_absorbed` array, filters to well-formed `{phase, decision}` string-pair entries (degrade-don't-throw on a non-array or malformed entries, same stance as the pre-existing `readUnitTitles` helper), formats each as `"<phase>: <decision>"`, and returns the **last 5, newest-first**. `RailLeft.tsx` renders these as `.hud-decision-item` entries under the existing Directives card (the same card that already shows the active loop's `progress.json` work-unit checklist) — one line per decision, duplicates rendered as distinct entries (not collapsed), an empty array rendering no sub-list at all rather than an empty placeholder.

**`readEvalsFrozen` gained the `grade-loop` stamp check** (mirroring the bash `als_read_loop_evals_result` reader #144 added), plus an explicit NO-GO-wins-over-tier-0 check matching the bash SSOT's precedence: a `GO`/`TIER0` verdict now additionally requires `data.grading` to be present with both `.by` and `.checksum` non-empty before reading as frozen; an explicit `result: "NO-GO"` always wins over a tier-0 exemption regardless of grading presence. This is **presence-only** — the TS reader does not recompute the checksum the way the bash hook does (an explicit KISS trade-off for a display surface, documented inline: "a status edited after grading without re-stamping will still show frozen here even though the hook would demote it to UNSTAMPED").

**#149 is a same-day follow-up tightening this exact check**, found after #148 merged: the first cut's stamp check only required `.grading.checksum` to be a non-empty string, not `.grading.by` — so a partial stamp (checksum present, `by` absent or both empty strings) would still read as frozen on the dashboard even though the bash hook's `[ -z "$stamped_by" ] || [ -z "$stamped_checksum" ]` check would correctly demote it to `UNSTAMPED`. #149 fixes the TS reader to require both fields non-empty (after trim), closing the drift between the two readers.

## Why the drift between #148 and #149 matters: a third seam on the same schema

`sessions.ts`'s `readEvalsFrozen` is a **third, independent consumer** of the loop-scope `evals.json` schema, alongside the two already documented in [[task-evals-gate]]'s "one schema, two seams": `post_evals.sh grade-loop` (the writer) and `loop_state_guard.sh`'s `als_read_loop_evals_result` (the enforcement reader). The dashboard reader is read-only and non-enforcing — it can't block anything — but it can still **display a wrong verdict** if its logic drifts from the bash SSOT, which is exactly what happened between #148 and #149: the first cut's stamp check was weaker than the bash reader's, so a partially-stamped artifact could show "frozen" on the dashboard while the hook would actually block on it as `UNSTAMPED`. The dashboard code comments explicitly acknowledge this as an accepted asymmetry (checksum-recomputation is skipped for KISS), but the #149 fix shows that even the *cheaper* presence-only check still needs to track the bash reader's exact boolean shape (both fields required, not just one) to avoid misleading the human reading the tile. See [[task-evals-gate]]'s "one schema, two seams" section, updated by this cluster to name three seams instead of two.

## See also

- [[task-evals-gate]] — the dual-scope enforcement design this cluster extends (`grading` field, `UNSTAMPED` category, rule 6, three-seams update)
- [[task-evals]] — the skill whose anti-gaming rule set gained rule 6
- [[loop_state_guard]] — the hook whose reader gained the `UNSTAMPED` demotion
- [[enforce_pr_workflow]] — the hook whose `gate_in_scope`/`enforce_required_step` gained the `merge.sh` matcher, decoy-number scoping, and `--dry-run` exclusion
- [[agentic-loop]] — Phase 2.7c/13 (grade-loop), Phase 4b (review tier ladder), Phase 2.5/2.6/5/6/13 (`decisions_absorbed`)
- [[loop-progress-fields]] — consolidating page for `progress.json`'s tracked fields; extended by this cluster with `decisions_absorbed` as a third field alongside `work_units`/`loop_stop_counts`, and the `loop_stop_counts` reset-on-rearm rule
- [[dashboard]] — the skill whose RailLeft Directives card gained the loop-decisions tile
- [[project_agentic_loop_hardening_from_loop_engineering]] — the diagnosis memory that scoped this cluster's four target weaknesses (oracle independence, external/binary exits, proportionality, comprehension debt)
