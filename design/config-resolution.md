---
title: "Config Resolution: workflow.config.yaml"
type: design
created: 2026-05-30
last_updated: 2026-06-26
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

All three commands run in **minimal mode** when they see `NO_CONFIG` — they do NOT halt or prompt for `/coderails:init`. Minimal mode defaults: `config.jira` = null (skip all Jira steps), `config.wiki_path` = null (skip wiki phases), `config.worktree_base` = git root, `config.worktree_script` = null (use plain `git worktree add`), `config.engineering_principles_paths` = null (skip engineering-principles pre-flight). (verified: prep.md:11–16, workflow.md:11–17)

## init: the writer, not the reader

`init.md` does not contain the bash one-liner. It expresses the same monorepo-vs-standalone decision as prose steps and *writes* the config file rather than reading it. (inferred: consistent with CLAUDE.md statement about the scaffolder)

The scaffolder checks whether `<git-root>/projects/<project-name>/` exists to decide which path to write to — the same branch the readers follow.

## Update discipline: all four files

The CLAUDE.md for this repo states explicitly: "If you add a config field, update **all four** of `workflow.md`, `prep.md`, `push.md`, and `init.md`." (verified: CLAUDE.md — project instructions)

This is required because:
- The three reader commands each embed the bash one-liner independently — they share no include mechanism.
- `workflow-init.md` scaffolds the YAML template and must include any new field so it appears in freshly created configs.

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
