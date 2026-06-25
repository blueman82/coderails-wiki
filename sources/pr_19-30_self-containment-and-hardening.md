---
title: "PRs #19–#30 — Self-containment + plugin hardening"
type: source
created: 2026-06-25
last_updated: 2026-06-25
sources: []
tags: [self-containment, skills, hooks, refactor, enforcement]
---

# PRs #19–#30 — Self-containment + plugin hardening

<!-- Ingested by /wiki-ingest. Cluster record — 12 PRs merged 2026-06-25. -->

## Cluster overview

One coherent body of work with three threads: (1) vendor 12 superpowers skills into coderails, (2) build two "phantom" enforcement hooks that review flagged as missing, (3) refactor install/hooks for maintainability.

## PRs included

| PR | Title | Thread |
|---|---|---|
| #19 | Vendor Tier A workflow skills (worktrees, code-review, finishing-branch) | Self-containment |
| #20 | Vendor dispatching-parallel-agents, systematic-debugging | Self-containment |
| #21 | Vendor verification-before-completion, writing-skills | Self-containment |
| #22 | Vendor subagent-driven-development, executing-plans | Self-containment |
| #23 | Vendor using-coderails + SessionStart bootstrap hook | Self-containment |
| #24 | agentic-loop: ref vendored subagent-driven-development + fix stale claude-guardrails | Self-containment |
| #25 | Vendor brainstorming + visual companion (blueprint theme + Decision Ledger) | Self-containment |
| #26 | Docs: coderails self-containment spec + plan | Self-containment |
| #27 | feat(hooks): no_edit_on_main PreToolUse hook | Enforcement |
| #28 | refactor(install): derive hook chmod list from hooks.json | Refactor |
| #29 | refactor(hooks): extract shared discipline_common.sh | Refactor |
| #30 | feat(hooks): enforce_pr_workflow PreToolUse gate | Enforcement |

## Thread 1 — Self-containment (PRs #19–#26)

Vendors 12 core dev-workflow skills from superpowers into the `coderails:` namespace. All superpowers references stripped. Net result: superpowers is now uninstallable without breaking coderails workflows. The one retained external dependency is `pr-review-toolkit` (used by agentic-loop Phase 4b). (verified — PR #26 design doc)

Skills vendored:
- `using-git-worktrees` (PR #19)
- `requesting-code-review` + `code-reviewer.md` companion (PR #19)
- `receiving-code-review` (PR #19)
- `finishing-a-development-branch` (PR #19)
- `dispatching-parallel-agents` (PR #20)
- `systematic-debugging` (PR #20)
- `verification-before-completion` (PR #21)
- `writing-skills` (PR #21)
- `subagent-driven-development` incl. `implementer-prompt.md`, `task-reviewer-prompt.md`, `scripts/` (PR #22)
- `executing-plans` (PR #22)
- `using-coderails` (PR #23, renamed from using-superpowers)
- `brainstorming` + visual companion (Node server, blueprint theme, Decision Ledger panel) (PR #25)

PR #23 also adds `hooks/scripts/inject_bootstrap.sh` — a SessionStart hook that bootstraps the session with coderails context, replacing the superpowers equivalent. TDD-tested (hooks/tests/).

PR #24 additive-rewires agentic-loop to point Phase 3 worker-prompt construction at `coderails:subagent-driven-development` and fixes dead `/claude-guardrails:*` references → `coderails:assumptions` / `coderails:notchecked`. C1/C2 no-touch regions kept byte-identical. (verified — PR #24 body)

Design spec lives at `docs/superpowers/specs/2026-06-25-coderails-self-containment-design.md` (PR #26).

## Thread 2 — Enforcement hooks (PRs #27, #30)

Two hooks that were flagged as "phantom" in the code review (review record: `docs/coderails-review.md`) and implemented:

**`no_edit_on_main.sh`** (PR #27): PreToolUse hook on Write/Edit/MultiEdit. Blocks code-file edits (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`) on `main`/`master` branches. Docs and config files pass. Uses `permissionDecision: deny` (mirrors `destructive_bash_gate`). 11/11 TDD tests pass.

**`enforce_pr_workflow.sh`** (PR #30): PreToolUse(Bash) hook that blocks `gh pr create` unless `/coderails:push` ran this session and blocks `gh pr merge` unless `/pr-review-toolkit:review-pr` ran this session. Guards the workflow chain mechanically. Opt-in via NO_CONFIG sentinel — only active when `workflow.config.yaml` is present. Transcript scan approach.

Both hooks auto-chmod'd via the hooks.json-derivation (#28 refactor).

## Thread 3 — Refactors (PRs #28, #29)

**PR #28**: `install.sh` now derives its hook `chmod` list from `hooks/hooks.json` via `jq .hooks[][].hooks[].command` instead of a hardcoded list. Lib scripts chmod'd via glob. The resolved set was verified identical to the prior hardcoded 15. No more drift when hooks are added.

**PR #29**: Extracted shared `hooks/scripts/lib/discipline_common.sh` from the three discipline hooks (`check_confidence_labels.sh`, `check_verify_loop.sh`, `discipline_catchup.sh`). Mirrors the pattern of `lib/loop_state_common.sh`. Behaviour-preserving (proven against origin/main pre-refactor). TDD test added.

## Wiki pages updated

[[self-containment]] (new design page) — [[no_edit_on_main]] (new hook) — [[enforce_pr_workflow]] (new hook) — [[inject_bootstrap]] (new hook) — 12 new skill pages — [[agentic-loop]] (stale ref fix) — [[discipline-loop]] (discipline_common.sh) — [[enforcement-model]] (new hooks) — [[index]] (all new pages) — [[log]]

## Caveats

- The `pr-review-toolkit` external dependency is intentional and documented (agentic-loop Phase 4b still needs it). (verified — PR #24)
- `enforce_pr_workflow.sh` is NO_CONFIG opt-in: the hook no-ops if `workflow.config.yaml` is absent. This is the correct behaviour for repos not using the full workflow chain. (inferred — PR #30 body)
- `brainstorming` visual companion requires Node.js (the start script launches a local server). This is a soft dependency — the skill works without it; the visual is opt-in. (inferred — PR #25)
