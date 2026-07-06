---
title: "PRs 23-24 — Follow-ups loop 2: hook-lib failure observability + repo-keyed loop state"
type: source
created: 2026-07-06
last_updated: 2026-07-06
sources: []
tags: [hook, agentic-loop, loop-state, session-keying, observability, worktree, followups]
---

# PRs 23–24 — Follow-ups loop 2: hook-lib failure observability + repo-keyed loop state

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR #23 | `followups/pr-a-hook-lib-robustness`, merged 2026-07-06, merge SHA `9837c15` |
| PR #24 | `followups/pr-b-loop-state-keying`, merged 2026-07-06, merge SHA `14c7f1c` |
| Session | 9509a663 (loop 2 of this follow-ups cluster) |

Both PRs are treated as one wiki cluster: they landed from the same loop, wu1
and wu2 of a 3-unit follow-ups loop (wu3 was three editorial consolidating
pages, delivered separately — see [[loop-progress-fields]],
[[trust-floor]], [[sync-docs]]).

## PR #23 — hook-lib: distinguish jq failure from genuine zero, fix als_log stderr leak

**Problem** (failure-hunter finding, reproduced empirically): `als_count_invocations`
used to log a `jq`-failure reason directly, on every call, via `als_log`. But
`als_stable_invocations` retries this function up to `MAX_ATTEMPTS` times to
ride out the transcript-flush race — so a transient failure that recovered on
a later attempt was indistinguishable in the log from one that never
recovered, and a sustained failure logged once **per attempt** with no final
verdict. Reproduced: a persistently-malformed transcript produced 2 identical
log lines at `MAX_ATTEMPTS=3`, with no way to tell "gave up" from "2
coincidental failures."

**Fix**: `als_count_invocations` no longer logs directly. On a `jq` failure it
now signals a reason tag (`jq_missing` / `jq_parse_error`) on **stderr**
instead of calling `als_log`. A global can't carry this signal because the
sole retrying caller invokes the function via command substitution
(`n=$(als_count_invocations ...)`), which runs in a subshell — any global the
function set would vanish with the subshell and never reach the caller.
Stderr survives that boundary.

`als_stable_invocations` captures both streams per attempt — stdout via
normal capture, stderr via a scratch file (one call per attempt, so the
transcript is never read twice per attempt, preserving the original
flush-race-tolerance property) — and logs **exactly one** summary line after
the retry loop exits: `reason=<tag> attempts=<N> outcome=recovered|exhausted`.
Zero lines when every attempt was clean. The stdout contract (empty or an
integer, fail-open) is unchanged for every consumer.

The scratch file is created with `mktemp` and used fail-open: if `mktemp`
itself fails, the loop falls back to discarding stderr entirely rather than
erroring out (mirrors the [[pr_21-22_loop2-suggestion-tier-followups|PR #21]]
tempfile lesson — a scratch-file failure must not become a new failure mode).

**One-shot caller carve-out**: `unregistered_loop_guard.sh`'s
`ulg_has_skill_invocation` calls `als_count_invocations` directly, not through
the retry wrapper. It now explicitly discards stderr
(`2>/dev/null`) rather than leaving the new stderr signal to leak into the
hook's own stderr — this preserves its prior (pre-hardening) behaviour of
being silent on a parse failure, unchanged in outcome, just now an explicit
choice instead of an accident of the old design.

**`als_log` fix**: also rewritten to fix a real stderr leak. The prior
implementation had a trailing `2>/dev/null` on the `printf` alone, which does
**not** suppress a redirection-open error (e.g. `LOG_FILE`'s parent directory
missing) — only errors from `printf` itself. Fixed via a brace-group wrapping
the whole `printf`+redirect, so the group's own `2>/dev/null` also catches the
redirection-open error. This was proven empirically, not assumed. **Accepted
tradeoff, documented inline**: if `LOG_FILE`'s parent directory is missing, a
caller's own failure-reporting line (e.g. `als_stable_invocations`' jq-failure
summary) is itself silently swallowed by this same brace-group — only
reachable via a misconfigured `CLAUDE_DISCIPLINE_LOG` override, not a normal
operating condition.

**Verification**: gate outcomes proven bit-identical to the pre-change build
via a cross-SHA consumer exit-code matrix (3 consumers × 4 fixtures) —
confirms the logging-only rework didn't perturb any hook's stdout contract or
block/allow decision.

**Files**: `hooks/scripts/lib/loop_state_common.sh`,
`hooks/scripts/unregistered_loop_guard.sh`,
`hooks/scripts/tests/unregistered_loop_guard.test.sh`.

## PR #24 — agentic_loop_path: validate git-common-dir output is absolute before use; repo-keyed loop state

**Problem** (the incident this PR fixes): `agentic_loop_path.sh`'s slug was
derived purely from `$PWD`, so a mid-loop `EnterWorktree` hop — which changes
cwd to a different worktree of the *same* repo — silently orphaned the
loop's `progress.json` onto a new slug. This is exactly the shared-checkout
workflow the 2026-07-06 user directive prescribes (mid-session `EnterWorktree`
is now a normal part of that workflow), and the orphaning happened live, in a
compliant session, during this very cluster's work.

**Fix**: when cwd is inside a git repo, the slug is now derived from
`git -C "$cwd" rev-parse --path-format=absolute --git-common-dir`
instead of the raw cwd. All worktrees of one repo share the same
`--git-common-dir`, so they now resolve to the **same** `progress.json` —
closing the orphaning bug at its root rather than patching around it.
Falls back to the old cwd-slug transform on **any** git failure (non-zero
exit, empty output, git binary missing) OR **non-absolute output**.

**Critical fix inside PR #24** (review fanout, independently reproduced): a
git older than 2.31 doesn't recognise `--path-format` and echoes it back
verbatim alongside a **relative** `.git` path, exiting **0** — not a failure
by exit code. The original implementation trusted any non-empty stdout, so
every repo on an old-git host would have collapsed onto one garbage shared
slug. Fixed by validating the captured value is actually an absolute path
(`case "$git_common_dir" in /*) ;; *) git_common_dir="" ;; esac`) before using
it; anything else falls through to the cwd-slug fallback, identical treatment
to an outright git failure. New test: a fake `git` that echoes non-absolute
garbage on exit 0 → helper falls back to the cwd-slug, not the garbage slug.

**Sanitisation collisions** (documented, not fixed — accepted tradeoff):
`session_id` sanitisation strips `/` (→ `_`) and collapses `..`, so e.g.
`"foo/bar"` and `"foo_bar"` both sanitise to `"foo_bar"` — two distinct raw
ids can collide. This is **replacement, not fresh-fallback**: a malformed id
still gets a stable, predictable path rather than being orphaned onto a
fresh-generated one. Accepted because `session_id` is harness-owned (Stop
payload / `$CLAUDE_CODE_SESSION_ID`), not attacker-controlled — this
sanitisation is defence-in-depth, not a security boundary. Documented at both
duplicate transform sites (`agentic_loop_path.sh` and
`als_sanitise_session_id` in `loop_state_common.sh`, kept intentionally
duplicated — `agentic_loop_path.sh` stays dependency-free by design) with
reciprocal "keep in lockstep" comments, plus a pinning test for the collision
itself.

**Also pinned**: cwd-with-spaces handling (already correct pre-PR, now locked
by a regression test).

**Clean-break, no compat flag**: old cwd-keyed state directories on disk are
left untouched (not migrated, not deleted) — a new loop simply resolves to
the new repo-keyed path going forward. The dashboard collector globbing all
slugs under `$HOME/.claude/agentic-loop/` was verified to still pick up both
old and new directory shapes.

**Accepted limitation** (design invariant, not a bug, re-affirmed): the
helper is stateless and re-derives the slug from current repo state on every
call. A session that changes its own cwd's repo-ness mid-loop (e.g. `git init`
on an until-then-non-git cwd, or `.git` disappearing) still splits its
`progress.json` state across old/new slugs — helper statelessness is a
deliberate design invariant, not something this PR guards against.

