---
title: Install and Cache Trap
type: design
created: 2026-05-30
last_updated: 2026-07-22
sources:
  - design/agentic-loop-path-keying.md
  - .claude-plugin/plugin.json
  - ~/.claude/settings.json
  - ~/.claude/plugins/known_marketplaces.json
  - ~/.claude/plugins/cache/coderails/coderails/
  - CLAUDE.md
  - sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md
  - sources/pr_11-14_gate-hardening-followups.md
  - sources/pr_86-107_2026-07-08_loop-lib-residuals.md
  - sources/pr_159_retire-catchup-add-telemetry.md
tags:
  - installation
  - cache
  - plugin
  - trap
  - nested-worktree
  - operational
  - exec-bit
  - test-sandboxing
  - home-sandbox
---

# Install and Cache Trap

The structural gap between editing the coderails repo and running coderails. This is the most operationally dangerous thing to get wrong.

## Three Locations

A coderails plugin lives in three distinct places at runtime (verified — confirmed via `ls`, `grep`, and `diff` between all three):

**1. The repo working copy**
`/Users/harrison/Documents/Github/coderails/`

This is what you edit. Every `Edit` tool call lands here. This is the source of truth for development.

**2. The marketplace registration**
Two files record where the plugin's marketplace is and where it's installed:

- `~/.claude/settings.json` lines 68–74 (verified): `extraKnownMarketplaces.coderails.source.path` points to the repo.
- `~/.claude/plugins/known_marketplaces.json` lines 10–15 (verified): `source.path` and `installLocation` both point to the repo.
- `~/.claude/settings.json` line 66 (verified): `"coderails@coderails": true` — the plugin is enabled.

These registrations tell Claude Code where to find the marketplace and that coderails is installed. They do not tell it what the plugin *contains*. That comes from location 3.

**3. The runtime cache**
`~/.claude/plugins/cache/coderails/coderails/<version>/`

This is what the Claude Code harness actually runs. It is a snapshot populated when the plugin was last installed or reinstalled. The directory structure mirrors the repo: `commands/`, `hooks/`, `skills/`, `scripts/`, etc. (verified — `ls` output).

