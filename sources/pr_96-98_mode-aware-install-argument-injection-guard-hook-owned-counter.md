---
title: "PRs #96–98 — mode-aware install sweep, command-substitution injection class closed, hook-owned loop-stop counter"
type: source
created: 2026-07-05
last_updated: 2026-07-05
sources:
  - install.sh
  - hooks/scripts/tests/install_mode_sweep.test.sh
  - commands/post-review.md
  - commands/merge.md
  - hooks/scripts/tests/post_review_command.test.sh
  - hooks/scripts/loop_stall_guard.sh
  - hooks/scripts/tests/loop_stall_guard.test.sh
  - skills/agentic-loop/SKILL.md
tags: [source, install, exec-bit, file-mode, security, injection, command-substitution, post-review, loop-stall-guard, progress-json, hook-owned]
---

# PRs #96–98 — mode-aware install sweep, command-substitution injection class closed, hook-owned loop-stop counter

## Repo rename (context for this and all future sources)

The GitHub repo moved to `blueman82/coderails-dev` mid-loop (old `blueman82/coderails`
URLs redirect). `origin` in the working checkout now points at
`https://github.com/blueman82/coderails-dev.git` (verified: `git remote -v`). This page is
the first source record to note the rename; README.md/INSTALLATION.md still show
the old clone URL — see Drift check below.

## PR #96 — mode-aware install sweep (merged 127a149, 2026-07-03T19:13:12Z)

