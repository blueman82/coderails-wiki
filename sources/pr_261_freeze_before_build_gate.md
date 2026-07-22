---
title: "PR #261 — freeze-before-build becomes mechanical (validate_structure check 8)"
type: source
origin: "coderails PR #261 (merged b10bdfb, 2026-07-22)"
created: 2026-07-22
last_updated: 2026-07-22
sources: ["scripts/post_evals.sh", "hooks/scripts/tests/post_evals.test.sh", "scripts/lib/eval-artifact.sh"]
tags: [source, task-evals, gate, post-evals, freeze-before-build, fail-open]
---

## The gap

`/coderails:task-evals` rule 1 says evals are "generated and frozen (timestamp + base SHA)
before implementation starts". `frozen_sha` was written into every artifact — and read by
nothing:

```
grep -rn frozen_sha hooks/ scripts/   →   0 hits
```

The merge gate already blocks on a posted GO artifact, but nothing checked *when* that
artifact was authored. Evals could be written after the code and pointed at any commit.
Freeze-before-build was a documented rule with no enforcement, i.e. rule 1 was the one
anti-gaming rule with no mechanism behind it.

## The check

Adds **check 8** to `post_evals::validate_structure`, factored into
`post_evals::validate_freeze`: `frozen_sha` must be an ancestor of the branch's merge-base
with the default branch — a commit that already existed before the branch's own
implementation commits.

| Case | Behaviour |
|---|---|
| Frozen at the branch base | pass |
| Frozen at one of the branch's own commits | **refuse**, naming `frozen_sha` and the resolved base |
| Disclosed late freeze | pass (see below) |
| `frozen_sha` git can't resolve | **refuse** — "git couldn't answer" must never read as compliance |
| No `frozen_sha` field, or file outside a work tree | skip — absence of applicability, not violation |
| Loop scope | skip — loop artifacts live beside `progress.json`, outside any repo |

The skip on an absent field is load-bearing for back-compat: every artifact predating this
check omits `frozen_sha`, and the loop scope never carries one.

## The disclosed-late-freeze hatch

assistant-agent PR #54 set the precedent (a different repo, so no wiki-link — its artifact is
at `blueman82/assistant-agent#54`). That artifact stated plainly that its evals were authored
after implementation and *not backdated*. That is honest, and the gate must enforce honesty
rather than forbid the disclosed case.

The disclosure must be explicit prose in `tier_justification` or an amendment `why`,
**deliberately not a boolean**: a flag can be set silently, a sentence has to be written and
is visible to any human reading the artifact. Same reasoning as `tier_justification` itself
(check 2).

> ⚠️ **Known limitation, grader-flagged and shipped anyway:** the hatch matches the
> substrings `freeze`/`frozen`. An unrelated sentence containing the word satisfies it. It
> is a heuristic, not a semantic check — a coarse bar deliberately preferred over a boolean,
> not a claim of rigour.

## The fail-open bug this surfaced

`validate_freeze` reads `frozen_sha` via `jq`. With `jq` absent, that read returns **empty**
— indistinguishable from "no `frozen_sha` field", which takes the skip path. **The gate would
pass while verifying nothing.**

Proven by stripping the guard and re-running a violating artifact:

```
WITHOUT guard, jq absent, VIOLATING artifact -> rc=0   (failed open)
WITH guard,    jq absent, VIOLATING artifact -> rc=1   (fails closed)
```

`validate_freeze` now guards explicitly on `command -v jq`, so a later refactor can't quietly
reintroduce it. **The general shape is worth carrying:** any check whose skip path keys on an
*empty read* fails open when the tool doing the reading is missing, because "no data" and "no
answer" become the same value. This is the same class as
[[design/enforcement-model]]'s fail-closed principle, arriving through a tooling absence
rather than a logic error.

## Dogfooding

This PR's own `evals.json` was frozen at `3de997b` — the merge-base of its branch — before any
implementation commit, and is accepted by the check it introduces. Its negative control (same
file, `frozen_sha` repointed at branch commit `4119cc5`, disclosure stripped) is correctly
refused.

That control was itself **vacuous on first write**: it put the mutated copy in `/tmp`, outside
any git work tree, where the check correctly skips — so it reported PASS while exercising
nothing. Fixed to write inside the repo. This is a live instance of the same failure class that drove
the evals-system redesign: **a negative control that passes for an irrelevant reason is
indistinguishable from one that passes for the right one.** That work is PR #264 (open as of
this ingest, no wiki page yet — ingest it when it merges); the four catalogued failure
instances live in the `project_evals_system_redesign` handoff memory.

## Verification

- 8 new cases in `hooks/scripts/tests/post_evals.test.sh`, each watched fail first (4 genuine
  RED failures before the check existed)
- Fixtures are **real throwaway git repos**, not stubbed git — the check *is* git ancestry, so
  a stub would test the stub
- 52 of 53 hook suites pass; `docs_sync_routine` skips (exit 3) on a cold worktree for missing
  `node_modules`, unrelated
- Eval artifact:
  `https://github.com/blueman82/coderails/pull/261#issuecomment-5049265285`

## Scope boundary

PR scope only, matching where the merge gate reads. Loop scope has the same rule but a
separate surface (`loop_state_guard`) — deliberately not folded in, so the two gates can be
reasoned about independently.
