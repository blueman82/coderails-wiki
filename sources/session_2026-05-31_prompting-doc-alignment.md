---
title: "Session 2026-05-31: Prompting-doc alignment (5 changes)"
type: source
created: 2026-05-31
last_updated: 2026-05-31
sources:
  - hooks/scripts/destructive_bash_gate.sh
  - skills/agentic-loop/SKILL.md
  - instructions/self-checking-discipline.md
  - commands/workflow.md
tags: [prompting-best-practices, discipline, agentic-loop, hooks, workflow]
---

# Session 2026-05-31 — Prompting-doc alignment

Five changes derived from Claude's prompting best-practices document, applied directly to the coderails plugin and verified live (reinstalled + cache-synced this session).

---

## Change 1 — destructive_bash_gate: block `git commit --no-verify`

**Source:** `hooks/scripts/destructive_bash_gate.sh:14` (verified)

Added `\bgit +commit +.*--no-verify` to the conservative destructive-pattern regex.

```
\bgit +commit +.*--no-verify
```

**Doc basis:** "don't bypass safety checks (e.g. --no-verify)" — bypassing git commit hooks defeats the discipline loop entirely.

**What it blocks:** any `git commit` invocation that includes `--no-verify`, regardless of position in the arguments.

**Known limitation:** the `.*` in the pattern means a command like `git commit -m "please don't use --no-verify"` would match and be denied — the commit message text is included in the pattern scan. This is a substring false-positive risk. The pattern is intentionally conservative; the expected low rate of messages containing `--no-verify` as literal text means this is an acceptable tradeoff for now. (inferred)

See [[destructive_bash_gate]] for the full updated regex.

---

## Change 2 — agentic-loop: Context-window persistence section

**Source:** `skills/agentic-loop/SKILL.md:209` (verified)

New section added after Phase 12:

> Do not stop work early because the context window is filling or a token budget is approaching. Context will compact and the session will continue — treat that as a non-event, not a stop condition.
>
> Before compaction happens, checkpoint state: commit all in-progress work to git, write a brief progress note to a memory or a `progress.md` in the worktree, and record where the loop is in the phase sequence. Git is the authoritative checkpoint — uncommitted work is unrecoverable state.

**Doc basis:** prompting best-practice — don't abandon work on token pressure; checkpoint to durable storage before compaction.

See [[agentic-loop]] for full skill documentation (page created this session).

---

## Change 3 — agentic-loop: When to delegate vs. work directly

**Source:** `skills/agentic-loop/SKILL.md:102` (verified)

Guidance appended to Phase 3 (TeamCreate):

> Use subagents when tasks run in parallel, need isolated context, or are genuinely independent workstreams. For single-file edits, sequential steps, or work that requires shared context across steps, work directly rather than delegating — the handoff overhead costs more than it saves.

**Doc basis:** prompting best-practice — avoid unnecessary agent spawning for sequential/coupled work; delegation overhead is a real cost.

See [[agentic-loop]] for full skill documentation.

---

## Change 4 — self-checking-discipline: Tightened scope + floor-naming clauses

**Source:** `instructions/self-checking-discipline.md:10–11` (verified)

Two rule changes:

1. **Confidence labels scope tightened:** "every substantive claim" → still applies, but with explicit floor clause:
   > *"This is the standard you aim for; the `check_confidence_labels.sh` Stop hook enforces a floor below it (it blocks only responses ≥200 chars with no label). Aim higher than the floor."*

2. **Did Not Verify scope tightened:** "after any multi-file change or claim of completeness" → "after any response that edits one or more files", plus floor clause:
   > *"The `check_verify_loop.sh` Stop hook enforces a floor: it blocks when a DNV bullet names a source-resolvable file the response could have read. Aim higher than the floor."*

**This is the key design point for the discipline loop:** prose instructions = the standard Claude should aim for; hooks = a mechanical backstop enforced below that standard. These are intentionally different layers. The advisory prose covers cases the hooks don't reach (short responses, conversational turns, runtime claims). The hooks fire only at the boundary where false completeness claims cause real harm.

See [[discipline-loop]] for the updated design section.

---

## Change 5 — workflow.md: Parallel tool calls bullet

**Source:** `commands/workflow.md:29` (verified)

New bullet added to the Purpose section:

> **Parallel tool calls**: when multiple tool calls or file reads have no dependency between them, issue them in parallel in a single message — not sequentially. Never serialize work that can run concurrently.

**Doc basis:** prompting best-practice — always fan out independent work in parallel; sequential I/O is a waste of latency.

---

## Summary

| # | File | Nature | Enforced? |
|---|---|---|---|
| 1 | `destructive_bash_gate.sh:14` | New blocked pattern | Yes — PreToolUse deny |
| 2 | `agentic-loop/SKILL.md:209` | New section: context-window persistence | Advisory (skill prose) |
| 3 | `agentic-loop/SKILL.md:102` | New guidance: delegate vs. direct | Advisory (skill prose) |
| 4 | `self-checking-discipline.md:10–11` | Scope tightening + floor-naming clauses | Advisory prose + hook floor |
| 5 | `workflow.md:29` | Parallel tool calls bullet | Advisory (command prose) |

All 5 changes were verified live in the plugin cache this session.

---

## Cross-references

- [[destructive_bash_gate]] — Change 1 documented in detail
- [[discipline-loop]] — Change 4 design implication documented
- [[agentic-loop]] — Changes 2 and 3 documented