**Correction (2026-07-13):** this page previously claimed the version was permanently pinned at `1.0.0` with no semver increments. That was true when originally written but is no longer the state of the repo — `plugin.json` and `marketplace.json` now carry real semver bumps (verified current: both read `"version": "1.1.5"`, PR #159 repaired a lockstep break between them), and `~/.claude/plugins/cache/coderails/coderails/` on this machine has accumulated separate versioned directories (`1.0.0`, `1.1.0`, `1.1.1`, `1.1.2`, `1.1.3`, `1.1.5` observed) rather than a single static `1.0.0` folder that gets overwritten in place. `claude plugin update` works against these versions. The core trap this page documents — editing the repo does not change what the harness runs until the cache is resynced — is unaffected by this correction; only the "version never changes" claim was wrong.

## The Trap

**Editing the repo does not change what the harness runs.**

When you edit `commands/workflow.md` in the repo, the harness continues to load and execute `~/.claude/plugins/cache/coderails/coderails/1.0.0/commands/workflow.md`. These are separate files. They can drift apart.

The current state (verified — `diff` confirmed): files are identical because the cache was populated recently. But the identity is coincidental, not guaranteed.

## What Syncs the Cache (and What Doesn't)

| Action | Updates cache? |
|---|---|
| Edit a file in the repo | No |
| `/reload-plugins` | No — reloads plugin metadata, does not re-copy files from repo to cache |
| `/plugin install coderails` | Yes — repopulates cache from repo |
| `install.sh` re-run | No — install.sh sets up CLAUDE.md and settings.json but does not touch the plugin cache |

The CLAUDE.md "Working in this repo" section (line 124–126) says `/reload-plugins` is sufficient after editing a command or hook. That claim is about the harness picking up the *already-cached* files freshly — it does not mean the cache is re-populated from the repo. If the cache file and the repo file have diverged, `/reload-plugins` will load the stale cache copy. (inferred from cache structure; the authoritative test is to make a distinct change in the repo and observe what the harness actually executes.)

## Practical Implication

After making a change to any hook, command, or skill:

1. **Check**: is the cache copy identical to the repo copy?
   ```bash
   diff /Users/harrison/Documents/Github/coderails/commands/workflow.md \
        ~/.claude/plugins/cache/coderails/coderails/1.0.0/commands/workflow.md
   ```
2. **If they differ**: run `/plugin install coderails` to repopulate the cache, then `/reload-plugins`.
3. **Verify**: re-run the diff to confirm the cache now matches.

The hooks that enforce the discipline loop live at `hooks/scripts/*.sh` in the repo — but the harness runs the cache copies. A hook fix that never made it into the cache is not a deployed fix.

## Why This Pattern Exists

Claude Code's plugin architecture separates the marketplace source (which can be a local directory or a GitHub repo) from the installed cache. This allows version-pinning, rollback, and offline operation. The cache is only refreshed when the harness is explicitly asked to reinstall or update.

**Superseded claim (see the Correction note above, 2026-07-13):** this section previously said the version for a local-directory plugin like coderails was permanently `1.0.0` with no semantic versioning increments on save. That is not the current state — `plugin.json`/`marketplace.json` are bumped deliberately as part of PRs (most recently to `1.1.5`, PR #159's lockstep repair), and `claude plugin update` picks up new versions. The trap this page is actually about — repo edits not reaching the harness until the cache resyncs — does not depend on whether the version string changes; it was simply illustrated with a version that, at the time, never changed.

## Bash 3.2 portability

`install.sh` must run on **bash 3.2** — macOS ships that as `/bin/bash`, and the
`#!/usr/bin/env bash` shebang picks up whatever is first on PATH (3.2 on a stock
Mac). No bash-4 syntax: no `${var,,}`/`${var^^}` case modification, no associative
arrays, no `mapfile`. A `${answer,,}` slipped in and broke the overwrite prompt on a
3.2 machine with "bad substitution"; fixed by `tr` lowercase.
See [[install-bash32-bad-substitution_2026-06-01]]. (verified: 2026-06-01)

## Exec-bit sweep is now git-index-mode-aware (PR #96)

Separate from the cache trap above, `install.sh`'s "ARMING SCRIPTS" sweep (a
distinct mechanism — it sets file *modes* at install time, unrelated to
repo-vs-cache file *content* sync) used to unconditionally `chmod +x` every
swept script, fighting the git-index-mode invariant [[pr_92_exec-bit-sweep|PR
#92]]/#94 established (some libs are deliberately `100644`/sourced-only). As of
[[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter|PR
#96]] (merged 2026-07-03), the sweep reads each file's git index mode and
branches: `100755`→ensure `+x`, `100644`→ensure `-x` (new), not-in-index→legacy
unconditional `+x`. A fix round closed a Critical the mode-read introduced: the
`git ls-files | awk` pipe under `set -euo pipefail` died with exit 128 on
non-git checkouts (release tarballs), silently skipping everything after the
sweep — fixed with `|| _index_mode=""` falling through to the existing fallback
branch.

## install_mode_sweep.test.sh must sandbox $HOME, not just MEMORY_TARGET (PR #12)

A real, recurring corruption bug found and fixed during [[pr_11-14_gate-hardening-followups|PR #12]] (`fdd29fd`, 2026-07-06), distinct from the exec-bit-sweep logic above but in the same test file (`hooks/scripts/tests/install_mode_sweep.test.sh`). The test invokes the real `install.sh` non-interactively to verify the exec-bit sweep, and had been redirecting `MEMORY_TARGET` to keep memory-seeding writes inside the temp tree — but `install.sh` **also writes unconditionally under `$HOME`**: the `installed_plugins.json` scan, the `~/.claude/commands` conflict scan, `settings.json`/`known_marketplaces.json` marketplace registration, and the `~/.claude/CLAUDE.md` discipline-rules append. None of that is redirected by `MEMORY_TARGET` alone.

Because this repo's own `test_gate` `PreToolUse` hook runs the full test suite on every `git commit`, every worker commit made inside a worktree carrying the unfixed test rewrote the **real developer machine's actual `~/.claude/settings.json`** — 10 recorded events during the loop that produced PRs #11–14, all auto-repaired, but a genuine and repeatable corruption, not a one-off. Telling workers not to touch that file was unenforceable, because the corruption happened inside a test the hook itself ran, outside worker awareness — the only real fix was sandboxing the test's own environment.

Fix: `HOME` is redirected to a freshly-`mktemp`'d sandbox directory for the duration of both `install.sh` invocations the test makes (the git-tracked-tree run and the no-git-tree run), with before/after `cksum` guards on the real `settings.json`, `known_marketplaces.json`, and `CLAUDE.md` proving they're byte-identical across the test run. The sandbox is pre-seeded with a stale marketplace key so the test still exercises `install.sh`'s real `jq` mutation logic against the sandbox rather than merely proving the sandbox inert.

**General lesson**: a test that shells out to a real installer/setup script needs every environment variable that script writes through sandboxed, not just the first one identified — `MEMORY_TARGET` alone looked sufficient until `$HOME` itself turned out to be the actual leak path.

## Nested-worktree concern investigated, did not reproduce ([[pr_86-107_2026-07-08_loop-lib-residuals|PR #87]], 2026-07-08)

A reported residual claimed `install_mode_sweep.test.sh` might fail when run from inside a worktree checked out under `.claude/worktrees/`, on the theory that git index resolution (`git ls-files -s`, which the sweep depends on) could behave differently when nested. Investigated both the plain nested-worktree case and the stronger "worktree of a worktree" reading — **neither reproduces**: `git` has no "worktree of a worktree" special case in its data model, so even a worktree created *from* a nested worktree still anchors its `.git` file at the original repo clone's administrative area, and `git ls-files -s` resolves index modes correctly from any point in such a chain. Full suite (35/35) and the test itself (27/27 checks) both passed from inside a nested worktree. Outcome: documented as a comment in the test's own header (the invariant that would need to break for the concern to resurface), not a code fix — zero executable lines touched.

## The marketplace-consumer analogue (assistant-agent, sub-project 4)

Everything above is about coderails' *own* local-directory dev loop (repo vs. cache within this one machine's coderails checkout). A distinct but related gap exists for any **downstream consumer** that installs coderails via the marketplace rather than developing it directly: assistant-agent's live hook-fire probe (2026-07-06) found its installed plugin cache stuck at version `1.0.0` (`autoUpdates: false` at the time) for roughly two weeks, missing `no_edit_on_main.sh`/`enforce_pr_workflow.sh`/loop-guard hooks that source had long since gained. Unlike the trap documented above, there was no local edit to diff against — the consumer has no visibility into how stale its cached install is relative to the published source, short of an explicit version check. The plugin self-updated overnight once `autoUpdates` allowed it, closing the gap silently. See [[assistant-link-send-gate-architecture]] for the full finding and the dynamic-path-resolution fix this prompted in assistant-agent's own test suite.

The same update also changed [[agentic-loop-path-keying]]'s slug scheme underneath any session still on the old cache (1.0.0 predates that keying change entirely) — a second, undocumented-until-now migration gap this one auto-update triggered. See that page's caveats section for the mechanism and workaround.

## The same trap on a FOREIGN plugin (PR #259, 2026-07-22)

Everything above concerns coderails' own cache. The identical mechanism bites
when the perishable file belongs to **another maintainer's** plugin. The
2026-07-17 token-burn effort's row 5 patched the **remember** plugin's
`session-start-hook.sh` inside `~/.claude/plugins/cache/<marketplace>/remember/<version>/scripts/`
— a version-pinned directory, so a bump (`remember/0.8.3` → `0.9.0`) installs a
fresh unpatched copy and the cap silently disappears. Unlike coderails' own
trap there is no repo copy to diff against: the patch existed *only* in the
cache, in no repo at all, and was nearly lost.

[[remember_inject_cap_guard]] closes the detection half (a `SessionStart` hook
that notices the cap's absence) and [[plugin-boundary-writes]] governs what it
may do about it — warn-only by default, writing only on explicit opt-in, because
rewriting someone else's package unasked is a different act from resyncing your
own cache. (verified — [[pr_259_remember-inject-cap-guard]])

## Cross-References

- [[enforcement-model]] — hooks only enforce what the cache contains
- [[plugin-boundary-writes]] — what coderails may do when the perishable cached file belongs to *another* plugin
- [[remember_inject_cap_guard]] — the hook that detects a foreign plugin's wiped patch
- [[pr_259_remember-inject-cap-guard]] — PR #259: the foreign-cache case and the row-5 recovery
- [[discipline-loop]] — if a hook fix is in the repo but not the cache, the discipline enforcement is running the old version
- [[install-bash32-bad-substitution_2026-06-01]] — the bash 3.2 syntax bug and fix
- [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] — PR #96: the exec-bit sweep becomes git-index-mode-aware
- [[pr_92_exec-bit-sweep]] — the git-index-mode invariant PR #96's sweep now respects
- [[pr_11-14_gate-hardening-followups]] — PR #12: HOME-sandboxed install_mode_sweep.test.sh, closing a real recurring corruption of the developer's actual ~/.claude/settings.json
- [[assistant-link-send-gate-architecture]] — the marketplace-consumer analogue of this trap, found in assistant-agent's installed plugin cache (sub-project 4)
- [[agentic-loop-path-keying]] — the loop-state re-keying migration gap the same plugin update triggered
- [[pr_86-107_2026-07-08_loop-lib-residuals]] — PR #87: the nested-worktree investigation that did not reproduce, documented in `install_mode_sweep.test.sh`'s header
- [[pr_159_retire-catchup-add-telemetry]] — PR #159: the version-lockstep repair (1.1.5) that prompted this page's version-pinning correction
