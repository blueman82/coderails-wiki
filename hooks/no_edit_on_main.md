---
title: "no_edit_on_main.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-29
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_44_no-edit-plugin-source.md, sources/pr_57-62_subagent-enforcement-gate-hardening.md, sources/pr_69_no-edit-message-worktree.md, sources/pr_70_gate-settings-json-edits.md, sources/pr_76_harden-hook-stdin-read.md]
tags: [hook, PreToolUse, enforcement, main-branch-protection, settings-json]
---

# no_edit_on_main.sh

PreToolUse hook that blocks edits (Write/Edit/MultiEdit) to source files on `main` or `master` branches, enforcing the no-direct-edits-on-main invariant. As of PR #60, the code arm uses an **allowlist** (everything not explicitly allowed is blocked) rather than a blocklist of specific extensions.

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse (Write \| Edit \| MultiEdit)` |
| Mode | **block** (permissionDecision: deny) |
| Timeout | (default) |

## Logic summary

The hook has three arms checked in order.

**Permission-file arm** (checked first, PR #70): blocks `.claude/settings.json` and `.claude/settings.local.json` on **any** branch, in any repo, regardless of the plugin marker. Matched on the `.claude/` parent (`case` glob `*/.claude/settings.json|.claude/settings.json|...`) so an unrelated `settings.json` elsewhere passes. This is the **only any-branch arm** — it fires before the allowlist (these are `.json`, otherwise allowlisted) and before the branch check. Rationale: the `permissions.allow` rules in these files pre-approve commands upstream of every PreToolUse gate, so editing them is the one move that can dismantle the discipline layer. See [[pr_70_gate-settings-json-edits]] and [[enforcement-model]]. (verified — PR #70)

**Plugin-source markdown arm** (checked second): blocks `skills/*/SKILL.md` and `commands/*.md` unconditionally on main/master in a plugin repo. Path arms are `/`-anchored to prevent lookalike directories in other repos (e.g. the wiki's `commands/` pages). Cross-repo correctness: the file's repo must carry `.claude-plugin/plugin.json` to be gated. (PR #44)

**Code arm** (allowlist model, PR #60): on main/master, blocks edits to everything **except** an explicit set:

| Allowed | Extensions / filenames |
|---|---|
| Docs | `.md`, `.txt`, `.rst` |
| Config | `.yaml`, `.yml`, `.json`, `.toml`, `.ini`, `.cfg` |
| Special dotfiles | Literal `.gitignore` basename only (not `deploy.gitignore`) — tightened PR #62 |
| Bare filenames | `LICENSE` |

Everything else — `.sh`, `.py`, `.ts`, `.go`, `.rb`, etc. — is blocked on main. Previously (original + PR #44) only 6 specific code extensions were blocked; all others passed. The allowlist inversion makes the default **deny** rather than allow for unrecognised types.

Branch resolution: both arms key off the **file's own repo**, not the session cwd. A relative path is resolved via the cwd only to make it absolute; the branch is then read from the file's own git repo. This is the cross-repo correctness established in PR #52.

(verified — PR #60, no_edit_on_main.sh)

## Block condition

Tool is Write, Edit, or MultiEdit AND either:
- the file is `.claude/settings.json` / `.claude/settings.local.json` (matched on the `.claude/` parent) — blocked on **any** branch, in any repo (PR #70); **OR**
- the file's git repo is on `main`/`master` AND:
  - the file is plugin source (`skills/*/SKILL.md`, `commands/*.md`) and the repo carries `.claude-plugin/plugin.json`; **OR**
  - the file is NOT in the allowlist (doc/config/special extensions listed above).

The path arms are anchored on a `/` boundary (`*/skills/*/SKILL.md|skills/*/SKILL.md`) so a stray directory like `myskills/` can't match, with a bare relative arm for a path passed without a leading directory. (verified — PR #44, `case`-statement glob)

## Log output

Appends to `$CLAUDE_DISCIPLINE_LOG` on block. Format matches the `key=value` convention used by other hooks.

## Environment variables

- `CLAUDE_DISCIPLINE_LOG` — path to the shared discipline log (default `~/.claude/discipline.log`)

## Design note

Uses `permissionDecision: deny` (JSON output to stdout), mirroring `destructive_bash_gate.sh`. This is the PreToolUse block mechanism — distinct from the `exit 2` used by Stop hooks. See [[hook-exit-codes]].

**Why plugin-source markdown is gated (PR #44).** `skills/*/SKILL.md` and `commands/*.md` ARE the plugin's source — editing `agentic-loop/SKILL.md` directly on main is the same class of mistake as committing a `.py` straight to main. The original "docs-only carve-out" let all markdown through, which had let a SKILL.md edit land direct on main earlier the same session. The carve-out is now narrowed to *plain* docs (root `README.md`, `docs/`, non-`SKILL.md` skill references).

**Why the allowlist inversion (PR #60).** The original blocklist had no principled boundary — `.sh` scripts, `.rb` files, `.rs`, etc. all slipped through because the original author only enumerated the languages they used at the time. An allowlist (doc/config/bare dotfiles) has a principled "these things are safe to edit on main" rationale; everything else is source. A settings.json `Write`/`Edit` permission rule covers any legitimate override.

**`.gitignore` basename tightening (PR #62, TDD-driven).** The original arm was `*.gitignore` which would have allowed `deploy.gitignore` to be edited on main. Fixed to a basename match (`case "$basename" in .gitignore|LICENSE)`). The bug was found by writing failing tests first (deploy.gitignore → DENY, src/.gitignore → ALLOW), then fixing the hook.

**Why the block message recommends worktree + branch (PR #69).** The deny `reason` string
steers the blocked user toward the resolution path, so its wording is part of the hook's UX.
It now leads with "create an isolated worktree + branch first" (`/coderails:prep` or
`git worktree add <path> -b <name>`) rather than the older "create a feature branch first".
The reword matches the isolation discipline [[prep]] enforces — a bare `git checkout -b` in
place leaves you in the same working tree, whereas a worktree + branch genuinely isolates the
work. The message also names the `settings.json` Write/Edit permission rule as the one-line-hotfix
escape. This is message text only; the block conditions above are unchanged. (decision — PR #69)

**Why the permission-file arm is any-branch, and why it sits first (PR #70).** The escape every gate documents is "add a `Bash` permission rule to `settings.json`." That rule pre-approves the command *upstream* of the PreToolUse hooks, so an allowlisted command skips [[enforce_pr_workflow]] entirely — making the permission file the one edit that can dismantle the discipline layer. An agent could previously write that escape itself via the edit tools; this arm closes that path. It is **any-branch** because a settings escape is dangerous on a feature branch too — the only [[no_edit_on_main]] arm not gated on main. It sits **first** because `.json` is in the allowlist, so a later check would let it through. This is friction, not a lock (see [[enforcement-model]] ceiling): it does **not** cover Bash-path edits (`sed -i`, `>`, `tee` → `.claude/settings*.json`) — a named-but-unbuilt follow-up for [[destructive_bash_gate]]'s in-Bash source-edit arm — and cannot stop an agent disabling the plugin. The true lock is server-side GitHub branch protection. (decision — PR #70)

**Why `git push` is deliberately NOT gated.** Edit-time (this hook) is the correct seam for the direct-to-main concern. Gating `git push` would be (a) redundant — the edit is already blocked here, and GitHub branch protection covers the server side; (b) breaking — the PR workflow *requires* pushing feature branches (`push.sh`); (c) brittle — "a push targeting main" hides behind implicit upstream, `HEAD`, and refspecs, with no clean token to match (unlike `gh pr create` / `git merge` in [[enforce_pr_workflow]]). Strengthening this edit-time gate is what makes a push-time gate unnecessary. (decision — PR #44 discussion)

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention.

## See also

[[enforcement-model]] — the hook/command distinction  
[[destructive_bash_gate]] — mirrors the same deny mechanism  
[[discipline-loop]] — the broader discipline hook composition  
[[enforce_pr_workflow]] — the companion PR-workflow enforcement hook  
`coderails/hooks/scripts/no_edit_on_main.sh`  
`coderails/hooks/tests/no_edit_on_main.test.sh`
