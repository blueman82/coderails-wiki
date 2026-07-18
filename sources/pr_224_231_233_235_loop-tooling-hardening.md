---
title: "PR #224, #231, #233, #235 — loop tooling hardening cluster"
type: source
created: 2026-07-17
last_updated: 2026-07-18
sources:
  - hooks/scripts/lib/loop_state_common.sh
  - hooks/scripts/loop_stall_guard.sh
  - hooks/scripts/tests/loop_stall_guard.test.sh
  - hooks/scripts/tests/run_all.sh
  - hooks/scripts/tests/run_all_skip.test.sh
  - hooks/scripts/tests/docs_sync_routine.test.sh
  - hooks/scripts/destructive_bash_gate.sh
  - hooks/scripts/tests/destructive_bash_gate.test.sh
  - skills/agentic-loop/finishing-out.md
  - skills/finishing-a-development-branch/SKILL.md
tags: [hook, agentic-loop, proof-gate, test-runner, destructive-bash-gate, worktree, pattern-id, skip-vs-fail, systemMessage, withdrawn-proofs]
---

# PR #224, #231, #233, #235 — loop tooling hardening cluster

Four independently merged PRs from one agentic loop, same theme: **give a
blocked, skipped, or deferred action a way to say what it is and why**,
mirroring the design line PR #216/#217 already established (see
[[pr_216_217_safe-routes-and-cost-miner-diagnostics]]). All four merged to
`origin/main` 2026-07-17, in this order:

| PR | Merge SHA | Merged at | Theme |
|---|---|---|---|
| #224 | `720ffbb39359dc046ba85f79b3bb5348eb6fbf34` | 19:42:37Z | proof withdrawal + merged systemMessage |
| #231 | `f43d37cf707b39f36e51a1457c7f859e10331ebc` | 20:15:59Z | `run_all.sh` SKIP class (exit 3) |
| #233 | `6437e83722cd624066f87c1c8305ad58f200bdf6` | 20:36:34Z | mention-safe `patternId` per deny() arm |
| #235 | `45fbb3485e968243cb42826fd2df692f22880f80` | 20:56:09Z | lock-aware worktree cleanup |

(verified: `gh pr view <n> --json mergeCommit,mergedAt`, all four cross-checked
against `git log origin/main` — SHAs and diffstats match exactly.)

A fifth item was in scope conceptually but did NOT land from this loop: **PR
#221** (`feat/discriminating-check-gate`, a freeze-time gate mechanically
re-running each scripted eval/proof command plus its negative control) was
**CLOSED**, not merged — it duplicated **PR #218**
(`worktree-discriminating-check-gate`), which had already merged earlier the
same day at `81c8857fe7712bb826ff8633960d763cf3e83712`, 2026-07-17T15:46:07Z,
well before this cluster started. #218 is the canonical implementation; #221
is not documented as landed anywhere in this ingest.

## PR #224 — proof withdrawal + merged systemMessage

Extends [[loop_stall_guard]]'s proof gate
(`als_gate_proofs_on_complete`, added PR #198 — see
[[pr_194_198_loop-complete-deferral-and-proof-gates]]) with a
**`withdrawn_proofs` array**: a way to say "I ran this proof, it failed, and
I'm not fixing it — here's why" instead of the only two prior options
(satisfy it, or never declare `complete`).

### Withdrawal validation — stricter than `.proofs`, not looser

A `withdrawn_proofs` entry must pass ALL of:

- **Actually executed** in this session's transcript (same `$exec_index`
  command-match the `.proofs` pass already builds — no second scan, closing
  the same O(proofs × executions) DoS-to-bypass class PR #198 fixed for
  `.proofs`).
- **Its last execution's `is_error` is strictly `true`.** This is the one
  place withdrawal is *stricter* than `.proofs`: `.proofs` treats
  `is_error: null` as "ran, no clear failure signal, let it through";
  `withdrawn_proofs` does not — a withdrawal is a claim that a failure was
  *witnessed*, so `null`/`false` both fail the check (`not_failed` verdict).
  Tolerance here would let a passing or ambiguous check be withdrawn instead
  of fixed.