`install.sh`'s "ARMING SCRIPTS" sweep unconditionally `chmod +x`'d every swept
script, fighting [[pr_92_exec-bit-sweep|PR #92]]'s and PR #94's git-index-mode
invariant (`scripts/lib/git-common.sh` and two other libs deliberately `100644`,
sourced-only, must never be executable) on every install run — a real
index-vs-installed-copy drift PR #93–94's source page had already flagged as
"surfaced, not actioned."

**Fix:** the sweep now reads each swept file's git index mode
(`git ls-files -s -- "$file" | awk '{print $1}'`) and branches:
- `100755` in index → ensure `+x` on disk (unchanged behaviour)
- `100644` in index → ensure `-x` on disk (**new** — corrects stale `+x`)
- not in the index (untracked/generated) → unconditional `+x` (old behaviour preserved)

New test `hooks/scripts/tests/install_mode_sweep.test.sh` builds a real `git
worktree` (so `git ls-files -s` inside it matches the real repo's index), overlays
the working tree's `install.sh`, deliberately drifts both a 644-index file (stale
`+x`) and a 755-index file (stale `-x`), runs the installer non-interactively, and
asserts both land on their index-mandated mode. A third case pins the no-git-index
(untracked file) fallback path.

### Critical caught pre-merge: installer died silently on non-git checkouts

Review reproduced a merge-blocking bug: the mode-read line ran under `set -euo
pipefail`. `git ls-files` exits 128 outside a git repository (e.g. a
release-tarball install with no `.git`), and that non-zero exit propagates
through the `| awk` pipe — `2>/dev/null` hid stderr but not the exit code.
Under `pipefail` the whole installer died at the very first swept file, silently
skipping everything after the sweep (marketplace registration, CLAUDE.md
discipline rules, memory seeding).

**Fix:** `|| _index_mode=""` on the assignment. The resulting empty value falls
through to the pre-existing `*)` fallback branch (unconditional `+x`), already
correct and tested. Confirmed exit 128 against the pre-fix script before
patching; new RED-first regression test copies the fixture into a directory with
`.git` fully removed, re-drifts the tracked files, runs `install.sh`, and asserts
(a) exit 0 rather than a mid-sweep death, (b) the 644-index lib still lands `+x`
via the fallback.

**Verification:** full suite 25/25 (24 prior + this one); `install.sh --dry-run`
smoke-tested. This closes the drift PR #94's source page flagged: "candidate fix
is a mode-aware sweep."

## PR #97 — command-substitution injection class closed (merged 76084fe, 2026-07-03T19:13:25Z)

**Security fix.** [[post-review|`/coderails:post-review`]]'s render-time PR-state
line — shipped in [[pr_93-94_post-review-injection-and-exec-bit-invariant|PR #93]]
— carried a command-substitution injection vulnerability via `$ARGUMENTS`. This
PR's own first-commit attempt at a fix (a "verdict line" re-displaying the
argument) reintroduced the identical flaw, then was abandoned in favour of
removing the class entirely.

**Experimentally confirmed (rev97-security, this session):** `$ARGUMENTS` is
textually spliced into a render-time `` !`cmd` `` line *before* the shell parses
it. A payload like `$(echo pwned)` executes as real command substitution the
moment the line renders — **regardless of surrounding double quotes, a `case`
guard, or any other in-line validation.** Quoting only constrains how a string is
interpreted *after* substitution already happened; it cannot stop the
substitution itself, because by the time the shell sees quotes, the payload's
shell syntax has already been spliced in as live text. Proof: a payload
appending a marker to a scratch file executed and the file was written, even
though the guard's own downstream logic reported the resulting string "INVALID."
This generalises what [[pr_93-94_post-review-injection-and-exec-bit-invariant|PR
#93]]'s "surfaced, not actioned" note undersold as a narrow ordering quirk — it
is a structural property of render-time `!`cmd`` substitution, not a fixable
quoting bug.

**Honest record:** PR #93 (07-03) shipped the vulnerable line. Two reviewers
rated it low-severity at the time without testing command substitution itself
— the theoretical risk was noted but not exercised. This experimental
methodology (actually running a payload) is what caught it; static/read-only
review had passed it twice.

### Fix

- `commands/post-review.md`: the PR-state line changed from
  `` !`gh pr view "$ARGUMENTS" ...` `` to the **argument-free**
  `` !`gh pr list --state open --limit 10` `` — matching `merge.md`'s existing
  convention (which was never vulnerable, since it never interpolated
  `$ARGUMENTS`). The render-time "Argument check" verdict line is removed
  entirely — it was itself vulnerable, which is how review proved the point.
- `## Step 0 — Argument gate` rewritten as a pure **model-level** instruction:
  verify the argument is a plain PR number by inspection (reasoning about the
  string), never by pasting it into a shell command; stop if empty or
  non-numeric; never interpolate an unvalidated argument into any command.
- Step 3's `HEAD_SHA=$(gh pr view "$ARGUMENTS" ...)` is unchanged — it remains
  the designed control point: a fenced-code-block command the model runs
  deliberately after clearing Step 0's gate, not an automatic render-time
  substitution. The distinction that matters: render-time `!`cmd`` lines execute
  unconditionally as the file renders, before the model reasons about anything;
  fenced-code-block commands only run when the model chooses to execute them,
  after Step 0's inspection gate has already run.
- **Class-wide test** added to `hooks/scripts/tests/post_review_command.test.sh`:
  no file in `commands/*.md` may carry a render-time `` !`cmd` `` line containing
  `$ARGUMENTS`, anywhere, ever. This is a standing regression guard against the
  entire vulnerability class recurring in any future command file, not just
  `post-review.md`.

**Verification:** RED confirmed (2 genuine failures from the vulnerable lines,
plus an unrelated BSD-grep option-parsing bug in the test itself, also fixed).
GREEN: 10/10 checks pass. Full suite 24/24. Manifest confirmed against
merge-base — exactly `commands/post-review.md` +
`hooks/scripts/tests/post_review_command.test.sh` differ.

## PR #98 — loop_stall_guard.sh is the sole writer of loop_stop_counts (merged 559d79b, 2026-07-05T21:29:56Z)

**Bug:** two writers of `progress.json`'s `loop_stop_counts` — the
`loop_stall_guard` hook's presence check, plus orchestrator prose in
`skills/agentic-loop/SKILL.md` asking the agent to self-maintain the same field
— raced under concurrent Stop-hook invocations, **undercounting 2 loops in a
recent session**. This was a lost-update race (two writers reading-then-writing
the same key without coordination), not a logic bug in either writer alone.

**Fix — sole writer:** `loop_stall_guard.sh` already validates every
`LOOP-STOP: <category> — <reason>` declaration against `LOOP_STOP_VOCAB` (the
shared vocab in `loop_state_common.sh`) before allowing the stop — the natural
place to also be the counter's sole writer. It now captures the matched
category and increments `loop_stop_counts.<category>` via a tmp-file
read-modify-write with `jq`, same pattern as `scripts/post_review.sh`'s
`write_cache`:

```bash
bump_loop_stop_count() {
  local category="$1"
  [ -n "$ALS_PATH" ] && [ -f "$ALS_PATH" ] || return 0
  local tmp="${ALS_PATH}.tmp"
  if jq --arg cat "$category" \
        '.loop_stop_counts[$cat] = ((.loop_stop_counts[$cat] // 0) + 1)' \
        "$ALS_PATH" > "$tmp" 2>/dev/null; then
    mv "$tmp" "$ALS_PATH" 2>/dev/null || { rm -f "$tmp" 2>/dev/null; als_log "counter_write=mv_failed"; }
  else
    rm -f "$tmp" 2>/dev/null
    als_log "counter_write=jq_failed"
  fi
}
```

`jq` auto-vivifies `.loop_stop_counts` and the per-category key on first write
(verified: piping `{}` through the same filter produces
`{"loop_stop_counts":{"hard-stop":1}}`), so no separate init branch is needed.
Category extraction reuses `LOOP_STOP_VOCAB` — one source of truth, not a second
vocab list.

**Design: best-effort, never fatal.** A missing `progress.json`, malformed JSON,
or a failed `mv` is logged via `als_log` and swallowed — the hook still exits 0
and lets the declared stop through. Declaring the stop matters more than the
counter write succeeding; a broken counter must never block every future stop in
every loop. This mirrors [[loop_state_guard]] and [[loop_stall_guard]]'s existing
honest-boundary posture (force the declaration, not the content).

**`SKILL.md` updated (body only) in 5 places** to state the field is now
**HOOK-OWNED**: the orchestrator reads it as-is and never computes, writes, or
increments it, carrying it forward verbatim on any wholesale `progress.json`
rewrite — same treatment as `completed_marker`.
1. Phase -2 stub: carry-forward instruction for `loop_stop_counts` alongside the
   existing `completed_marker` carry-forward (so a mid-session second loop
   doesn't reset the hook-maintained count).
2. Phase 0.5 orchestrator rules: appended clause — the `LOOP-STOP` declaration
   line is all that's required; `loop_stop_counts` is hook-owned, never
   self-write it.
3. Phase 13 self-audit: HOOK-OWNED clause on the `LOOP-STOP` category-counts
   bullet — read as-is, don't compute or edit.
4. Context-window persistence (field description): HOOK-OWNED clause +
   carry-forward-on-rewrite instruction, cross-referencing `completed_marker`.
5. Context-window persistence (lifecycle bullets): explicitly excludes
   `loop_stop_counts` from orchestrator-computed fields at phase boundaries;
   requires carrying it forward verbatim.

### Fix round (post-review) — invariants encoded as standing tests

No further `SKILL.md` changes this round (verified: diff against the prior head
touches only `loop_stall_guard.sh` + its test file). Four invariants added, each
a permanent regression guard:

1. **No-clobber guarantee** — fixture carries an arbitrary nested key
   (`custom_field.nested`) and a `work_units` array alongside the counter. After
   a valid `LOOP-STOP` declaration, both survive byte-identical
   (`jq -S 'del(.loop_stop_counts)'` diffed before/after) while the counter still
   increments.
2. **jq-absence is fail-open by design.** `bump_loop_stop_count` now opens with
   `command -v jq >/dev/null 2>&1 || return 0` — a missing `jq` short-circuits
   before any read-modify-write attempt. New test runs the guard with a `PATH`
   containing every coreutil it needs except `jq`, asserting the stop is still
   allowed.
3. **Multi-declaration tie-break: last wins.** A message with two `LOOP-STOP:`
   lines is proven to count the **last** one's category — intentional, because
   `SKILL.md` defines the declaration as the turn's *ending* line, so only the
   final one reflects the turn's actual outcome.
4. **Degraded-filesystem safety.** An unwritable progress directory
   (`chmod 555`) still allows the stop with no `.tmp` leak; an `ALS_PATH` under a
   never-created directory also still allows the stop, writing nothing.

**Test-harness bug found and fixed in the same round:** the `run_env` helper
(`env "$1" bash -c "... \$2 ... \$3 ..." _ "$2" "$GUARD"`) had an off-by-one —
pinning `arg0` to `_` shifts the payload and guard path into `$1`/`$2`, not
`$2`/`$3`. This made the jq-absence test **error out itself** (exit 127,
"command not found") rather than exercising the guard — a false green if left
unnoticed at "test written." Fixed to reference `$1`/`$2`.

**Verification:** `loop_stall_guard.test.sh` 21/21 (13 prior + 8 new/fixed);
full suite 24/24; scope check (`git diff origin/main...HEAD --name-only`) is
exactly `hooks/scripts/loop_stall_guard.sh`,
`hooks/scripts/tests/loop_stall_guard.test.sh`, `skills/agentic-loop/SKILL.md`
(the last untouched in this second round).

## ai_docs refresh (outside coderails repo, `~/.claude` commits bd949f0/1e5fc50/d96eddb)

Local `~/.claude` hooks-docs snapshot rewritten 8 events → 29 events
(`cc_hooks_docs.md`, 777 lines); subagents doc rewritten from a 32-line stub to a
full reference (`anthropic_docs_subagents.md`, 536 lines); slash-commands
snapshot now carries a "merged into skills" notice
(`anthropic_custom_slash_commands.md`). Context for why this mattered: the
[[pr_89-91_skills-doc-frontmatter-injection|PR #89–91]] cluster's stale
skills-docs snapshot (dated 2025-11-07, missing `paths`/`user-invocable`) had
already misled a PR-89 reviewer on 07-03 — this refresh is the same corrective
pattern applied to the hooks and subagents snapshots before they caused a
second stale-doc reviewer mistake.

## Process notes (this loop cluster)

- An org-auth outage killed a worker mid-fix-round; its uncommitted worktree
  work survived the kill, and a versioned respawn inherited it rather than
  restarting from scratch.
- `origin/main` advanced under the loop twice during this cluster; workers
  switched to merge-base-relative scope assertions (`git diff
  origin/main...HEAD --name-only` against the branch's own merge-base, not a
  fixed snapshot of main) to keep scope-check claims valid despite main moving.

## Drift check (scoped, report-only — not fixed by this ingest)

Grepped `README.md`, `AGENTS.md`, `INSTALLATION.md`, `docs/REFERENCE.md` on
current `main` for claims these three PRs made stale:

- **`INSTALLATION.md:73`** — `- arms the scripts (\`chmod +x\`)` describes the
  install sweep as unconditionally `+x`. As of PR #96 this is incomplete: the
  sweep is now git-index-mode-aware (`100755`→`+x`, `100644`→`-x`,
  untracked→legacy `+x`). Not incorrect for the common case (most swept files
  are `755` in the index) but no longer the full picture. **Flagged, not
  fixed** — narrow one-line prose change, left for a docs-focused pass.
- **Repo URL**: `README.md:22` and `INSTALLATION.md:41,45` still show
  `git clone https://github.com/blueman82/coderails.git` — predates the
  rename to `blueman82/coderails-dev` noted above. Old URLs redirect, so this
  is not functionally broken, but is stale. **Flagged, not fixed.**
- **`post-review` argument handling**: no README/INSTALLATION/REFERENCE prose
  found describing `$ARGUMENTS` handling or render-time injection risk in
  `post-review.md` — nothing stale to correct; this was always
  implementation-level detail, not user-facing doc content.
- **`loop_stop_counts` maintenance**: no README/INSTALLATION/REFERENCE mention
  of this field at all (it's `SKILL.md`-internal / `design/` territory) — no
  stale claim found there either.

## See also

- [[install-and-cache-trap]] — install.sh design page, needs a mode-aware-sweep
  cross-reference
- [[pr_92_exec-bit-sweep]] · [[pr_93-94_post-review-injection-and-exec-bit-invariant]]
  — the two prior PRs this cluster closes gaps for
- [[post-review]] · [[merge]] — command pages updated with the injection-class
  closure
- [[loop_stall_guard]] · [[loop_state_guard]] — hook pages
- [[spec-plan-progress-artifact-chain]] — design page with the counter/undercount
  discussion this PR resolves
- [[review-artifact-seam]] — broader truth-seam architecture this injection fix
  sits within
