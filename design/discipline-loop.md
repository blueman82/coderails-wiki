---
title: Discipline Loop
type: design
created: 2026-05-30
last_updated: 2026-05-30
sources:
  - templates/failure_log.md
  - hooks/scripts/check_confidence_labels.sh
  - hooks/scripts/check_verify_loop.sh
  - CLAUDE.md
  - instructions/self-checking-discipline.md
tags:
  - discipline
  - hooks
  - confidence-labels
  - did-not-verify
  - enforcement
---

# Discipline Loop

The self-checking discipline system: what it is, what it enforces mechanically, and what it only requests.

## The Founding Thesis

`templates/failure_log.md` line 26 (verified) records the empirical failure that motivated block-mode:

> Multiple turns this session lacked the DNV section while warn-mode hooks fired silent reminders that I ignored. The user explicitly observed "another claude session just ignored the hooks prompt" — same failure mode. **Warn-mode + memory-only enforcement is mechanically insufficient** — confirms the Shingo prediction quoted during the build.

This is the founding data point. The discipline system existed in warn-mode first. It was tested in a live session. It failed. Claude ignored the hooks. The lesson: a rule that only reminds does not produce the behaviour. A rule that blocks does.

## Two Disciplines, Two Enforcement Levels

### Advisory: the CLAUDE.md prose rules

`instructions/self-checking-discipline.md` installs rules into `~/.claude/CLAUDE.md` via `install.sh`. These cover:

- Confidence labels: tag non-trivial assertions `(verified)`, `(inferred)`, or `(guess)`
- Did Not Verify section: after multi-file changes, list what was NOT checked
- Ask on ambiguity: use `AskUserQuestion` rather than silently filling in interpretations
- Verify memory before acting: re-read/grep the file rather than recalling

These are prose instructions to Claude. Claude *should* follow them. They are not enforced by any gate.

### Mechanically Enforced: the Stop hooks

Two Stop hooks promote the most important of those rules into block-mode gates.

**`check_confidence_labels.sh`** — blocks responses ≥200 chars that carry no confidence label.

From the script header (verified): *"BLOCK-MODE: exits 2 when confidence labels are missing (promoted from warn-mode 2026-05-05)."*

Logic at lines 44–66 (verified):
- Reads the last assistant text block from the transcript, retrying up to 5 times with 0.3s backoff to handle the transcript-flush race.
- If text length < 200 chars: passes (short responses are not substantive claims).
- If text contains `(verified`, `(inferred`, or `(guess`: passes.
- Otherwise: blocks with `exit 2` and a message naming the failure.

**`check_verify_loop.sh`** — blocks when a `## Did Not Verify` bullet names a source-resolvable file token.

From the script header (verified): *"A '## Did Not Verify' bullet that names a file or file:line (e.g. prep.md:96, hooks/scripts/x.sh) is something the model could have confirmed by reading the working tree, so listing it unresolved is the failure this hook catches. Bullets about runtime behaviour, external systems, or user intent name no such token and are treated as genuinely unverifiable — they pass."*

Gate chain at lines 31–130 (verified):
1. No transcript file — allow stop.
2. No files edited this turn (file_count < 1) — allow stop (pure conversation turns are exempt).
3. `stop_hook_active == true` — allow stop (loop-guard: already blocked once this turn).
4. Last response has no text — allow stop.
5. No `## Did Not Verify` bullets — allow stop.
6. A DNV bullet matches `[file.ext]` or `file:line` pattern — **block** with `exit 2`.

The regex at line 117 (verified) matches tokens with known source extensions (`.md`, `.sh`, `.json`, `.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`, `.yaml`, `.yml`, `.txt`) or `filename:linenumber` form. Bullets naming only runtime behaviour, external systems, or user intent carry no such token and pass through gate 5.

## What This Means in Practice

Writing a DNV section that says:

```markdown
## Did Not Verify
- Whether the CI pipeline will pass
- User intent on the edge case
```

passes both hooks. Neither bullet names a file.

Writing:

```markdown
## Did Not Verify
- prep.md:96 — the exact config field name
```

blocks. `prep.md` has a known extension and `:96` is a line reference. The hook demands you read it before stopping.

This is the design intent: file-resolvable claims must be resolved before stopping. Genuinely unverifiable things (future runtime, external systems, another person's intent) may stay in DNV.

## Advisory vs. Enforced: Summary

| Rule | Where it lives | Enforcement |
|---|---|---|
| Confidence labels on non-trivial claims | `~/.claude/CLAUDE.md` prose | Advisory |
| ≥200-char response must have one label | `check_confidence_labels.sh` | **Block (Stop hook)** |
| Did Not Verify section after multi-file changes | `~/.claude/CLAUDE.md` prose | Advisory |
| DNV bullets must not name unread source files | `check_verify_loop.sh` | **Block (Stop hook)** |
| Ask on ambiguity, verify memory before acting | `~/.claude/CLAUDE.md` prose | Advisory |

The prose rules are not redundant. They cover the cases the hooks don't — short responses, conversational turns, runtime claims. The hooks cover the high-value case where Claude might stop with a false claim of completeness.

## Cross-References

- [[enforcement-model]] — why hooks can enforce things that commands cannot
- [[check_verify_loop]] — the verify loop hook in detail
- [[install-and-cache-trap]] — hook edits in the repo do not take effect until cache is re-synced
