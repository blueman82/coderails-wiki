---
title: "Hook: loop_state_guard.sh"
type: hook
created: 2026-06-25
last_updated: 2026-07-08
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_49_gate-function-rename.md
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_87_agentic-loop-path-session-keying.md
  - sources/pr_1-4_task-evals-feature.md
  - sources/pr_11-14_gate-hardening-followups.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_23-24_hook-lib-observability-and-repo-keyed-loop-state.md
  - sources/pr_86-107_2026-07-08_loop-lib-residuals.md
tags: [hook, agentic-loop, progress-json, stop-hook, loop-state, session-keying, task-evals, work-units, tier-0-nogo, unregistered-loop, repo-keyed, malformed-transcript]
---

# loop_state_guard.sh (C1)

Stop hook that enforces `progress.json` presence + ownership when an agentic loop is active, AND (since PR #2 of [[pr_1-4_task-evals-feature]]) blocks loop completion at ≥3 work-units without a passing loop-scope `evals.json`. Part of the two-hook loop-state guard architecture; the C1 (foundation) layer.

Source: `coderails/hooks/scripts/loop_state_guard.sh`
Shared lib (Gates 1–4): `coderails/hooks/scripts/lib/loop_state_common.sh` (extracted PR #49; `als_gate_*` functions shared with [[loop_stall_guard]])
Path helper: `coderails/hooks/scripts/lib/agentic_loop_path.sh`

## Event and mode

| Field | Value |
|---|---|
| Event | `Stop` |
| Mode | block (exit 2) |
| Timeout | 15s (hooks.json) |
| Stop order | 3rd (after confidence + verify, before loop_stall_guard) |

## What it enforces

**Presence + ownership**, plus (since PR #2 of [[pr_1-4_task-evals-feature]]) a **loop-scope eval requirement**. When an agentic loop is active in this session, `progress.json` must exist at the helper-resolved path and be stamped with the current `session_id`. It does NOT police whether the file's content is accurate or current — that is an honest boundary the same as `check_verify_loop.sh` documents ("forces the file to exist and be mine; cannot force content to be accurate"). The eval-gate addition is narrower and more targeted: it only fires exactly when the loop is trying to declare `complete` with ≥3 recorded work-units, and only checks for a *passing* sibling `evals.json` — it does not itself judge quality, only presence-of-a-GO.

## Logic: skip gates (cheap first)

Gates 1–4 are implemented as named `als_gate_*` functions in `loop_state_common.sh` (shared with [[loop_stall_guard]] — extracted PR #49, formerly byte-identical between the two guards). Gates 5–6 are local to this script. The new eval-gate check (`gate_loop_evals_required`, added PR #2 of the task-evals cluster) is inserted **between** gate 4 and gate 5 — see below.

1. **`als_gate_no_transcript`** — allow (nothing to inspect).
2. **`als_gate_stop_hook_active`** — allow (`stop_hook_active == true`; already blocked this turn; avoid stop-loop).
3. **`als_gate_not_a_loop`** — allow if no agentic-loop Skill `tool_use` in transcript. Detection is a structured `jq` match on `name == "Skill"` and `input.skill` matching `(^|:)agentic-loop$`. Text grep for "agentic-loop" is explicitly forbidden (would trip sessions that edit/discuss the skill).
4. **`als_load_progress`** — reads `progress.json` state into globals (status, session, re-armed marker).
4a. **`gate_loop_evals_required`** (new, local, task-evals cluster) — if the loop is otherwise ready to be treated as complete (same three conditions gate 4b below checks) AND `progress.json`'s `work_units` field reports ≥3 units: reads a sibling `evals.json` beside `progress.json`. `GO`/justified `TIER0` → allow (logged). `NO-GO`/`ABSENT` (no file, malformed, or wrong scope) → **BLOCK** (exit 2), pointing at `/coderails:task-evals`. Below 3 units, or `work_units` absent (legacy loop), or `jq` missing: allow, logged with a distinct reason each time (`skipped-below-threshold` / field-absent fail-open / `jq_missing`). **Must run before gate 4b** — see "Gate ordering" below. **Explicit `NO-GO` wins at tier 0 too** ([[pr_11-14_gate-hardening-followups|PR #11]], owner directive, 2026-07-06): the reader (`als_read_loop_evals_result` in `loop_state_common.sh`) now checks `result == "NO-GO"` before the tier-0 exemption branch — previously a tier-0 artifact with an explicit `result: "NO-GO"` still read as the allowed `TIER0` case, because the exemption branch fired on tier alone without inspecting `result` first. "An exemption justifies having no evals, not overriding a recorded failure." A tier-0 artifact that never sets `result` (the legitimate exemption) still reads `TIER0` and is allowed.
4b. **`als_gate_loop_complete` / done-and-not-rearmed check** — allow if `status == "complete"` AND not re-armed AND session-owned (loop is done). Re-armed = `invocation_count > completed_marker`.
5. **File present AND session-owned AND `status != "complete"`** — allow (presence satisfied).
6. **BLOCK (exit 2)** — three failure shapes: absent, session mismatch, stale-complete-after-rearm.

## Gate ordering: why the eval check runs before the shared "loop complete" gate

`als_gate_loop_complete` exits 0 **directly** — not just "returns" — the instant `status == "complete"` AND not re-armed AND session-owned; it has no way to signal "checked, still active" back to a caller placed after it. So `gate_loop_evals_required` re-checks those same three conditions **locally**, rather than restructuring the shared function to support an insertion point after it. This is a reviewed, intentional exact deviation from the guard's usual "compose named gates top-to-bottom" pattern — documented inline in the source with the rationale. (verified: `hooks/scripts/loop_state_guard.sh`; see [[task-evals-gate]] for the full architecture)

## Block messages

- **Absent**: includes the exact path from `agentic_loop_path.sh` and the stub JSON template. The model copies the path; it never computes it. (This is the C1 path-deadlock lesson: a model cannot reproduce a cwd-derived key.)
- **Session mismatch**: shows both the file's session and the current session; tells the model to adopt (re-stamp) or reinitialise.
- **Stale-complete-rearmed**: a new loop started but the prior loop's `complete` is still the recorded status; tells model to reinitialise the stub (carry `completed_marker` forward).

## Path authority design

`agentic_loop_path.sh` is the sole path authority. Both this hook (reader) and the orchestrator (writer, via `Bash` call) resolve the path through it. The model never computes the path — it reads the resolved path from a block message or from a direct `bash "$CLAUDE_PLUGIN_ROOT/hooks/scripts/lib/agentic_loop_path.sh"` call. Path (updated PR #87, slug scheme updated again by PR #24): `$HOME/.claude/agentic-loop/<slug>/<session_id>/progress.json` where `session_id` comes from the Stop-hook payload (this hook) or `$CLAUDE_CODE_SESSION_ID` (the orchestrator's own resolution) — added by PR #87 (see [[pr_87_agentic-loop-path-session-keying]]). `<slug>` is now **repo-keyed, not cwd-keyed**: when cwd is inside a git repo it derives from `git --git-common-dir` (falls back to the old cwd-slug transform on any git failure or non-absolute output — an F3-style cwd fallback), so every worktree of one repo shares the same path — closing a mid-loop `EnterWorktree` orphaning bug. See [[agentic-loop-path-keying]] for the full mechanism, the git<2.31 absolute-path guard, and [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state|PR #24]] for the source record.

## progress.json schema fields (C1 adds)

`schema_version`, `session_id`, `status` (`initialising` | `in-progress` | `complete`), `created`, `last_updated`, `completed_marker`. Plus (task-evals cluster): `work_units` — a JSON object keyed by unit id, each entry carrying at least a `status`; this hook reads `.work_units | length` off it to decide whether the ≥3-unit eval threshold applies. Fail-open when absent (legacy loops predating this field never block on it).

## Log output

Appends a `key=value` line to `$CLAUDE_DISCIPLINE_LOG`:
`hook=loop_state_guard session=<id> invocations=<n> status=<s> owned=<0|1> reason=<r> blocked=<0|1>`

The eval-gate check logs its own line shape: `hook=loop_state_guard session=<id> work_units=<n> evals=<GO|TIER0|NO-GO|ABSENT|skipped-below-threshold|skipped> blocked=<0|1>` — plus a distinct `reason=jq_missing` variant when `jq` is unavailable, kept separate from the "file genuinely absent" case in the audit trail.

## Malformed-transcript tolerance ([[pr_86-107_2026-07-08_loop-lib-residuals|PRs #91, #107]])

The shared lib's `als_count_invocations` — which this hook's `als_gate_not_a_loop`
gate calls — used to abort its **entire** transcript parse on a single malformed
JSONL line (a bare `jq -s` slurp), collapsing the invocation count to 0 and
making this hook treat a genuinely active loop as "not a loop." PR #91 made the
parse per-line tolerant (`jq -R 'fromjson? // empty'` pre-filter, dropped lines
reported as `skipped_malformed=N` on stderr). A same-day post-merge review
(PR #107) found this had introduced two failure-attribution regressions, fixed
forward: a `read_error` tag for an unreadable transcript or a stage-1 `jq`
binary death (previously indistinguishable from "clean"), and an
`all_lines_malformed` tag distinguishing total parse loss (every line
malformed) from a benign partial skip — the distinction gates the
`als_stable_invocations` retry/settle loop, restoring the 5-attempt
flush-race window a mis-attributed "clean" result would have short-circuited.
See the source page for full detail; `unregistered_loop_guard.sh`'s own
separate `jq -s` slurp (`ulg_count_dispatch_turns`) was explicitly NOT touched
by this fix and remains a standing residual.

## Known limitations

- Cannot force the file's content to be accurate or current; a model can write a stub and never enrich it.
- ~~Concurrent same-cwd sessions share a path; the second sees a session mismatch~~ — **fixed by PR #87**: the path is now keyed on cwd + session_id, so concurrent sessions in the same directory no longer share a path or collide. The session-mismatch check is retained but now only fires on a rarer case: a file's content disagreeing with its own session-scoped path (corruption — copied or hand-edited content), not the routine cross-session case it used to catch. See [[pr_87_agentic-loop-path-session-keying]].
- Forgotten within-session teardown leaves a benign `present+owned+in-progress` file (C1 does not block); over-fire risk from stale `in-progress` is C2's concern.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention and the fail-open rationale.

## Relationship to unregistered_loop_guard.sh (new sibling, PR #17)

This hook only ever sees a loop **after** it has registered a `progress.json` — that is its entire enforcement surface (presence + ownership). It has no visibility into a loop that never registered one at all. [[unregistered_loop_guard]] (new sibling Stop hook, F1) closes that blind spot with a heuristic nudge rather than an extension of this hook's own gates — a deliberate choice, since detecting "no registration happened" is a different evidence class (heuristic, not ground truth) from what this hook enforces. See [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] for the incident that motivated it.

## See also

- [[loop_stall_guard]] — C2: requires a `LOOP-STOP` declaration when active+incomplete; shares loop-active detection via `loop_state_common.sh`; untouched by the task-evals gate addition
- [[unregistered_loop_guard]] — new sibling Stop hook (PR #17): nudges when a loop looks unregistered (no `progress.json` this hook could otherwise gate on)
- [[hook-exit-codes]] — this hook blocks via plain `exit 2` on `Stop`, the mechanism this page's table documents
- [[agentic-loop]] — the skill that creates and maintains `progress.json`; Phase -2 is the stub-first contract this hook enforces; Phase 2.7c freezes the loop-scope `evals.json` this hook reads
- [[task-evals]] — the skill that generates the `evals.json` this hook's gate consumes
- [[task-evals-gate]] — design page for the full dual-scope (pr + loop) eval-gate architecture
- [[spec-plan-progress-artifact-chain]] — how `progress.json` relates to `spec.md` / `plan.md`
- [[loop-progress-fields]] — consolidating page for `work_units` (this hook's `.work_units | length` eval-gate threshold read) and `loop_stop_counts`
- [[discipline-loop]] — how the discipline hooks compose (5 Stop hooks as of PR #17)
- [[enforcement-model]] — hooks vs. commands
- [[pr_87_agentic-loop-path-session-keying]] — PR #87: path keyed on cwd+session_id, closing the cross-session race this page previously described as unfixed
- [[pr_1-4_task-evals-feature]] — PR #2 source record: the loop-scope eval gate added to this hook
- [[pr_11-14_gate-hardening-followups]] — PR #11 source record: explicit NO-GO now wins over the tier-0 exemption
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #17 source record: the sibling unregistered-loop nudge hook this blind spot motivated
- [[agentic-loop-path-keying]] — design page for the repo-keyed slug scheme (PR #24)
- [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state]] — PR #24 source record: repo-keyed slug closing the worktree-hop orphaning bug; PR #23 (same cluster): hook-lib failure-observability rework, unrelated to this hook directly but shares the cluster
- [[pr_86-107_2026-07-08_loop-lib-residuals]] — PRs #91/#107 source record: malformed-transcript tolerance in the shared lib's invocation-count parse, then a same-day failure-attribution correction (`read_error`, `all_lines_malformed`)
