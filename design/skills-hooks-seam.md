---
title: "Skills↔Hooks Seam Convention"
type: design
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_42_skills-hooks-seam.md]
tags: [design, hooks, skills, convention, cross-reference]
---

# Skills↔Hooks Seam Convention

The structural gap between skills (advisory) and hooks (mechanical) that existed until PR #42.

## The Problem

Skills instruct Claude to take actions. Hooks block certain actions unless preconditions are met. Before this convention was documented, those two halves were independently maintained:

- A skill could instruct `git merge` on main without mentioning that `enforce_pr_workflow.sh` would block it.
- A hook could gate a commonly-instructed action without any corresponding note in the skills that instruct it.

This produced a silent failure mode: Claude follows a skill's guidance, hits a hook block with no resolution context in the skill itself, and either stalls or improvises.

PR #40 introduced the `git merge` gate in `enforce_pr_workflow.sh`. The `finishing-a-development-branch` skill's "merge locally" Option 1 already instructed that action — but with no awareness of the gate. PR #42 fixed this gap and codified the obligation going forward. (verified — CLAUDE.md "Skills↔hooks seam convention" subsection)

## The Convention

Two obligations, one for each direction of the seam:

**Skill side:** When a skill instructs a hook-gated action, it must name the hook and state the resolution path. The reader should never need to discover the block by failing.

**Hook side:** When a hook is added or modified to gate a commonly-instructed action, the skills that instruct that action must be updated in the same PR.

This is documented in `CLAUDE.md` as a permanent working convention, not a one-off fix.

## Known hook-gated actions and their skill cross-references

| Gated action | Hook | Skills that must reference it |
|---|---|---|
| `gh pr create` | [[enforce_pr_workflow]] | [[wiki-ingest]], [[wiki-lint]], [[requesting-code-review]] |
| `gh pr merge` | [[enforce_pr_workflow]] | [[wiki-ingest]], [[wiki-lint]], [[finishing-a-development-branch]] |
| `git merge` on main/master | [[enforce_pr_workflow]] | [[finishing-a-development-branch]] |
| code-file edits on main/master | [[no_edit_on_main]] | (any skill instructing direct code edits) |

## Regex footgun: word boundaries and hyphens

PR #42's headline bug was `\bgit +merge\b` also matching `git merge-base`. The root cause: in POSIX ERE (used by `grep -E`), `-` is a word boundary. So `merge-base` has a word boundary between `merge` and `-`, making `\bmerge\b` match.

The safe pattern when guarding a command that has hyphenated variants is:

```
\bcommand([[:space:]]|$)
```

This requires the matched token to be followed by whitespace or end-of-line — excluding any hyphenated suffix. Applied at both Gate 3 (command classification) and the subcommand-detection block in `enforce_pr_workflow.sh`. (verified — hook source lines 28 and 37)

## See also

[[enforce_pr_workflow]] — the hook most directly governed by this convention  
[[finishing-a-development-branch]] — the skill whose Option 1 triggered the convention  
[[enforcement-model]] — the foundational hooks-vs-commands distinction  
`coderails/CLAUDE.md` — the "Skills↔hooks seam convention" subsection  
[[pr_42_skills-hooks-seam]] — the source record for this change
