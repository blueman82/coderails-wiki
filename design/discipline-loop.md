---
title: Discipline Loop
type: design
created: 2026-05-30
last_updated: 2026-06-25
sources:
  - templates/failure_log.md
  - hooks/scripts/check_confidence_labels.sh
  - hooks/scripts/check_verify_loop.sh
  - CLAUDE.md
  - instructions/self-checking-discipline.md
  - sources/session_2026-05-31_prompting-doc-alignment.md
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

**`check_verify_loop.sh`** — **total enforcement** (as of 2026-06-01): blocks when *any* `## Did Not Verify` bullet is left untagged. The sole escape is an explicit `(unverifiable: <reason>)` tag on the bullet's leading clause.

Gate chain (verified: [[check_verify_loop]]):
1. No transcript file — allow stop.
2. No files edited this turn (file_count < 1) — allow stop (pure conversation turns are exempt).
3. `stop_hook_active == true` — allow stop (loop-guard: already blocked once this turn).
4. Last response has no text — allow stop.
5. No `## Did Not Verify` bullets — allow stop.
6. Any DNV bullet **not** tagged `(unverifiable: …)` — **block** with `exit 2`.

The earlier source-token regex (matching `.md`, `.sh`, etc. extensions) and the `meta_pattern` allowlist were both removed in the 2026-06-01 escalation. Now prose claims block exactly as filename claims do — any untagged bullet is treated as something the model could have confirmed. See [[check_verify_loop]] for the full history. (inferred: [[session_2026-06-01_verify-loop-total-enforcement]])

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
| Did Not Verify section after file-editing responses | `~/.claude/CLAUDE.md` prose | Advisory |
| DNV bullets must not name unread source files | `check_verify_loop.sh` | **Block (Stop hook)** |
| Ask on ambiguity, verify memory before acting | `~/.claude/CLAUDE.md` prose | Advisory |

The prose rules are not redundant. They cover the cases the hooks don't — short responses, conversational turns, runtime claims. The hooks cover the high-value case where Claude might stop with a false claim of completeness.

## Prose as Standard, Hook as Floor (key design invariant)

This is the central layering principle of the discipline loop, made explicit in `instructions/self-checking-discipline.md:10–11` (verified) as of 2026-05-31.

The **prose rules** (in `~/.claude/CLAUDE.md`) describe the standard Claude *should* aim for:
- Label every substantive claim with a confidence tag
- Write a Did Not Verify section after any response that edits one or more files

The **Stop hooks** enforce a *floor* that is intentionally lower than the prose standard:
- `check_confidence_labels.sh` only blocks responses ≥200 chars with no label at all — it does not require every claim be labelled, just that the response isn't entirely unlabelled (verified: [[check_confidence_labels]])
- `check_verify_loop.sh` only blocks when a DNV bullet names a source-resolvable file token — it does not require a DNV section on every response, just that named files be resolved (verified: [[check_verify_loop]])

**Why the floor is lower than the standard:** mechanical hooks cannot encode nuanced judgment about what "substantive" means in context, or when a DNV section genuinely adds value vs. boilerplate. The prose standard asks Claude to apply judgment; the hooks catch the case where that judgment fails in a high-stakes way (a long response with no accountability markers, or a DNV that lists an unread file as a known gap). See [[hook-exit-codes]] for the block mechanisms.

**What this means in practice:** following the hook floor is necessary but not sufficient. A response can satisfy both hooks (≥1 label, no file-named DNV bullet) while still falling short of the prose standard (many unlabelled claims, no DNV section at all on a file-editing response). The prose standard is the real target. The hooks are the backstop for clear failures.

## Cross-References

- [[enforcement-model]] — why hooks can enforce things that commands cannot
- [[check_confidence_labels]] — the confidence-label Stop hook in detail
- [[check_verify_loop]] — the verify loop hook in detail
- [[hook-exit-codes]] — which hook events block on exit 2 vs. permissionDecision: deny
- [[install-and-cache-trap]] — hook edits in the repo do not take effect until cache is re-synced
