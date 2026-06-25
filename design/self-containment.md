---
title: "Self-containment"
type: design
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [design, self-containment, vendor, superpowers, skills]
---

# Self-containment

The decision to vendor 12 superpowers dev-workflow skills into the `coderails:` namespace, making superpowers an uninstallable external dependency. Completed 2026-06-25 across PRs #19–#26.

## Context

coderails previously depended on superpowers being installed to provide core dev-workflow skills (`using-superpowers`, `subagent-driven-development`, `brainstorming`, `requesting-code-review`, etc.) and a SessionStart bootstrap hook. This was an implicit runtime dependency: if superpowers was uninstalled or unavailable, coderails workflows silently degraded. The design spec (`docs/superpowers/specs/2026-06-25-coderails-self-containment-design.md`) captured the full plan. (verified — PR #26)

## The rule

All coderails workflow skills are self-hosted under the `coderails:` namespace. Zero `superpowers:*` references anywhere in the plugin. The one retained external dependency is `pr-review-toolkit` (agentic-loop Phase 4b — review gate). (verified — PR #24 body)

## Skills vendored

12 skills from superpowers, rebranded with zero superpowers references:

| coderails skill | Origin |
|---|---|
| `coderails:using-git-worktrees` | superpowers:using-git-worktrees |
| `coderails:requesting-code-review` | superpowers:requesting-code-review |
| `coderails:receiving-code-review` | superpowers:receiving-code-review |
| `coderails:finishing-a-development-branch` | superpowers:finishing-a-development-branch |
| `coderails:dispatching-parallel-agents` | superpowers:dispatching-parallel-agents |
| `coderails:systematic-debugging` | superpowers:systematic-debugging |
| `coderails:verification-before-completion` | superpowers:verification-before-completion |
| `coderails:writing-skills` | superpowers:writing-skills |
| `coderails:subagent-driven-development` | superpowers:subagent-driven-development |
| `coderails:executing-plans` | superpowers:executing-plans |
| `coderails:using-coderails` | superpowers:using-superpowers (renamed) |
| `coderails:brainstorming` | superpowers:brainstorming (+ blueprint theme, Decision Ledger) |

## Bootstrap hook

PR #23 adds `hooks/scripts/inject_bootstrap.sh` — a SessionStart hook registered in `hooks.json`. Runs `coderails:using-coderails` at session start to replace the superpowers bootstrap. See [[inject_bootstrap]].

## agentic-loop rewire (PR #24)

agentic-loop Phase 3 worker-prompt construction now references `coderails:subagent-driven-development` (additive change — no existing prose removed). Dead `/claude-guardrails:*` references fixed to `coderails:assumptions` and `coderails:notchecked`. The six C1/C2 no-touch regions were kept byte-identical. (verified — PR #24)

## brainstorming twist

The vendored `coderails:brainstorming` adds a creative twist: a blueprint/rail visual theme (Node.js local server + frame template) and a Decision Ledger panel not present in the superpowers original. The visual companion is opt-in — the skill functions without it. (inferred — PR #25 body)

## Rationale

- Eliminates a hidden runtime dependency that caused silent degradation.
- Gives the coderails maintainer full control over skill evolution without waiting on superpowers releases.
- Makes the plugin genuinely portable: clone + install + use, no other plugin needed (except `pr-review-toolkit` for review gate).

## Known caveats / edge cases

- `pr-review-toolkit` remains an external dependency for agentic-loop Phase 4b. This is intentional and documented. If review-toolkit is unavailable, only the review gate step is affected — the rest of the workflow runs.
- `brainstorming` visual companion requires Node.js. Soft dependency — the skill itself does not require Node.
- All 12 vendored skills maintain their original logic; only namespace references and any internal `superpowers:*` skill calls were rewritten.

## Where it is enforced

Advisory (no hook): The self-containment rule is a codebase discipline, not a runtime hook. It's enforced by code review and the vendor-then-brand convention.

## See also

[[enforcement-model]] — hooks vs. commands  
[[inject_bootstrap]] — the SessionStart hook that replaces superpowers' bootstrap  
[[agentic-loop]] — the skill whose Phase 3 worker-prompt was rewired  
[[using-coderails]] — the renamed entry-point skill  
[[subagent-driven-development]] — the most-referenced vendored skill
