---
title: "PR #42 — fix(hooks): exclude git merge-base from gate; reorder hint; skills↔hooks cross-refs"
type: source
created: 2026-06-25
last_updated: 2026-06-25
sources: []
tags: [source, hooks, skills, enforce_pr_workflow, bug-fix, seam-convention]
---

# PR #42 — fix(hooks): exclude git merge-base from gate; reorder hint; skills↔hooks cross-refs

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #42 |
| Branch | `chore/skills-hooks-seam` |
| Merged | 2026-06-25 |
| Merge SHA | `fbe24f1` |
| JIRA ticket | — |

## Summary

Four changes addressing the seam between skills and hooks. The headline bug (F4): PR #40's `git merge` gate regex `\bgit +merge\b` also matched `git merge-base` because `-` is a word boundary character, causing read-only ancestor-lookup commands to be wrongly blocked. Fixed with `\bgit +merge([[:space:]]|$)`. The three companion changes (F1–F3) document the skills↔hooks relationship that was previously undocumented.

## Files changed

- `hooks/scripts/enforce_pr_workflow.sh` — regex fix at Gate 3 and subcommand-detection (F4); reordered git-merge block-message hint (F2)
- `skills/finishing-a-development-branch/SKILL.md` — added note about enforce_pr_workflow gate in Option 1 (F1)
- `CLAUDE.md` — new "Skills↔hooks seam convention" subsection; accurate hook-awareness notes for wiki-ingest and wiki-lint (F3)

## Changes in detail

### F4 — merge-base exclusion (bug fix, headline)

**Before:** `\bgit +merge\b` at Gate 3 and subcommand-detection

**After:** `\bgit +merge([[:space:]]|$)` at both locations

**Why:** In POSIX ERE used by `grep -E`, `-` is a word boundary when adjacent to a non-word character. So `git merge-base` had a word boundary between `merge` and `-base`, and `\bgit +merge\b` matched it. The fix requires the token after "merge" to be whitespace or end-of-line, which excludes `merge-base`, `merge-file`, and `merge-tree` (all read-only plumbing commands). New test Case 14 asserts `git merge-base HEAD main` on main → allow. (verified — hook source lines 28 and 37)

### F2 — reordered git-merge block-message hint

The block message for `git merge` on main now leads with the actual resolution path ("Run /pr-review-toolkit:review-pr first") before listing the `/coderails:merge` alternative and the settings.json bypass. This matches the ordering used by the adjacent `gh pr merge` hint and reduces the chance a reader skips past the right answer.

### F1 — finishing-a-development-branch: Option 1 hook-awareness

The skill's "merge locally" option gained a note that `enforce_pr_workflow.sh` blocks `git merge` on main/master (in a configured repo) unless `/pr-review-toolkit:review-pr` ran, with the resolution step. Previously the skill instructed an action the hook would silently block without explanation in the skill itself.

### F3 — skills↔hooks seam convention in CLAUDE.md

New subsection documents the cross-reference obligation in both directions:
- When a skill instructs a hook-gated action, the skill must name the hook and the resolution path.
- When a hook gates a commonly-instructed action, the affected skills must be updated.

Accurate hook-awareness notes added to wiki-ingest and wiki-lint (both instruct `gh pr create`/`gh pr merge` steps). `subagent-driven-development` was deliberately NOT touched — it uses only read-only `git merge-base`, which is not gated.

## Durable / non-obvious points

1. **Regex word-boundary footgun:** In POSIX ERE, `-` (hyphen) is a word boundary. `\bfoo\b` matches inside `foo-bar` because the boundary fires between `foo` and `-`. The safe pattern for "the word `merge` not followed by a hyphenated suffix" is `merge([[:space:]]|$)`, not `\bmerge\b`. This is the underlying cause of the merge-base false positive and the pattern to use for any similar command-prefix guard.

2. **Skills and hooks were independent halves:** Before PR #42, skills instructed actions without knowing what hooks guarded them, and hooks blocked actions without signalling back to skills. The new convention in CLAUDE.md makes this a maintained cross-reference obligation rather than an accidental gap.

## Wiki pages updated

- [[enforce_pr_workflow]] — regex before/after, the word-boundary footgun, reordered hint
- [[finishing-a-development-branch]] — Option 1 hook-awareness note
- [[skills-hooks-seam]] — new design page for the cross-reference convention
