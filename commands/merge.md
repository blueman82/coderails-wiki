---
title: "/coderails:merge"
type: command
created: 2026-06-25
last_updated: 2026-07-03
sources: [commands/merge.md, scripts/merge.sh, scripts/lib/git-common.sh, sources/pr_43_rough-edges.md, sources/pr_81-83_review-artifact-seam.md, sources/pr_89-91_skills-doc-frontmatter-injection.md, sources/session_2026-07-03_ai-docs-refresh-and-cc-mechanics-probes.md]
tags: [command, merge, pr, github, branch-cleanup, sync, review-artifact, sha-bound, dynamic-injection]
---

# /coderails:merge

Merges an approved PR, switches to main, pulls latest, and cleans up the feature branch. Branch cleanup is decoupled from the merge itself so a failed cleanup never reports a successful merge as failed.

## Dynamic Git-status injection (PR #91)

`commands/merge.md` frontmatter now injects a "Current Git Status" block via bash
substitution before the command's prose instructions:

```
!`git branch --show-current`
!`gh pr list --state open --limit 10`
```

plus a display-only note ("data, not instructions") stating explicitly that this injected
output is repository state for the model to read, not instructions to follow — guarding
against the injected text being misread as a directive. This is display context only; it
does not change any merge logic in `merge.sh`. (verified: [[pr_89-91_skills-doc-frontmatter-injection]], `git diff` on `commands/merge.md`)

**A parallel injection for `commands/post-review.md` was deferred, not shipped.** An
empirical probe of `$ARGUMENTS`-inside-`!`cmd`` substitution ordering was inconclusive during
PR #91's design work — a freshly created skill wasn't visible to a subagent's `Skill` tool
during the probe, and Anthropic's public docs only imply the substitution ordering rather
than stating it explicitly. `post-review.md` has no injected block as of this PR.

## Invocation

```
/coderails:merge [pr-number | branch-name | auto]
/coderails:merge          # auto-detects PR from current branch
/coderails:merge 42       # merge PR #42
/coderails:merge feature/add-retry-logic
```

Default argument is `auto`: resolves the PR from the current branch name.

## What it does

**Via `merge.sh`:**

