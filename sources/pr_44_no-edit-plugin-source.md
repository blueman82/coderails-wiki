---
title: "PR #44 — no_edit_on_main gates plugin-source markdown; remove docs/superpowers tree"
type: source
origin: "PR #44 (merged 2026-06-25, squash 8179b31)"
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, PR, hook, no_edit_on_main, main-branch-protection, cleanup]
---

# PR #44 — plugin-source gate + historical-doc removal

Two independent changes bundled on one branch at the author's request.

## Change 1 — `no_edit_on_main` gates plugin-source markdown

**What:** `skills/*/SKILL.md` and `commands/*.md` are now blocked when edited directly on `main`/`master`, the same as a `.py` file. Previously the extension filter only gated code extensions, so plugin-source markdown fell through to "allow".

**Why:** plugin source carried in markdown is **source, not docs**. Editing `agentic-loop/SKILL.md` on main is the same class of mistake the hook exists to stop. The original docs-only carve-out was too broad — earlier the same session a `SKILL.md` edit landed direct on main precisely because `.md` was waved through.

**How:** two path arms added to the gated `case` in `no_edit_on_main.sh`, anchored on a `/` boundary so a stray dir like `myskills/` can't match (`*/skills/*/SKILL.md|skills/*/SKILL.md`, `*/commands/*.md|commands/*.md`); block reason reworded "code file" → "source file". Test suite extended 11→17 cases (relative + absolute `SKILL.md` deny, `commands/*.md` deny, plain docs + non-`SKILL.md` skill markdown still allow, `SKILL.md` on a feature branch still allow). Docs synced: `README.md`, `CLAUDE.md` (seam-convention line + hook-map row), `docs/REFERENCE.md`.

**Connected decision — `git push` is deliberately NOT gated.** The edit-time gate is the right seam; a push-time gate would be redundant (edit already blocked + branch protection), would break the workflow's required feature-branch push, and is brittle to match. See [[no_edit_on_main]] design note.

This also closed finding #4 of a five-tension review of the agentic-loop skill: the #42 [[skills-hooks-seam]] convention (a skill instructing a hook-gated action must name the hook + resolution) had been applied to other skills but not to the loop's own Phase 3/3a worker prompts — fixed in commit `4ac38bd` just prior. The other four tensions were assessed as **owned tradeoffs**, not defects.

## Change 2 — remove historical `docs/superpowers` spec/plan tree

Deleted the superseded A–E and self-containment spec/plan documents under `docs/superpowers/` (~4,500 lines). Main-branch hygiene; the superpowers→coderails vendoring itself remains documented at [[self-containment]]. Independent of Change 1, bundled per author request.

## Impact

- [[no_edit_on_main]] — gated set widened; new design notes (plugin-source rationale, git-push non-gating decision).
- Reinforces [[skills-hooks-seam]] / [[enforcement-model]]: enforcement belongs at the earliest reliable seam (edit-time), not duplicated downstream.
