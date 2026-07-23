---
title: "PR #285 — document the wiki_taxonomy_gate.sh hook in AGENTS.md (supersedes stranded #276)"
type: source
origin: "coderails PR #285 (merged 2bd1bce, 2026-07-23; supersedes closed #276)"
created: 2026-07-23
last_updated: 2026-07-23
sources: []
tags: [source, hooks, wiki-taxonomy, AGENTS.md, docs, tier-review, stranded-pr]
---

# PR #285 — document wiki_taxonomy_gate.sh

A one-line docs change: adds the `wiki_taxonomy_gate.sh` row to AGENTS.md's Hook Activation
Matrix. The hook itself was introduced by **PR #257** ("block writes to unapproved wiki
folders", merged 2026-07-22, `71a2e45`) — it was live and registered but documented nowhere
in AGENTS.md. #285 closes that doc gap.

## What the hook does (as now documented, verified against the code)

`PreToolUse (Write|Edit|MultiEdit)` — **blocks** a write into an unsanctioned top-level
directory of an LLM wiki vault:

- The sanctioned directory list is **parsed from the vault's own `AGENTS.md` "Page types"
  table**, never hardcoded — so editing that table changes enforcement with no hook edit.
- A vault is identified by the **file's own repo root** (not session cwd) carrying a literal
  `wiki-vault: true` marker, a parseable Page types table, AND ≥2 of the parsed sanctioned
  dirs actually existing on disk — so a code repo that merely *documents* a wiki's taxonomy
  from outside is never mistaken for the vault.
- **Fails open** on any ambiguity (marker absent, table unparseable, <2 dirs present).
- Always-allowed regardless of the table: `raw/`, **any file directly at the vault root** (a
  *structural* rule — no directory component — not a fixed name list; `index.md`/`log.md`/
  `AGENTS.md`/`README.md` are examples), and dotfile dirs (`.git/`, `.obsidian/`, `.claude/`).

## The vault-root doc-precision fix (review caught it)

The first cut of the row listed the four vault-root files as if the allowance were a
name-based allowlist. Review checked the row against the hook (`wiki_taxonomy_gate.sh:123-127`)
and found the actual rule is **structural** — any root-level file, any name, passes. The row
was corrected to say so, with the four files as examples. A doc PR's own review catching a doc
inaccuracy, and the fix landing before merge.

## Why a fresh PR, not #276

#276 added this identical line but stranded on two things, both instructive:

1. **Stale-branch diff illusion.** Its branch fell 29 commits behind main, so its diff *against
   current main* showed 24 files / −1359 lines — looking like it reverted a day of merged
   enforcement work. Against its *own base* it was +1 line. A reminder that a PR's apparent
   diff is meaningless without knowing its base; check `merge-base HEAD origin/main`.
2. **Wrong tier claim.** It claimed **tier 0**, which the tier-review daemon
   ([[sources/pr_232_tier-review-gate]]) correctly ruled `illegitimate` — a newly-documented
   hook has no pre-existing test/verify-criterion, the tier-0 requirement. The honest tier is 1.

#285 re-did the change on a **fresh branch off current main** (a plain non-force push, avoiding
the force-push a rebase of a 29-behind branch would need — the destructive-command guardrail
blocks that) and claimed tier 1. Same fresh-branch-not-force-push method as
[[sources/pr_284_check10_hardening]].

## Provenance

#276 was authored by a separate overnight Claude session running the [[agentic-loop]] skill;
it got the PR through eval and review artifacts but could not clear the tier-review verdict.
Traced via the transcript-find skill on the branch name, then closed out here.