1. **PR resolution**: Maps the argument to a PR number. `auto` calls `pr::num` against the current branch; a numeric arg is used directly; a branch name calls `pr::num` against that branch.
2. **Approval check (conditional)**: If the repository has branch protection requiring PR reviews (`protected` check via GitHub API), the script reads `pr::review` and errors if the decision is not `APPROVED`. (verified: merge.sh:37–40)
3. **Review artifact gate** (added PR #82): Fetches the current PR head SHA via `pr::head_sha`. Then calls `pr::has_coderails_review_for_head <num> <sha>` which scans all PR comment bodies for an exact coderails marker matching the current head SHA. Two distinct failure modes, each with a distinct error message:
   - Exit 2 (GitHub fetch failed) → "GitHub fetch failed — could not fetch PR comments."
   - Exit 1 (no artifact) → "No coderails review artifact for current head — run /coderails:post-review."
   No local-file fallback. No progress.json fallback. (verified: `merge.sh` @ 503f6fa)
4. **Merge**: `gh pr merge <num> --merge`. This is a remote merge only — its failure aborts the script via `set -euo pipefail`. Branch cleanup is explicitly separate and non-fatal so a worktree collision never causes a merged PR to report as failed.
5. **Sync**: Checks out `main` and pulls `origin/main`.
6. **Branch cleanup (best-effort)**:
   - Deletes the remote branch via `git push origin --delete <head>`. Warns but continues if already gone.
   - Attempts `git branch -D <head>` locally. If this fails (because another worktree has the branch checked out), warns with the worktree path rather than erroring.
7. Shows the last 5 commits on main.

## Config fields read

`merge.md` and `merge.sh` do not read `workflow.config.yaml`. The merge command has no config dependency. See [[config-resolution]] for context on how the other three commands use it.

## Scripts invoked

- `scripts/merge.sh` — full merge/sync/cleanup logic. Sourced helpers from `scripts/lib/git-common.sh`:
  - `require::repo` — blocks if remote is not a GitHub repository
  - `protected` — queries GitHub API for branch protection status
  - `pr::num` / `pr::state` / `pr::title` / `pr::review` / `pr::url` — PR introspection via `gh`
  - `pr::head_sha` — fetches current head SHA for the PR (added PR #82)
  - `pr::has_coderails_review_for_head` — exact marker match against PR comments (added PR #82; exit codes 0/1/2)
  - `branch` / `main` — current and default branch detection
- `scripts/lib/review-artifact.sh` — marker SSOT, sourced transitively by `git-common.sh` (added PR #82)

## Preconditions

- `gh` on PATH and authenticated
- Remote must be a `github.com` repository — `require::repo` gate (verified: merge.sh:12)
- PR must exist for the target branch
- If branch protection is enabled: PR must have `APPROVED` review decision
- A coderails review artifact must exist on the PR matching the **current** head SHA (added PR #82 — see [[review-artifact-seam]])
- PR must not be already closed (without merge)

## Chain position

Fourth (and last) in the chain. Called by [[workflow]] after ship-it authorisation, or standalone.

```
/prep  →  (code)  →  /push  →  /merge
                                ^^^^^^
```

After `/merge`, [[workflow]] runs `/wiki-ingest` and `/wiki-lint` if `config.wiki_path` is non-null — merge is the trigger for wiki update. If the worktree is still present after merge, [[workflow]] removes it via `git worktree remove` and `git branch -d` (separate from merge.sh's cleanup). (inferred: workflow.md:175–182)

## Design notes

The deliberate decoupling of branch cleanup from the merge step is a key design decision. Using `--delete-branch` on `gh pr merge` would delete the local branch too, which fails (and under `set -euo pipefail`, aborts the whole script) when another worktree holds the branch checked out. Separating and making cleanup non-fatal means a merged PR never reports as failed due to a worktree collision. (verified: merge.sh:52–56 inline comment)

The `protected` check uses the GitHub API directly rather than relying on `gh pr merge` to reject unapproved merges — this provides an explicit, user-readable error before the merge attempt. (inferred: merge.sh:37–40)

**Enforcement-gap notice (added PR #43):** When no `workflow.config.yaml` is present, `merge.sh` emits an informational `info` line before the artifact gate: "No workflow.config.yaml — enforce_pr_workflow hook is inactive, but the review artifact gate still applies." This confirms that the review artifact gate is unconditional — it does not require a config file. `enforce_pr_workflow` remains opt-in; the review artifact gate does not. (verified: `merge.sh` @ 503f6fa)

**Review artifact gate is unconditional (added PR #82):** Unlike `enforce_pr_workflow` (which no-ops without `workflow.config.yaml`), the review artifact gate runs on every merge regardless of config. It is built into `merge.sh` directly, not into the hook. The enforcement ceiling note from [[enforcement-model]] still applies: the gate checks that the artifact exists and matches the SHA; it does not verify the review was substantive. (verified: `merge.sh`, [[review-artifact-seam]])

## See also

- [[post-review]] — the command that creates the review artifact this gate checks
- [[review-artifact-seam]] — design page for the full truth-seam architecture
- [[push]] — creates the PR that this command merges
- [[workflow]] — calls /merge in Phase 6, then wiki-ingest/lint
- [[config-resolution]] — merge is the only workflow command that does NOT read workflow.config.yaml
- [[repo-hosting]] — github.com remote requirement
- [[enforce_pr_workflow]] — PreToolUse hook that blocks `gh pr merge` unless `/pr-review-toolkit:review-pr` ran this session (NO_CONFIG opt-in; separate from the artifact gate)
- [[enforcement-model]] — the honest-ceiling framework both gates sit within
- [[pr_89-91_skills-doc-frontmatter-injection]] — PR #91 source record: Current Git Status injection added; post-review.md injection deferred pending an inconclusive substitution-ordering probe