- **Non-empty `withdrawn_reason` and `cmd`.**
- **No double-dip:** the same `id` must not also appear in `.proofs` — a
  proof can't be simultaneously pending and withdrawn.
- **Combined cap:** `len(.proofs) + len(.withdrawn_proofs) <= 100`, not 100
  each — both arrays feed the same mining pass, so 100-and-100 would
  recreate the ~100×100 timeout shape the original cap (PR #198) was sized
  to rule out.

**Edge case closed:** the old gate's `.proofs` absent/null path was a bare
`return 0` ("nothing to prove"). That's wrong once `withdrawn_proofs` exists
— a loop whose *only* proof was withdrawn produces exactly `.proofs` absent
+ `withdrawn_proofs` populated, and the old early-return would let an
unvalidated withdrawal sail through. Fixed: the true "nothing to prove" allow
now requires **both** arrays empty; either one alone still runs the full
mining pass.

Fail-closed posture is preserved throughout — a withdrawal that doesn't meet
every check above blocks `exit 2`, same as an unproven `.proofs` entry always
has. Only the **final human-visible message** (naming which ids were
successfully withdrawn) inherits a never-block posture, and only because it
runs after every validation has already passed — a reporting failure must
never re-block an already-passed gate.

### `ALS_PENDING_SYSMSG` — one JSON document, not two colliding ones

Before this PR, `als_gate_proofs_on_complete` (for withdrawal notices, once
added) and `als_report_cost_on_complete` (PR #204 — see
[[pr_204_cost-reporter]]) would each need to emit their own top-level
`{systemMessage: ...}` JSON on the same hook invocation's stdout. Two
concatenated top-level JSON objects are not valid as a single document under
a whole-buffer parse — the second would corrupt or shadow the first.

Fixed with a shared global accumulator in `loop_state_common.sh`,
`ALS_PENDING_SYSMSG`, and a helper `als_append_pending_sysmsg` that both
functions call (newline-joined if the accumulator is already non-empty)
instead of emitting JSON directly. `loop_stall_guard.sh`'s call site emits
the **single** merged `jq -n --arg m "$ALS_PENDING_SYSMSG" '{systemMessage:
$m}'` only after both gates have had their chance to append — the only place
either message reaches the human terminal.

### Q2 review fix folded into the same PR

A same-PR review pass (commit `4ee8907`) corrected a stale doc comment (the
cost reporter's header still claimed it emits `systemMessage` directly, after
the rewiring above changed it to append) and removed two dead
`proofs_absent`/`withdrawn_absent` flags — `jq length` already returns `0`
for an absent/null/empty array, so the flags gated nothing the length read
didn't already handle; the type-guard case blocks that fail closed on a
non-array shape were kept.

## PR #231 — `run_all.sh` treats exit 3 as SKIP, not FAIL

`hooks/scripts/tests/run_all.sh` (the test-suite aggregator that globs
`*.test.sh` in its own directory and runs each) previously had exactly two
outcomes per suite: pass (exit 0) or FAIL (any non-zero exit). This
collapsed "the suite ran and found a real problem" and "the suite couldn't
run because a precondition is missing" into the same signal.

**New third class: exit 3 = SKIPPED**, absorbed into an overall exit 0
alongside real passes — a run with only passes and skips is still green.
Real failures (any non-zero exit that isn't exactly 3) still fail the run
exactly as before.

**All-skipped floor:** if `skips == total`, the run exits non-zero with a
`WARNING: all N suites skipped — nothing was actually verified` — a run
where every suite skipped is not a legitimate green; it verified nothing.

First consumer: `docs_sync_routine.test.sh`'s node_modules precondition
check. A missing `skills/dashboard/lib/node_modules` or
`skills/dashboard/runner/node_modules` is a normal cold-clone state
(documented in `docs/routines.md`, "Prerequisites for a cold clone"), not a
broken assertion — it now `exit 3`s with a `SKIP:` message instead of
`exit 1`/`FAIL`. Scoped narrowly: the earlier `command -v node` check in the
same file stays a hard FAIL — a missing `node` binary isn't the documented
cold-clone case this exists to unblock.

**Downstream-consumer caveat, noted explicitly in the source comment:** any
eval/proof formula reading `run_all.sh`'s output should gate on `fails`, not
`passed == total` — a legitimate skip makes `passed < total` without being a
failure.

### Test coverage — four negative controls pin the exact boundary

`run_all_skip.test.sh` (new file) copies the real `run_all.sh` into a scratch
dir alongside synthetic `exit N` fake suites and drives it for real (not a
reimplementation). Six cases: pass+fail+skip mixed (fail wins, skip named
separately); pass+skip only (green, skip still reported); all-skip (the
floor — non-zero + WARNING); and three negative controls — exit 1, exit 2,
and exit 4 must each still report FAILED, never SKIPPED, pinning the exact
`== 3` boundary from both sides (ruling out a mutant `-ge 3` or `-ne 0` skip
check).

**Follow-up:** `run_all_skip.test.sh` — this PR's own new test file — was at
risk of silently dropping out of the PR under `push.sh`'s then-current
staging (`git add -u` stages tracked-modified only; a genuinely new file
still needed a separate explicit `git add`). Caught by the worker, not an
automated gate. Fixed by [[pr_239_push-sh-add-flag|PR #239]]'s opt-in `--add <path>` flag.

## PR #233 — mention-safe `patternId` per `destructive_bash_gate.sh` deny() arm

Extends every named-route case arm added by PR #216 (see
[[pr_216_217_safe-routes-and-cost-miner-diagnostics]]) with a hyphenated
`pattern_id` (e.g. `git-reset-hard`, `chmod-r-777`), threaded through to a new
`patternId` field in the hook's JSON output
(`hookSpecificOutput.patternId`, `null` for the unmapped generic fallback
arm). **Message-only** — the matcher regexes and DENY/ALLOW verdict set are
byte-identical before and after; only the output shape gained a field.

**Why "mention-safe" matters:** a test or artifact that needs to reference
"the `chmod -R 777` block" by name previously had to either quote the
literal blocked string (risking the destructive-bash gate itself denying the
line that documents the block — a known standing tension, see
[[pr_216_217_safe-routes-and-cost-miner-diagnostics]]'s "Lessons" #4) or use
prose that could drift from the actual matcher. The id is deliberately
hyphenated so it can never itself match the matcher's own whitespace-based
regexes — `"chmod-r-777"` does not match
`chmod[[:space:]]+-R[[:space:]]+777`. Every id/mention pair is tested both
ways: the id string alone must ALLOW, and the real blocked literal must DENY
and carry that same `patternId`.

**Source-drift tripwire extended:** the existing "Deliverable B" tripwire
(PR #216 — compares the gate's blockable pattern set against committed
`EXPECTED_*` snapshots) gained a second check —
`assert_every_route_arm_has_pattern_id` walks the `case "$pat_lc" in ... esac`
block via `awk`, finds every non-fallback `route=` assignment, and fails if
the immediately-following line isn't a `pattern_id=` assignment. A future
route arm added without an id is caught here, not just a future *pattern*
added without a route.

## PR #235 — lock-aware worktree cleanup

Extends `finishing-a-development-branch`'s Step 6 (merged-worktree removal,
invoked per-work-unit at [[agentic-loop]] Phase 4b — see
[[pr_162_agentic-loop-finishing-out]]) with a lock check before removal.
Previously Step 6 went straight to `git worktree remove` for any
coderails-owned worktree (path under `.worktrees/`/`worktrees/`); a locked
worktree would simply fail that command with no interpretation of *why*.

**New three-way branch**, parsed from `git worktree list --porcelain`'s
`locked <reason>` line via `awk`:

1. **Unlocked** (`LOCK_REASON` empty) — remove normally, unchanged from
   before.
2. **Locked, live pid** — a pid is parsed out of the lock reason (harness
   lock reasons look like `claude session <name> (pid NNNNN start <date>)`,
   extracted with `grep -oE '\(pid [0-9]+ '`), then `kill -0 <pid>` checked.
   A live pid means another session is still using the worktree — **report
   and defer, never force**.
3. **Locked, dead pid or unparseable reason** — no pid found: report and
   leave in place (can't distinguish live/dead, so treated the same as
   "live" for safety). A confirmed-dead pid (`kill -0` fails): clear the
   lock (`git worktree unlock`) and remove.

**Only a confirmed-dead pid ever triggers a force-clear.** No-pid-parseable
and confirmed-live both mean "leave it alone" — the asymmetry is
deliberate: a merged PR does not by itself mean the worktree is safe to
remove, since some other session may still be actively working in it at the
exact moment loop teardown runs.

`skills/agentic-loop/finishing-out.md` gained a matching caveat paragraph
alongside its existing "never remove the worktree that is the shell's own
cwd" caveat. `SKILL.md`'s Common Mistakes and Never/Always checklists both
gained a new line naming this failure mode explicitly (force-removing a
locked worktree without checking the pid → yanks a live session).

## Lessons (repo-agnostic, worth carrying beyond this cluster)

1. **A blocked/skipped/withdrawn/deferred action needs a name for itself,**
   not just a binary pass/fail — this is the fourth PR in nine days
   (#203/#216 named routes, #217 named stderr causes, this cluster names
   proof withdrawals, skip reasons, pattern ids, and lock states) to replace
   an undifferentiated signal with a discriminated one.
2. **A validation gate's "nothing to check" early-return needs re-auditing
   every time a new optional field is added next to the one it originally
   guarded.** PR #224's edge case (an empty `.proofs` alongside a populated
   `withdrawn_proofs`) is the same shape as several prior loop_stall_guard
   incidents: an early-return written for one field silently swallows a
   sibling field added later, unless the return condition is explicitly
   widened to check both.
3. **A new "meh" outcome class (SKIP) needs the same negative-control
   discipline as a new blocking class.** PR #231 pinned the exact `== 3`
   boundary with exit-2 and exit-4 negative controls, the same rigor
   [[pr_216_217_safe-routes-and-cost-miner-diagnostics]] applied to route
   needles — an off-by-one skip check (`-ge 3`, `-ne 0`) would silently
   absorb real failures as skips.

## Files changed

**PR #224:** `hooks/scripts/lib/loop_state_common.sh`,
`hooks/scripts/loop_stall_guard.sh`,
`hooks/scripts/tests/loop_stall_guard.test.sh`

**PR #231:** `hooks/scripts/tests/run_all.sh`,
`hooks/scripts/tests/run_all_skip.test.sh` (new),
`hooks/scripts/tests/docs_sync_routine.test.sh`

**PR #233:** `hooks/scripts/destructive_bash_gate.sh`,
`hooks/scripts/tests/destructive_bash_gate.test.sh`

**PR #235:** `skills/agentic-loop/finishing-out.md`,
`skills/finishing-a-development-branch/SKILL.md`

## Wiki pages updated

- [[loop_stall_guard]] — new "Proof withdrawal" subsection under the
  existing Proof gate section; `ALS_PENDING_SYSMSG` noted at the
  `als_report_cost_on_complete` call-site description
- [[destructive_bash_gate]] — route table section gains a `patternId`
  column note; tripwire section notes the id-presence check
- [[finishing-a-development-branch]] — Step 6 section gains the lock-check
  branch
- `run_all.sh` documented here (source page) only — no dedicated wiki page
  created; the aggregator is referenced from several existing pages
  ([[loop_stall_guard]]'s sibling `offload_push_guard.md`,
  `design/agentic-loop-path-keying.md`) but doesn't yet carry enough
  standalone concept weight to justify one, per curator minimalism

## See also

- [[pr_194_198_loop-complete-deferral-and-proof-gates]] — PR #198, the
  original proof gate this cluster's withdrawal mechanism extends
- [[pr_204_cost-reporter]] — PR #204, the other `ALS_PENDING_SYSMSG`
  consumer
- [[pr_216_217_safe-routes-and-cost-miner-diagnostics]] — the immediately
  prior cluster establishing the "name the cause" design line this cluster
  continues (named routes, named stderr causes → named withdrawal reasons,
  named skip reasons, named pattern ids)
- [[pr_162_agentic-loop-finishing-out]] — the original per-work-unit
  worktree cleanup this cluster's lock-awareness extends
- [[agentic-loop]] — Phase 4b, where finishing-a-development-branch's Step 6
  runs
