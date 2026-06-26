---
title: "PR 54 — vendor strictcode as engineering-principles"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, engineering-principles, strictcode, vendoring, rename, self-containment, code-quality]
---

# PR 54 — vendor strictcode as engineering-principles

<!-- Ingested by /wiki-ingest after merge. Immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #54 |
| Branch | `worktree-engineering-principles` |
| Merged | 2026-06-26 |
| Merge SHA | `2f1ad1c` |
| JIRA ticket | — |

## Summary

Brought the **global** `strictcode` skill family (`strictcode` + `strictcode-python`/`-go`/`-ts`, which lived in `~/.claude/skills/`, outside the plugin) **into coderails** as the **`engineering-principles`** family, completing the self-containment goal for code-quality enforcement. This **supersedes [[pr_47_strictcode-skill-config]]**, which made the (then-external) skill configurable; the skill is now vendored and renamed. (verified — 4 skills present on `main`, loadable as `coderails:engineering-principles*`)

## What changed

- **4 skills vendored + renamed**: `skills/engineering-principles/SKILL.md` (coordinator) + `-python`/`-go`/`-ts`. The coordinator's Phase 0/Step 1 dispatch tables route to the renamed sub-skills.
- **Scout + SlimCode stripped**: the coordinator's Phase 3 (Scout semantic search) and "Integration with SlimCode" sections were removed, and the principle table's DRY/SSOT/YAGNI rows rewritten to a **Grep/Glob fallback**. **Serena (LSP via `mcp__mcp-exec__*`) is retained** with its existing graceful-degradation clause. Rationale: keep the vendored skill self-contained — `scout-*` and `/slimcode` are not shipped with coderails. (verified — no `scout`/`slimcode` tokens in the shipped skill)
- **Config keys renamed**: `config.strictcode_paths`/`strictcode_skill` → **`engineering_principles_paths`/`engineering_principles_skill`**; default `/strictcode-python` → **`/engineering-principles-python`**. Applied across [[push]], [[workflow]], [[prep]], [[init]], and [[config-resolution]]. The `/coderails:init` autodetection now maps `go.mod` → `/engineering-principles-go`, `package.json`+`.ts` → `/engineering-principles-ts`, else `/engineering-principles-python`.
- **Three touchpoints** (the new wiring): planning ([[writing-plans]] self-review gate gains an engineering-principles item; [[brainstorming]] gains a principle-vetting note), pre-push pre-flight (existing, renamed), and **PR-review** (new `workflow.md` Phase 3 steps `2b` run `/engineering-principles` + `2c` run `/simplify`). `/simplify` is the built-in command, kept because `review-pr`'s own `code-simplifier` agent is not guaranteed to run.
- **README**: 23 → 27 skills across **four** groups (new "Engineering principles" group).
- **Global copies removed** post-merge so there is one engine, not two.

## Process notes

- The plan was put through `/coderails:planning-sequence` before implementation (per the gate landed in [[pr_50_planning-sequence-gate]]). That stress-test raised a "branch name blocks `/push`" concern which **verification disproved** (`require::feature` only rejects main/master, not non-`feature/` names), and surfaced two real hardening items folded into the plan (positively assert the renamed default landed; verify the no-Serena degradation path).
- Review (`/pr-review-toolkit:review-pr`) returned clean apart from one article grammar nit (`a engineering-principles` → `an`) from the mechanical rename, fixed before merge.

## Caveats / gotchas

- `.claude/workflow.config.yaml` is **gitignored** (local-only), so the live config's key rename to `engineering_principles_paths` is not in the PR — it was updated in place on the maintainer's machine.
- This is a **skill rename, not a hook** — the touchpoints are advisory ([[enforcement-model]]): nothing mechanically blocks an implementation that skips the principle checks.
- Muscle-memory break: `/strictcode-python` no longer resolves (globals removed); use `/engineering-principles-python`.
