---
title: "/coderails:workflow"
type: command
created: 2026-06-25
last_updated: 2026-06-30
sources: [commands/workflow.md, sources/pr_47_strictcode-skill-config.md, sources/pr_81-83_review-artifact-seam.md]
tags: [command, workflow, orchestrator, worktree, pr-review, wiki, jira, post-review, review-artifact]
---

# /coderails:workflow

The umbrella orchestrator for the canonical feature-change cycle. Chains [[prep]] → code → [[push]] → adversarial review → [[merge]] → wiki, with two interactive pauses for the developer.

## Invocation

```
/coderails:workflow <branch> [description]
/coderails:workflow feature/add-retry-logic
/coderails:workflow bug/fix-timeout --summary "Request timeout too short"
```

Branch name is required and must start with `feature/`, `bug/`, or `bugfix/`. If absent, the command asks once rather than guessing.

## What it does

The command runs five phases, two of which pause for human input:

**Phase 1 — Prep (auto)**: Delegates to [[prep]] with parsed args. Creates the worktree and Jira ticket (if `config.jira` is non-null).

**Phase 2 — Orient (auto, conditional)**: Runs `/wiki-query` against the feature area to surface known constraints, open gaps, and adjacent behaviour before a line is written. Skipped if `config.wiki_path` is null. If a gap worth preserving is found, files an `investigations/<topic>_<YYYY-MM-DD>.md` page now, not post-deploy. Reports "wiki clear" if nothing found.

**Phase 2b — Design adversarial review (conditional)**: If the investigation page meets any trigger (≥40 lines, spans >1 service, new DDB schema, LLM call in the data path), launches 2–3 specialist agents in a single parallel message. Agent selection is driven by what the design touches, not a fixed list. Findings are classified as accept/skip and applied to the investigation page before coding starts. (inferred: agent table in workflow.md:109–116)

**Phase 3 — Code (interactive pause)**: Hands control to the developer. Does not proceed until a ready signal ("push", "ship it", "done coding"). Pre-flight runs `config.engineering_principles_skill` (default: `/engineering-principles-python`) on changed files matching `config.engineering_principles_paths` or any file with ≥20 lines changed. (updated PR #47)

**Phase 3 (end) — Post-review artifact (auto, added PR #83)**: After `review-pr` and simplify complete, runs `/coderails:post-review <PR#>`. Posts a machine-marked, SHA-bound review summary to the PR. This creates the artifact that [[merge]] gate-checks before `gh pr merge`. See [[post-review]] and [[review-artifact-seam]].

**Phase 4 — Push + adversarial review (auto after ready signal)**: Calls [[push]], then `/pr-review-toolkit:review-pr all` (four specialist agents in parallel). Findings are classified blocking/worthwhile/cosmetic and applied inline without re-asking per finding. A ledger comment is posted to the PR.

**Phase 5 — Ship-it (interactive pause)**: Waits for merge authorisation ("ship it", "merge", "ok to merge").

**Phase 6 — Merge + wiki (auto after ship signal)**: Calls [[merge]], then (if `config.wiki_path` non-null) runs `/wiki-ingest` and `/wiki-lint`. Cleans up the worktree after merge.

## Config fields read

See [[config-resolution]] for how `workflow.config.yaml` is located at runtime.

| Field | Used for |
|---|---|
| `config.worktree_base` | Passed to [[prep]] for worktree path derivation |
| `config.worktree_script` | Passed to [[prep]]; if null, plain `git worktree add` |
| `config.jira.*` | Passed to [[prep]] for ticket creation; used by [[push]] for auto-resolve |
| `config.engineering_principles_paths` | Pre-flight path pattern matching before calling [[push]] |
| `config.engineering_principles_skill` | Slash-command for the engineering-principles pre-flight (absent/null → defaults to `/engineering-principles-python`; also `/engineering-principles-go` or `/engineering-principles-ts`). To skip engineering-principles entirely, set `engineering_principles_paths: null`. Added PR #47 |
| `config.wiki_path` | Gates Orient, wiki-ingest, and wiki-lint phases |

`NO_CONFIG` collapses the workflow: skip Orient, skip engineering-principles, skip wiki phases, skip Jira. The chain still runs: prep (worktree only) → code → push → review → merge.

## Scripts invoked

This command delegates to sub-commands, not scripts directly. See [[push]] and [[merge]] for the scripts they invoke (`push.sh`, `merge.sh`).

## Preconditions

- A `workflow.config.yaml` exists (or accept minimal/NO_CONFIG mode)
- [[prep]] preconditions apply for Phase 1
- `pr-review-toolkit@claude-plugins-official` installed for Phase 4 review
- `gh` on PATH and authenticated for Phase 4–6

## Chain position

This is the top-level orchestrator. All sub-commands remain callable standalone for edge cases — `/workflow` is the happy path, not the only path.

```
/workflow
  └─ /coderails:prep      (Phase 1)
  └─ /wiki-query          (Phase 2, if wiki)
  └─ [code/iterate loop]  (Phase 3, interactive)
  └─ /coderails:push      (Phase 4)
  └─ /pr-review-toolkit:review-pr  (Phase 4)
  └─ /coderails:merge     (Phase 6)
  └─ /wiki-ingest + /wiki-lint     (Phase 6, if wiki)
```

The [[agentic-loop]] skill sits *above* this command — it uses `/workflow` as a subroutine for each PR in a multi-PR autonomous session.

## Design notes

The command is **advisory, not enforcement**. Claude has to choose to invoke each phase. Mechanical enforcement (blocking `gh pr create` unless `/push` ran) belongs in `PreToolUse` hooks, not here. See [[enforcement-model]].

Phase 2b (design adversarial review) is distinct from Phase 4's `/pr-review-toolkit:review-pr`: Phase 2b reviews the *design page* before coding; Phase 4 reviews the *code* before merge. Both are required on non-trivial features. (verified: workflow.md:36)

## Design notes — PR #47 engineering_principles_skill

The `allowed-tools` frontmatter pre-authorises `/engineering-principles-go` and `/engineering-principles-ts` alongside `/engineering-principles-python` (added PR #47). This means projects configured with a non-python engineering-principles skill will not hit permission prompts during the workflow. (verified: `gh pr diff 47`) See [[pr_47_strictcode-skill-config]].

## See also

- [[prep]] — Phase 1 delegate
- [[push]] — Phase 4 delegate
- [[merge]] — Phase 6 delegate
- [[config-resolution]] — walk-up config resolver
- [[agentic-loop]] — uses /workflow as a subroutine
- [[enforcement-model]] — why commands are advisory, not enforcing
- [[enforce_pr_workflow]] — the PreToolUse hook that mechanically enforces the push→review→merge sequence
