---
title: "PRs #86, #87, #91, #94, #107 — loop-lib residuals cluster"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, agentic-loop, loop-state, jq, malformed-transcript, dashboard, obsidian, gitignore, task-evals]
---

# PRs #86, #87, #91, #94, #107 — loop-lib residuals cluster (2026-07-08)

Five independently-merged PRs, all 2026-07-08, grouped as one source page because
they share a theme: closing residuals left open by earlier same-day loop-state and
dashboard work. The main thread is #91 → #107 (malformed-transcript tolerance in
the shared loop-state lib, then a same-day failure-attribution correction);
#86, #87, #94 are smaller, unrelated fixes swept into the same ingest pass.

**⚠️ PR-number collision:** this repo's `#86`/`#87` (2026-07-08) are unrelated to
the older `sources/pr_86_agentic-loop-hardening.md` (merged 2026-07-01,
`b8a1958`) and `sources/pr_87_agentic-loop-path-session-keying.md` (merged
2026-07-01, `344a849`) — both from before the repo recreation described in
[[repo-hosting]]. This page's #86/#87 are the 2026-07-08 obsidian-exec and
nested-worktree PRs, not the older content. Filenames disambiguate: this page's
slug carries the `2026-07-08` cluster theme, not a bare `pr_86_`/`pr_87_` name.

## PR metadata

| PR | Title | Merge SHA | Merged |
|---|---|---|---|
| #91 | loop state tolerant parse | `f1db15c4` | 2026-07-08 09:52 |
| #107 | loop state failure attribution | `d4256542` | 2026-07-08 15:09 |
| #86 | obsidian exec buildargv parity | `9b300355` | 2026-07-08 09:15 |
| #87 | test/install mode sweep nested worktree | `81a705f6` | 2026-07-08 09:20 |
| #94 | chore/remove leaked evals json | `f4e05ee9` | 2026-07-08 14:48 |

## #91 → #107: malformed-transcript tolerance in `loop_state_common.sh`

### The bug (#91)

`hooks/scripts/lib/loop_state_common.sh` had two `jq -s` (slurp-the-whole-file)
call sites. A single malformed JSONL line anywhere in a transcript aborted the
**entire** parse, not just that line:

- `als_count_invocations` — one bad line collapsed the invocation count to
  empty/0. `als_gate_require_active_loop` then read `ALS_INVOCATIONS=0` and
  treated a genuinely active agentic loop as "not a loop," defeating
  [[loop_state_guard]], [[loop_stall_guard]], [[voice_announce]], and (via the
  shared delegate `ulg_has_skill_invocation`) [[unregistered_loop_guard]].
- `als_extract_last_text` — same failure mode over the tail-window slurp; a
  malformed line collapsed the final-message extraction to empty, indistinguishable
  from "no text yet."

### The fix (#91)

Both functions now pre-filter each line through `jq -R 'fromjson? // empty'`
before the aggregating `jq -s`, so a malformed line is dropped instead of
aborting the whole parse. `als_count_invocations` additionally counts dropped
lines and reports `skipped_malformed=N` on stderr (only when N>0, mirroring the
existing reason-tag stderr-only convention); `als_stable_invocations` folds
that into its existing single per-call summary log line.

A pre-existing test asserted the old `reason=jq_parse_error` value for a
scenario now correctly reported as `reason=none skipped_malformed=1` —
updated in this PR. `reason=jq_parse_error` narrows post-fix: pre-fix, ANY
malformed line caused it (the whole slurp failed); post-fix it's only
reachable via an actual jq-binary-level failure, not malformed content.

Verification (#91): full suite 36/36 suites passed; tier-1 PR-scope evals, 4
evals, all P0 pass, each with a negative control confirmed to fail against the
pre-fix lib; RED-then-GREEN on all new/modified assertions.

### The post-merge correction (#107, same day)

A 4-reviewer verification pass on #91 found that the malformed-line tolerance
had introduced two **failure-attribution regressions** — the fix stopped the
false "not a loop" outcome, but in doing so silently absorbed two genuinely
distinct failure conditions into the same non-signal. #107 fixes these forward:

- **`read_error` tag** — an unreadable transcript file or a stage-1 `jq`
  binary-level death (jq crashes, not "line didn't parse") previously came back
  as an empty/zero signal indistinguishable from "clean, nothing to report."
  #107 checks readability and stage-1 exit status explicitly, before either
  count is computed, so these surface their own `read_error` reason instead of
  masquerading as a benign skip.
- **`all_lines_malformed` tag** — when stage 1 succeeds but produces **zero**
  parsed lines out of a **non-empty** input (every single line malformed),
  that's a total parse loss, not "N lines skipped, rest counted." Pre-#107 this
  read identically to `skipped_malformed=N`; #107 reports it as a distinct
  `all_lines_malformed` reason. This distinction is load-bearing: it gates the
  retry/settle loop in `als_stable_invocations`, restoring the 5-attempt
  flush-race window that a mis-attributed "clean, count=0" result would have
  short-circuited past.

Also in #107: comment corrections (the header comment's description of the
tolerant-parse behavior updated to match), and 4 new tests covering the two new
tags plus the retry-window interaction.

### Blast radius