**Files**: `hooks/scripts/lib/agentic_loop_path.sh`,
`hooks/scripts/lib/loop_state_common.sh`,
`hooks/scripts/tests/agentic_loop_path.test.sh` (14→17 checks).

### Lineage: this reverses PR #86 point 6

[[pr_86_agentic-loop-hardening|PR #86]]'s point 6 (2026-07-01) considered and
**rejected** locking machinery for the single-loop-per-directory race,
calling it "disproportionate complexity for a rare, unsupported
configuration" and pointing at `coderails:using-git-worktrees` as the
resolution instead — i.e. "just use worktrees, don't key on them." The
calculus changed: the shared-checkout workflow now *prescribes* mid-session
`EnterWorktree` (2026-07-06 user directive), and a live orphaning incident in
an otherwise-compliant session showed that "just use worktrees" was itself
the trigger for a new, distinct bug (worktree hop → cwd changes → slug
changes → orphaned state) that PR #86's point-6 discussion didn't anticipate,
because at the time of #86 concurrent-session collision (not worktree
orphaning) was the concern being weighed. The fix landed here is a ~15ms
`git rev-parse` keying change — not the locking machinery #86 rejected — so
the two decisions aren't in tension: #86 rejected a heavier mechanism for a
narrower problem; this PR ships a lighter mechanism for the problem the
workflow shift actually created.

### Live proof

The fix re-keyed the very loop that shipped it, mid-flight: this session's
own `progress.json` state relocated from the worktree-cwd slug
(`-Users-harrison-Github-coderails-.claude-worktrees-loop-hardening-orchestrator`)
to the repo-keyed slug (`-Users-harrison-Github-coderails-.git`) the moment
PR #24 merged and a subsequent Stop hook re-resolved the path. Worth recording
as the first live consumer of the new keying scheme, observed in the wild
rather than only in tests.

## Wiki pages updated

- [[loop_state_guard]] — cwd fallback (F3-style) note, path-authority section
- [[unregistered_loop_guard]] — one-shot stderr-discard behaviour preserved
- [[loop-progress-fields]] — no direct change (fields untouched by this
  cluster); cross-referenced for context
- [[pr_86_agentic-loop-hardening]] — superseded-by note added to point 6
- [[design/agentic-loop-path-keying]] — new page: the repo-keyed slug design,
  the git<2.31 absolute-path guard, the sanitisation-collision tradeoff
- [[index]] — new design page + source entry
- [[log]] — this ingest + lint entries

## Caveats / gotchas

- The `als_log` stderr-swallowing tradeoff (PR #23) and the sanitisation
  collision tradeoff (PR #24) are both **accepted, documented, not fixed** —
  do not re-open either as a fresh finding without new evidence changing the
  calculus.
- The mid-loop re-keying this cluster's own fix caused is expected behaviour,
  not a residual bug: any loop that started before PR #24 merged and
  continues after will see its `progress.json` path change once, at the
  first Stop hook evaluation after merge. This is a one-time transition
  artifact, not a repeating instability.
