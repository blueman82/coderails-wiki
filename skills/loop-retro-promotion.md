---
title: "Skill: loop-retro-promotion"
type: skill
created: 2026-07-17
last_updated: 2026-07-17
sources:
  - sources/pr_36-41-33-53-65_verified-routines.md
  - sources/pr_201_202_203_routine-followups.md
  # last_updated unchanged (already 2026-07-17); this edit is same-day
tags: [skill, routines, agentic-loop, retro, promotion, machine-run, agentic-os]
---

# Skill: loop-retro-promotion

A **machine-run** pipeline that mines accumulated [[agentic-loop]] retros for durable, repo-agnostic lessons and promotes them out of the per-repo overlay (`standing-orders.md` / `retro.json`) into `skills/agentic-loop/learned-failure-modes.md`. It is dormant until enough loop history exists.

Source: `coderails/skills/loop-retro-promotion/SKILL.md`

## Not for interactive use

The description explicitly says "Runs on a schedule, NOT for interactive use — do not invoke this for a single loop's retro or from inside an active agentic-loop session" `(verified, SKILL.md frontmatter)`. Its interactive counterpart is the per-loop Phase 13 teardown, which writes one `retro.json`; this skill is the *aggregator* across many of them. It runs as the `loop-retro-promotion-weekly` routine — see [[routines]].

## The graduation predicate — why it's usually dormant

Every scheduled run evaluates the predicate **first**, before doing anything else, and this happens even on a dormant run. All three conditions must hold `(verified, SKILL.md §1)`:

1. `>= 10` `retro.json` files under the repo-key dir.
2. `>= 1` `standing-orders.md` entry with `last_recurred != created` — one full lifecycle.
3. `>= 1` entry in `standing-orders-decayed.md` — one clean decay.

Each run appends one line to `promotion-runs.log`:

```
<ISO8601> predicate=<met|unmet> retros=<n> lifecycle=<0|1> decay=<0|1>
```

**On an unmet predicate the run STOPS there, and the log line IS the run's artifact** — no branch, no PR, no gate chain. This is the design's key move: a dormant run is a *successful* run, and it still produces the artifact its routine gate checks for. Dormancy is therefore indistinguishable from health to the runner, by intent.

## Promotion bar — recurrence AND repo-agnosticism

A lesson is promoted only if it recurred `>= 2` times across `>= 2` distinct `session_id`s **and** is repo-agnostic — applicable to any repo running [[agentic-loop]] `(verified, SKILL.md §2)`. Repo-specific lessons are **rejected even when they clear the recurrence bar**; they stay in the overlay. This is what keeps `learned-failure-modes.md` a shared asset rather than an accumulation of this repo's local trivia.

## Delivery — manifest-locked, full gate chain

The pipeline runs the same gate chain as any human change ([[review-artifact-seam]], [[task-evals-gate]]): freeze evals *before* the edit, then push → `review-pr` → `post-review` → `post-evals` → `/coderails:merge`.

Two constraints are load-bearing and worth naming:

- **Manifest lock.** After editing, it asserts `git diff origin/main --name-only` equals **exactly** `skills/agentic-loop/learned-failure-modes.md`. Any other file in that diff means **abort with cleanup** — close the PR, delete the branch locally and remotely, append `abort=<reason>` to the log. A self-editing pipeline that writes exactly one file is auditable; one that can touch its own gates is not — hence the explicit prohibition on editing SKILL.md, hook scripts, gate logic, its own definition, the routine config, or the predicate `(verified, SKILL.md §5)`.
- **Never raw `gh pr merge`.** It merges ONLY via `/coderails:merge`, because **PreToolUse hooks do not fire in this headless execution mode** — so `merge.sh`'s script-internal artifact gates are the only merge rail available `(verified, SKILL.md §5)`. This is a concrete instance of the [[enforcement-model]] ceiling: the hook layer simply is not present here, and the script-internal gate is what remains. See [[skills-hooks-seam]].

## Failure modes encoded

**Born red — the deploy incident (2026-07-13, PRs #151 + #152).** Added 2026-07-10 without a live fire, this routine failed its artifact gate every run at exit 0 in ~4–9s, from two stacked causes invisible to the runner: the skill had merged *after* the last plugin version bump (so the installed plugin cache a headless `claude -p` actually loads never contained it), and its frontmatter declared `user-invocable: false`, which removes the slash command entirely — "machine-run only" authored via the exact mechanism that disables the machine's invocation path. Full detail in [[routines]].

> **✅ Corrected (2026-07-17, same day as the note below) — the "gate can never pass" claim was FALSE against live config.** A same-day handoff memory asserted the routine's artifact gate "CAN NEVER PASS," citing the reasoning captured below. Re-verified against **live state, not source greps** `(verified 2026-07-17)`: `~/.claude/coderails-dashboard.json` (the config the routine actually runs against — `examples/dashboard-config.json` is documentation-only, see [[dashboard]]) points `artifactPath` at `~/.claude/agentic-loop/-Users-harrison-Github-coderails-.git/promotion-runs.log` — the **correct** repo-key dir. That file exists, last modified 2026-07-13, age ≈4.4 days against the 8-day (691200s) `maxAgeSeconds` window — **the gate currently passes**. The claim was true only of `examples/dashboard-config.json`'s stale `Documents/Github` path (fixed same-day by PR #201, see [[pr_201_202_203_routine-followups]]), never of the live config this routine actually uses. This is a **timing-sensitive** pass: without a fresh `loop-retro-promotion-weekly` fire before 2026-07-21 (≈8 days after the 2026-07-13 artifact), the gate will age out and genuinely fail again — that would be a real (if different) recurrence of staleness, not a repeat of this false claim.
>
> **Original note (superseded above, kept for the incident record):** "the routine's artifact gate can never pass... The repo-key dir the loop-state resolver now produces has drifted from the one pinned in the routine config... this is path rot, not a regression." This diagnosis was written against the example config's rot, not the live config's actual (correct) path — a handoff memory's claims are hypotheses against source, not facts about live state, until independently re-verified. See [[pr_201_202_203_routine-followups]] for the generalised lesson.
