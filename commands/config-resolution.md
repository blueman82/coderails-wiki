---
title: "Config Resolution: workflow.config.yaml"
type: design
created: 2026-05-30
last_updated: 2026-06-25
sources:
  - commands/workflow.md
  - commands/prep.md
  - commands/push.md
  - commands/init.md
tags: [commands, config, workflow, monorepo]
---

# Config Resolution: workflow.config.yaml

Every workflow command resolves `workflow.config.yaml` at runtime using a dual-path lookup. The same resolution pattern is used in three commands; the fourth writes the file using equivalent prose logic.

## The lookup pattern

Three commands — `workflow.md`, `prep.md`, and `push.md` — carry the identical inline `!`-bash substitution in their YAML frontmatter: (verified: workflow.md:9, prep.md:9, push.md:9)

```bash
GIT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null) && {
  cat "$GIT_ROOT/projects/$(basename $(pwd))/.claude/workflow.config.yaml" 2>/dev/null \
  || cat "$GIT_ROOT/.claude/workflow.config.yaml" 2>/dev/null \
  || echo "NO_CONFIG"
}
```

Priority:
1. **Monorepo layout**: `<git-root>/projects/<project-name>/.claude/workflow.config.yaml`
2. **Standalone repo**: `<git-root>/.claude/workflow.config.yaml`
3. **Sentinel**: `NO_CONFIG` — the string that signals the project has not been initialised

All three commands run in **minimal mode** when they see `NO_CONFIG` — they do NOT halt or prompt for `/coderails:init`. Minimal mode defaults: `config.jira` = null (skip all Jira steps), `config.wiki_path` = null (skip wiki phases), `config.worktree_base` = git root, `config.worktree_script` = null (use plain `git worktree add`), `config.strictcode_paths` = null (skip strictcode pre-flight). (verified: prep.md:11–16, workflow.md:11–17)

## workflow-init: the writer, not the reader

`workflow-init.md` does not contain the bash one-liner. It expresses the same monorepo-vs-standalone decision as prose steps and *writes* the config file rather than reading it. (verified: workflow-init.md:13–17)

The scaffolder checks whether `<git-root>/projects/<project-name>/` exists to decide which path to write to — the same branch the readers follow.

## Update discipline: all four files

The CLAUDE.md for this repo states explicitly: "If you add a config field, update **all four** of `workflow.md`, `prep.md`, `push.md`, and `workflow-init.md`." (verified: CLAUDE.md — project instructions)

This is required because:
- The three reader commands each embed the bash one-liner independently — they share no include mechanism.
- `workflow-init.md` scaffolds the YAML template and must include any new field so it appears in freshly created configs.

## `NO_CONFIG` as a sentinel

`NO_CONFIG` is the literal string echoed when neither config path resolves. Commands check for it by string match and stop early. This lets a command fail fast in a misconfigured project rather than silently skipping features. (inferred: consistent usage across workflow.md:11, prep.md:11, push.md behaviour)

## Related

- [[enforcement-model]]
- [[workflow-command-chain]]
- [[workflow-init]]
