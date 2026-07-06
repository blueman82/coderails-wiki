---
title: "/coderails:push"
type: command
created: 2026-06-25
last_updated: 2026-07-06
sources: [commands/push.md, scripts/push.sh, scripts/lib/git-common.sh, sources/pr_47_strictcode-skill-config.md, sources/pr_11-14_gate-hardening-followups.md]
tags: [command, push, pr, commit, jira, strictcode, github, staging, untracked-files]
---

# /coderails:push

Stages, commits, pushes to origin, and creates or updates a GitHub PR. After PR creation, auto-resolves the linked Jira ticket if one was stored by [[prep]].

## Invocation

```
/coderails:push [commit message] [--quick]
/coderails:push "Add retry logic with exponential back-off"
/coderails:push --quick
```

`--quick` skips the engineering-principles pre-flight entirely. Stripped from arguments before passing to `push.sh`.

## What it does

**Pre-flight — engineering-principles check (skipped by `--quick` or if `config.engineering_principles_paths` is null):**

1. Determines the base branch via `git symbolic-ref refs/remotes/origin/HEAD`.
2. Runs `git diff --name-only $(git merge-base HEAD <base>)..HEAD` and checks for matches against `config.engineering_principles_paths` patterns.
3. If matches found, runs `config.engineering_principles_skill` (default: `/engineering-principles-python`) on those files. (updated PR #47)
4. Blocking findings (architectural violations, DI protocol deviations) prompt the user: "Fix before pushing, or push as-is?" Non-blocking findings (style, naming) are logged and execution continues.
5. In non-interactive contexts (conductor, background agent), logs findings but does not block — always proceeds to push. (verified: push.md:22)

**Push (via `push.sh`):**

1. If the working tree is dirty, stages all changes (`git add -A`) and commits with the provided message or an auto-generated "Update N files" message.
2. If a Jira key is present in branch config, prefixes both the commit message and PR title with it — JIRA's GitHub integration uses this prefix to link commits to tickets even after squash merge. (verified: push.sh:29, push.sh:54)
3. Pushes the branch to `origin` with `-u` to set upstream.
4. If a PR already exists for the branch, posts a comment and reports the URL. If not, creates a PR with `gh pr create`, using the humanised branch name as title and the last 10 commits as body.

**Jira auto-resolve (after PR creation):**

Reads `git config branch.<current-branch>.jira-ticket`. If a key is found, transitions the ticket to `config.jira.transitions.resolve` with a standard comment: `"Resolved via PR merge. Work implemented via AI-assisted development (Claude Code). Branch: $BRANCH."` Then reports the resolved key alongside the PR URL. (verified: push.md:57–68)

If no key is stored, skips silently — not all branches have tickets.

## Config fields read

See [[config-resolution]] for how `workflow.config.yaml` is located at runtime.

| Field | Used for |
|---|---|
| `config.engineering_principles_paths` | Path patterns that trigger the pre-flight check |
| `config.engineering_principles_skill` | Slash-command for the engineering-principles pre-flight (absent/null → defaults to `/engineering-principles-python`; also `/engineering-principles-go` or `/engineering-principles-ts`). To skip engineering-principles entirely, set `engineering_principles_paths: null`. Added PR #47 |
| `config.jira.transitions.resolve` | Transition name for Jira auto-resolve after PR creation |
| `config.jira.mcp_namespace` | Jira MCP tool namespace for the resolve call (default: `jira`) |

`NO_CONFIG` or absent `config.engineering_principles_paths`: skip engineering-principles entirely, proceed to push.

## Scripts invoked

- `scripts/push.sh` — core commit/push/PR logic. Sourced helpers from `scripts/lib/git-common.sh`:
  - `require::feature` — blocks if currently on `main`/`master`
  - `require::repo` — blocks if the remote is not a GitHub repository (verified: push.sh:11–12; git-common.sh:50–52)
  - `dirty` / `ahead` / `ahead_list` — state checks
  - `pr::exists` / `pr::num` / `pr::url` — PR introspection via `gh`

## Preconditions

- Must be on a feature/bug branch (not `main` — `require::feature` gate)
- Remote must be a `github.com` repository (not GitLab or Bitbucket) — `require::repo` gate (verified: git-common.sh:33–35)
- `gh` on PATH and authenticated
- Working tree may be dirty (changes will be staged) or already committed and ahead of origin

## Chain position

Third in the chain. Called by [[workflow]] after the code loop, or standalone.

```
/prep  →  (code)  →  /push  →  /merge
                      ^^^^^
```

The PR URL output here feeds [[workflow]]'s Phase 4 ledger comment and is the URL reported to the user before the ship-it pause.

The engineering-principles pre-flight in [[workflow]] Phase 3 and the pre-flight inside this command overlap intentionally: [[workflow]] catches large diffs not covered by `config.engineering_principles_paths` patterns; `/push` catches path-pattern matches if [[workflow]]'s pre-flight was skipped. (inferred: workflow.md:139–141)

## See also

- [[prep]] — sets the `jira-ticket` branch config key consumed here
- [[workflow]] — calls /push in Phase 4
- [[merge]] — next step after push
- [[config-resolution]] — walk-up config resolver
- [[repo-hosting]] — github.com remote requirement enforced by `require::repo`
- [[enforce_pr_workflow]] — PreToolUse hook that blocks `gh pr create` unless this command ran first (NO_CONFIG opt-in)
