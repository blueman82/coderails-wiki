---
title: "agentic_loop_path.sh: repo-keyed (not cwd-keyed) progress.json slug"
type: design
created: 2026-07-06
last_updated: 2026-07-08
sources:
  - sources/pr_23-24_hook-lib-observability-and-repo-keyed-loop-state.md
  - sources/pr_87_agentic-loop-path-session-keying.md
  - sources/pr_86_agentic-loop-hardening.md
tags: [design, agentic-loop, progress-json, session-keying, worktree, git-common-dir]
---

# agentic_loop_path.sh: repo-keyed (not cwd-keyed) progress.json slug

`progress.json`'s path is keyed on the **repository** (via
`git --git-common-dir`), not the raw working directory, so that every
worktree of one repo shares the same loop state. This closed a real,
observed orphaning bug: a mid-loop `EnterWorktree` hop used to silently
split a loop's state across two paths.

## Context

`agentic_loop_path.sh` is the sole path authority for
`$HOME/.claude/agentic-loop/<slug>/<session_id>/progress.json` (see
[[loop_state_guard]]). Before this change, `<slug>` was the raw cwd with `/`
replaced by `-`. [[pr_87_agentic-loop-path-session-keying|PR #87]]
(2026-07-01) had already added `session_id` to the key, fixing the
concurrent-sessions-in-one-directory race — but the slug component was still
cwd-only.

The shared-checkout workflow now prescribes mid-session `EnterWorktree`
(2026-07-06 user directive) — an orchestrator hopping into an isolated
worktree partway through a loop is normal, expected behaviour, not an edge
case. Because `EnterWorktree` changes `$PWD` to a different path (the
worktree's own directory) while operating on the *same underlying repo*, the
old cwd-only slug computed a different, brand-new path the instant the hop
happened — orphaning all of the loop's existing `progress.json` state. This
happened live, in a compliant session, during the cluster that shipped the
fix ("Live proof" below).

## The rule

When `$PWD` (or the `cwd` argument) is inside a git repository, the slug is
derived from:

```
git -C "$cwd" rev-parse --path-format=absolute --git-common-dir
```

slugified the same way as before (`/` → `-`). All worktrees of a single repo
report the *same* `--git-common-dir` (the shared `.git` directory, or the
common dir for a linked worktree), so every worktree resolves to one shared
`progress.json` path automatically — no explicit migration or lookup table
needed.

Falls back to the pre-existing cwd-slug transform when:
- `cwd` is not inside a git repo, OR
- the `git rev-parse` call fails (non-zero exit, empty output, git binary
  missing), OR
- the captured output is **not an absolute path** (see the git<2.31 guard
  below).

## Rationale: why validate absoluteness, not just exit code

A git older than 2.31 does not recognise `--path-format` at all. Rather than
erroring, it **echoes the flag back verbatim** alongside a **relative** `.git`
path, and exits **0**. Trusting "non-zero exit = failure" alone would have
let this poisoned, non-absolute output through as a valid slug — collapsing
every repo on an old-git host onto one shared garbage slug (all such repos'
loops would silently share state). The fix validates the captured value is
actually an absolute path (`case "$git_common_dir" in /*) ;; *) git_common_dir="" ;; esac`)
before using it, treating "non-absolute" identically to an outright git
failure: fall through to the cwd-slug.

## Sanitisation collisions: accepted tradeoff

Independently of the slug change, `session_id` sanitisation (stripping `/`,
collapsing `..`) can make two distinct raw ids collide — e.g. `"foo/bar"` and
`"foo_bar"` both sanitise to `"foo_bar"`. This is **replacement, not
fresh-fallback**: a malformed id gets a stable, predictable path rather than
being silently orphaned onto a freshly-generated one. Accepted because
`session_id` is harness-owned (Stop-hook payload or
`$CLAUDE_CODE_SESSION_ID`), not attacker-controlled — this transform is
defence-in-depth against payload anomalies, not a security boundary. The
transform is intentionally duplicated (not shared via `source`) between
`agentic_loop_path.sh` and `als_sanitise_session_id()` in
`loop_state_common.sh`, because `agentic_loop_path.sh` stays dependency-free
by design; both copies carry a "keep in lockstep" comment pointing at the
other, and a pinning test locks the collision behaviour itself so it can't
silently change.

## Where it is enforced

- `hooks/scripts/lib/agentic_loop_path.sh` — the slug computation and both
  guards (git failure, non-absolute output)
- `hooks/scripts/tests/agentic_loop_path.test.sh` (14→17 checks) — including
  a fake-git fixture that echoes non-absolute garbage on exit 0, asserting
  fallback to the cwd-slug rather than the poisoned value
- Consumed identically by [[loop_state_guard]] (reader) and the
  agentic-loop orchestrator (writer, via a `Bash` call) — both resolve
  through this one helper, per its "sole path authority" invariant

## Known caveats / edge cases

- **Stateless by design**: the helper re-derives the slug from current repo
  state on every call. A session that changes its own cwd's repo-ness
  mid-loop (`git init` on an until-then-non-git cwd, or `.git` disappearing)
  still splits its `progress.json` state across old and new slugs — this is
  accepted as rare and self-inflicted, not guarded against.
- **Clean break, no compat flag**: old cwd-keyed state directories on disk
  are left untouched, not migrated. The dashboard's collector globs all
  slugs under `$HOME/.claude/agentic-loop/`, so both old and new directory
  shapes remain visible.
- **One-time re-keying transition**: any loop active when this change
  merged saw its own `progress.json` path change once, at the next Stop-hook
  evaluation — expected, not a bug. See "Live proof" below.
- **Plugin-version upgrade is a second, distinct re-keying trigger, undocumented until now.**
  The re-keying transition above describes a same-version, mid-loop merge. A
  *different* trigger was found live (2026-07-07, during the
  [[assistant-link-send-gate-architecture]] cluster): a consumer whose
  installed plugin cache predates PR #24 entirely — verified: the 1.0.0 cache
  has no `agentic_loop_path.sh` file at all, i.e. no `--git-common-dir` keying
  logic existed yet, only the older cwd-slug scheme — silently switches slug
  schemes the moment the plugin auto-updates past PR #24's version, exactly
  like the mid-loop-merge case, but with **no loop-side signal that a re-key
  is imminent** (unlike a merge, which the active session observes). A
  session with a maintained `progress.json` under the old slug can get
  orphaned and reported as an *unregistered* loop by the newer guards
  (`unregistered_loop_guard.sh` et al., now live post-update) despite having
  faithfully maintained state the whole time. Workaround applied live: a
  symlink from the new slug to the old state directory. **RESOLVED 2026-07-08
  by [[pr_103_agentic-loop-path-session-fallback|PR #103]]** (the first of the
  two candidate fixes named here — a session-id fallback probe in the helper).
  See "Update — session-id fallback probe" below.

## Live proof

The fix re-keyed the very loop that shipped it, mid-flight: this session's
`progress.json` moved from the worktree-cwd slug
(`-Users-harrison-Github-coderails-.claude-worktrees-loop-hardening-orchestrator`)
to the repo-keyed slug (`-Users-harrison-Github-coderails-.git`) as soon as
the change merged and the next Stop hook re-resolved the path — the first
live consumer of the new keying scheme, observed directly rather than only
in tests.

## Lineage: supersedes PR #86 point 6

[[pr_86_agentic-loop-hardening|PR #86]]'s point 6 (2026-07-01) rejected
locking machinery for the single-loop-per-directory race as "disproportionate
complexity for a rare, unsupported configuration," pointing at
`coderails:using-git-worktrees` as the resolution instead. That calculus
changed once the workflow itself started prescribing mid-session
`EnterWorktree` hops: "just use worktrees" became the trigger for a *new*
bug (state orphaning on worktree hop) that #86's discussion didn't anticipate,
because it was weighing concurrent-session collision, not worktree
orphaning. The fix here is a ~15ms `git rev-parse` keying change, not the
heavier locking machinery #86 rejected — the two decisions are not in
tension; #86 rejected a heavy mechanism for a narrower problem, this ships a
light mechanism for the problem the workflow shift created.

## Update — session-id fallback probe (2026-07-08, PR #103)

The plugin-version re-keying gap from the "Known caveats" bullet above
**recurred and was fixed**. Merge commit `bf02e8a`.

**Incident.** Session `46d6c1b5` registered its loop's `progress.json` under
the legacy raw-cwd slug
(`-Users-harrison-Github-coderails-.claude-worktrees-routines`) because its
checkout predated PR #24. The live Stop hooks ran from a newer installed
plugin cache (current main) whose helper keys the slug by `--git-common-dir`
(`-Users-harrison-Github-coderails-.git`). The readers resolved the canonical
git-common-dir path, found nothing, and `unregistered_loop_guard` nudged a
COMPLETED, registered loop on every Stop; the same split earlier cost the loop
its `loop_stop_counts` at Phase 13.

**Mechanism (the general lesson).** The "sole path authority" is only sole
**per copy**. The writer (orchestrator intake) and the readers (Stop hooks)
can run *different helper versions*, and a session's cwd can also drift
mid-loop. So the same session's state can be written under one slug and read
back under another. Any fix that requires version lockstep or a stable cwd is
therefore insufficient.

**Fix.** `agentic_loop_path.sh` now, when the canonical slug path has no
`progress.json`, probes `<base>/*/<session_id>/progress.json`. `session_id` is
unique per session, so it is a sufficient key on its own — state parked under
*any* slug for this session is found. Resolution order: canonical-exists →
print canonical (unchanged); else probe → print the match; else (nothing
anywhere) → print canonical so a fresh loop registers there. Matches are
deduped by the **physical identity** of their containing dir (the live
workaround symlinks the same file under several slugs; `pwd -P` collapses
them), with a deterministic pick if distinct real files somehow coexist. The
existing `session_id` sanitisation now runs **load-bearingly before** the
glob, so the `<session_id>` segment cannot expand into a sibling/parent dir.
The helper stays dependency-free, pure (prints a path, creates nothing),
bash-3.2-safe, and empty-glob-safe.

This largely **heals** the "stateless by design" caveat too: a mid-loop
repo-ness change no longer splits state *once a `progress.json` has been
written*, because session_id finds it under the old slug. The one residual
gap: if the slug changes *before any file is written*, there is nothing to
probe for and the fresh registration lands at the new canonical path (correct
behaviour). The live workaround symlinks were left in place — they remain
load-bearing for session `46d6c1b5` until its plugin cache refreshes.

**Verify.** TDD: 5 new checks in `agentic_loop_path.test.sh` (17→22) —
canonical-exists-wins, the incident (legacy-slug found), fresh registration,
symlink-duplicate dedupe, generated-fallback-id resolution. Full `run_all.sh`
green (37/37) under bash 5 and bash 3.2. PR-scope evals GO, tier 1, 6/6 P0
against a fresh clone with passing negative controls. `docs/REFERENCE.md`
updated to describe the probe.

## See also

- [[loop_state_guard]] — the hook that reads via this path authority
- [[pr_87_agentic-loop-path-session-keying]] — PR #87: added `session_id` to
  the key (the other half of the current `<slug>/<session_id>/` path shape)
- [[pr_86_agentic-loop-hardening]] — point 6's original (now superseded)
  rejection of locking machinery
- [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state]] — this
  cluster's full source record
- [[using-git-worktrees]] — the worktree mechanics this keying scheme now
  supports natively rather than routing around
- [[assistant-link-send-gate-architecture]] — where the plugin-version
  re-keying gap above was found live, alongside the sibling stale-plugin-cache
  finding for the same 1.0.0→1.1.0 update
