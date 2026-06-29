---
title: "Config Resolution: workflow.config.yaml"
type: design
created: 2026-05-30
last_updated: 2026-06-29
sources:
  - commands/workflow.md
  - commands/prep.md
  - commands/push.md
  - commands/init.md
  - sources/pr_72_config-walkup-symlink-hang.md
tags: [commands, config, workflow, monorepo, config.sh, walk-up]
---

# Config Resolution: workflow.config.yaml

Every workflow command — and the [[enforce_pr_workflow]] hook's opt-in check — resolves `workflow.config.yaml` at runtime through a **shared resolver**, `scripts/lib/config.sh`. `init.md` writes the file using equivalent prose logic.

## The lookup pattern (walk-up resolver)

> ⚠️ This page previously documented an inline dual-path `projects/<name>/` bash
> one-liner. That was replaced by PR #67 (directory walk-up) and extracted into a
> shared library by PR #71. The old hardcoded `projects/` layout is gone.

`scripts/lib/config.sh::coderails::config_path()` walks **up** from a start dir to the git root and returns the first `.claude/workflow.config.yaml` found; nearest wins (replacement, not inheritance). This is **layout-agnostic** — standalone repos, classic `projects/<name>/` monorepos, and arbitrary layouts (`apps/web`, `services/api`, …) all resolve from any subdir. (verified: config.sh, PR #67/#71)

`coderails::resolve_config()` wraps it: emit the file contents, or the literal `NO_CONFIG` sentinel if none found.

Three consumers source this one resolver so their answers agree:
- `commands/{workflow,prep,push}.md` frontmatter (via `${CLAUDE_PLUGIN_ROOT}/scripts/lib/config.sh`)
- `scripts/merge.sh`
- `hooks/scripts/enforce_pr_workflow.sh` — **critically**, the merge gate's "is enforcement active?" must use the *same* resolver as the commands, or the gate and the commands would disagree on whether a project is initialised. (verified: config.sh header)

### The symlink hang (PR #72) — a fixed invariant worth remembering

The walk-up terminated only on `d == git_root` *string equality*. `git rev-parse` returns a **symlink-resolved** path (macOS `/tmp` → `/private/tmp`), so a start dir in the unresolved namespace never matched, `dirname` bottomed out at `/`, and the loop spun forever — but **only on the NO_CONFIG path** (a found config short-circuits first). Fix: canonicalise `start` with `pwd -P` + a hard `/` floor. The resolver is now guard-script-safe (no `set -euo pipefail`, always `return 0`, empty-on-miss) AND termination-safe. See [[pr_72_config-walkup-symlink-hang]]. (verified: config.sh:39,45)

All consumers run in **minimal mode** on `NO_CONFIG` — they do NOT halt or prompt for `/coderails:init`. Minimal mode defaults: `config.jira` = null (skip all Jira steps), `config.wiki_path` = null (skip wiki phases), `config.worktree_base` = git root, `config.worktree_script` = null (use plain `git worktree add`), `config.engineering_principles_paths` = null (skip engineering-principles pre-flight). (verified: prep.md, workflow.md)

## init: the writer, not the reader

`init.md` does not contain the bash one-liner. It expresses the same monorepo-vs-standalone decision as prose steps and *writes* the config file rather than reading it. (inferred: consistent with CLAUDE.md statement about the scaffolder)

The scaffolder checks whether `<git-root>/projects/<project-name>/` exists to decide which path to write to — the same branch the readers follow.

## Update discipline: all four files

The CLAUDE.md for this repo states explicitly: "If you add a config field, update **all four** of `workflow.md`, `prep.md`, `push.md`, and `init.md`." (verified: CLAUDE.md — project instructions)

This is required because:
- The reader commands and the hook now share **one resolver** (`scripts/lib/config.sh`), so *resolution* logic lives in a single place — but each command still independently *reads* the fields it needs and applies its own minimal-mode defaults, so a new field must be wired into each consumer that uses it.
- `init.md` scaffolds the YAML template and must include any new field so it appears in freshly created configs.

(Historical note: before PR #71 the three reader commands each embedded an identical inline bash one-liner with no include mechanism — the "update all four" rule originally guarded against those copies drifting. PR #71 collapsed the *resolution* duplication into `config.sh`; the field-wiring discipline remains.)

## Config fields

Known fields in `workflow.config.yaml` (as of 2026-06-26):

| Field | Type | Default / NO_CONFIG | Added |
|---|---|---|---|
| `project` | string | — | original |
| `wiki_path` | string \| null | null | original |
| `worktree_base` | string | git root | original |
| `worktree_script` | string \| null | null | original |
| `jira.*` | object \| null | null | original |
| `engineering_principles_paths` | list \| null | null | original |
| `engineering_principles_skill` | string \| null | `/engineering-principles-python` | PR #47 |

`engineering_principles_skill` is the slash-command used for the pre-flight engineering-principles check in [[push]] and [[workflow]]. Auto-detected during [[init]] setup: `go.mod` → `/engineering-principles-go`; `package.json` + `.ts` files → `/engineering-principles-ts`; otherwise `/engineering-principles-python`. Absent or null `engineering_principles_skill` defaults to `/engineering-principles-python` at runtime — it does NOT disable engineering-principles. To disable engineering-principles entirely, set `engineering_principles_paths: null`. Note: `init.md`'s inline YAML comment `# nil = skip engineering-principles entirely` is misleading versus the executing logic in `push.md` and `workflow.md`. See [[pr_47_strictcode-skill-config]].

## `NO_CONFIG` as a sentinel

`NO_CONFIG` is the literal string echoed when neither config path resolves. Commands check for it by string match and enter minimal mode — they do not stop. This lets commands degrade gracefully in an uninitialised project while still providing core worktree + push functionality. (verified: workflow.md:11–17, prep.md:11–16)

## Related

- [[enforcement-model]]
- [[workflow]]
- [[init]]
