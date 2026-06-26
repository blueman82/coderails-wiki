---
title: "PR #46 — gate git push on main/master in enforce_pr_workflow"
type: source
origin: "PR #46 (merged 2026-06-26, squash 7a4906c)"
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, PR, hook, enforce_pr_workflow, main-branch-protection, pr-workflow, refspec]
---

# PR #46 — gate `git push` to main/master

Adds a fourth gated subcommand (`git_push`) to [[enforce_pr_workflow]], mirroring the `git merge` gate added in [[pr_40_hook-hardening]]. When `workflow.config.yaml` is present (opt-in), a `git push` that lands on `main`/`master` is now blocked unless `/pr-review-toolkit:review-pr` ran this session.

## PR metadata

| Field | Value |
|---|---|
| PR number | #46 |
| Title | feat(hooks): gate git push on main/master in enforce_pr_workflow |
| Merged | 2026-06-26 |
| Merge SHA | `7a4906c` |
| JIRA ticket | — |

## Summary

`git push` joins `gh pr create`, `gh pr merge`, and `git merge` as a gated action in `enforce_pr_workflow.sh`. The required evidence is the same as for `git merge`: `/pr-review-toolkit:review-pr` must have run in the session transcript. Feature-branch pushes are never gated — the PR flow requires them, so gating them would break the workflow itself.

## Destination-refspec gating (not just current branch)

The headline design decision: Gate 4b for `git_push` keys off the push **destination**, not only the checked-out branch. It gates when **either**:

- (a) the current branch is `main`/`master`, OR
- (b) the command names an explicit main/master destination refspec — `HEAD:main`, `feature:master`, `:refs/heads/main`.

This is stricter than the `git merge` gate, which can rely on the current branch alone (a merge integrates into the checked-out branch). A push's effect is decided by where it *sends* commits, so the destination must be parsed. (verified — hook source lines 67–86)

The destination anchor `:(refs/heads/)?(main|master)([[:space:];&|)]|$)` accepts whitespace, EOL, **or** a shell separator (`;& |)`) after the ref. Without the metachar class, `git push origin HEAD:main;echo` would abut `;` directly onto `main` and slip the gate. (verified — hook source line 82)

## Documented limitation

Bare positional `git push origin main` from an off-main branch is **not** parsed and not gated. This is low risk: it pushes the local `main`, which tracks `origin/main` anyway. The realistic direct-to-main bypass is the colon-refspec form, and that is closed. (verified — hook source lines 9–10, 70–73)

## Review process point worth keeping

The first implementation gated on the **current branch only**. The multi-agent `pr-review-toolkit` review caught this as a **Critical silent false-allow**: a `HEAD:main` refspec push from a feature branch passed straight through. The fix was destination-refspec gating, followed by a metachar-anchor hardening so a separator abutting the ref couldn't evade it — each landed TDD-first. The enforce test suite grew 14→27 cases. This is a worked example of a current-state check missing the actual decision variable (destination, not location). (verified — PR #46 body / review trail)

## Relationship to PR #44's "don't gate git push" decision

This does **not** reverse [[pr_44_no-edit-plugin-source]]'s decision that `git push` should not be gated. That decision was about [[no_edit_on_main]] — protecting *source files from direct edits*, where edit-time is the right seam and a push gate would be redundant. PR #46 gates push in a *different hook for a different reason*: requiring review evidence before commits reach main, mirroring the `git merge` gate. The two are not in tension — they govern different invariants. (inferred — comparing both source pages; see [[enforcement-model]])

## Wiki pages updated

- [[enforce_pr_workflow]] — git_push gate, destination-refspec model, bare-positional limitation.
- [[skills-hooks-seam]] — new `git push` row in the gated-actions table.

## Caveats / gotchas

- Opt-in only: no `workflow.config.yaml` → no-op (NO_CONFIG), same as every other gate in this hook.
- Bypass: add a `git push` Bash permission to `settings.json`, or run `/pr-review-toolkit:review-pr` first.