Both changes are confined to `hooks/scripts/lib/loop_state_common.sh` but affect
every consumer that reads its shared functions: [[loop_state_guard]],
[[loop_stall_guard]], [[voice_announce]], and [[unregistered_loop_guard]] (via
its own delegate call into the shared lib, not its own parsing logic).

### Known residuals, explicitly out of scope for #91/#107

- **`unregistered_loop_guard.sh`'s own `jq -s` slurp is untouched.**
  `ulg_count_dispatch_turns` (inside `unregistered_loop_guard.sh` itself, not
  the shared lib) has the identical bare-`jq -s` fragility — a single malformed
  line there still collapses its own dispatch-turn count. This was flagged as
  out-of-scope in #91's own PR description and remains a standing residual.
- **`discipline_common.sh`'s `dc_extract_last_text` carries the identical bare
  `jq -s` fragility.** Different lib family (the 3-discipline-hook shared lib,
  not the loop-state lib) — not touched by this cluster.

## #86: obsidian `exec.ts` `pressButton` buildArgv parity

`skills/dashboard/obsidian/src/exec.ts`'s `pressButton` function now wraps its
`buildArgv(button, input)` call in a `try`/`catch`, returning
`{ ok: false, reason: "invalid-input" }` on throw instead of letting the
exception propagate uncaught:

```ts
let argv: string[];
try {
  argv = buildArgv(button, input);
} catch (err) {
  console.error(`pressButton: ${(err as Error).message}`);
  return { ok: false, reason: "invalid-input" };
}
```

This gives the Obsidian command-centre plugin's direct-exec path the same
error handling the dashboard web app's `route.ts` already had — `buildArgv`
throws on an empty effective prompt (no command and no input) or a
leading-whitespace-then-dash flag-smuggling shape, and pre-#86 `pressButton`
had no catch around that call, so either input crashed the press instead of
resolving to a `PressResult`. The `input.startsWith("-")` check just above the
try/catch is a pre-existing, separate guard for the plain-leading-dash case;
the new catch covers what that guard doesn't (whitespace-padded dashes and the
empty-prompt case, both only detectable inside `buildArgv` itself).

## #87: `install_mode_sweep.test.sh` nested-worktree investigation (no fix)

A reported residual: `install_mode_sweep.test.sh` was suspected to fail when
run from inside a worktree checked out under `.claude/worktrees/`, based on a
"worktree of a worktree" concern. Investigated both the plain nested-worktree
case and the stronger "worktree of a worktree" reading — **did not reproduce**
in either form. Full suite (35/35) and the test itself (27/27 checks) passed
from inside a nested worktree, and a worktree created *from* a nested worktree
still anchors its `.git` file at the **original** repo's administrative area
(`git worktree` has no "worktree of a worktree" special case — every worktree,
however deep the creation chain, resolves against the original clone).

**Outcome: document, not fix.** A comment-only addition to
`install_mode_sweep.test.sh`'s header states the investigated invariant
(worktree administrative-area anchoring) and what would have to break for the
concern to resurface. Zero executable lines touched; `install.sh` needed no
change.

## #94: leaked root `evals.json` removed (third occurrence)

The task-evals skill's working file `evals.json` — a PR-scope working artifact,
never the durable one (that's the SHA-bound PR comment `/coderails:post-evals`
posts) — had again been committed to the repo root via the auto-commit-on-Edit
friction the wiki's own `[[task-evals]]`-adjacent hooks carry. This is the
**third** occurrence of this exact leak.

Fix: `git rm evals.json` at the repo root, plus a root-anchored `.gitignore`
entry:

```
# task-evals working artifact — PR-scope working material only, never a durable artifact (that's the PR comment)
/evals.json
```

The leading `/` anchors the ignore to the repo root only. A bare `evals.json`
pattern (no anchor) would match at any depth and silently shadow the
legitimately-tracked dashboard test fixture at
`skills/dashboard/app/test/fixtures/projects/loops/-work-project/S1/evals.json`
— confirmed post-fix via `git check-ignore` returning exit 1 (not ignored) for
that fixture path while the anchored root pattern matches the leaked file.

## Verification

- (verified) #91/#107 diff and header-comment content read directly from
  `git show f1db15c47dd5032cd8a9cdb0d82d14db67c858e1:hooks/scripts/lib/loop_state_common.sh`
  and the post-#107 version at `d425654234f77303ed15744630e541423a3602b2`.
- (verified) #86 diff read directly from
  `git show 9b3003556e9a311106965e3e0d45a0e1920feed1:skills/dashboard/obsidian/src/exec.ts`.
- (verified) #94's `.gitignore` anchoring confirmed against the current repo
  `.gitignore` (line 26: `/evals.json`).
- (inferred) #107's PR body on GitHub is empty/a bare commit-log dump (not a
  written description) — the summary above is reconstructed from the
  orchestrating session's brief plus direct source-diff verification, not from
  a `gh pr view` body field.

## See also

- [[loop_state_guard]], [[loop_stall_guard]], [[voice_announce]],
  [[unregistered_loop_guard]] — the four consumers of `loop_state_common.sh`
  affected by the #91/#107 fix
- [[dashboard]] — the Obsidian command-centre plugin #86 patches
- [[task-evals]] — the skill whose working artifact #94 un-leaked
- [[repo-hosting]] — PR-number collision convention this page follows for its
  #86/#87 disambiguation
