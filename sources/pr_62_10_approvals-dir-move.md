---
title: "PR #62/#10: send-gate approval dir moves queue/ -> approvals/"
type: source
created: 2026-07-07
last_updated: 2026-07-07
sources: []
tags: [source, dashboard, send-gate, queue, approvals, assistant-agent, workflow-audit, routines, sub-project-4-of-5]
---

# PR #62/#10: send-gate approval dir moves `queue/` → `approvals/`

Cross-repo directory-move fixing a collision the queue-mode work ([[pr_43-44-46_workflow-audit-queue-seam]]) and the routines runner seam introduced against the send-gate's own approval files.

| | |
|---|---|
| Coderails PR | [#62](https://github.com/blueman82/coderails/pull/62), merged `c7e363c` |
| assistant-agent PR | [#10](https://github.com/blueman82/assistant-agent/pull/10), merged `a2b7e6a` |
| Builds on | [[pr_28_assistant-link-queue-contract-and-panel-spec]]'s on-disk layout, [[pr_43-44-46_workflow-audit-queue-seam]]'s WU1 proposal writer, the routines/runner intent-queue seam |

## The collision (why this moved)

`~/.claude/coderails-dashboard/queue/` had grown two independent consumers that both glob every `.json` file in the directory and interpret anything they don't recognise as **malformed input to quarantine or reject**:

1. The **routines runner sweep** (`skills/dashboard/runner/`) — quarantines any file that doesn't parse as its own `Intent` shape (`{button, input?, requestedAt, source}`).
2. **workflow-audit's `propose-skill` proposals** ([[pr_43-44-46_workflow-audit-queue-seam]]) — a distinct producer/consumer pair sharing the directory.

The send-gate's `QueueFileEntry` approval files ([[assistant-link-send-gate-architecture]]) are exactly the wrong shape for both of those parsers — a legitimate pending approval would get swept and quarantined by the runner as a malformed intent. The fix splits send-gate approval traffic into its own sibling directory, `approvals/`, leaving `queue/` untouched for the runner and workflow-audit.

## What changed

**Coderails PR #62** (dashboard reader side, 3 files):
- `skills/dashboard/app/src/app/api/events/route.ts` — `DEFAULT_QUEUE_DIR` → `approvals`
- `skills/dashboard/app/src/app/api/queue/route.ts` — `DEFAULT_QUEUE_DIR` → `approvals`
- `docs/coderails/specs/2026-07-06-assistant-link-panel-design.md` — on-disk layout path updated, plus a new "Note (2026-07-07)" section explaining the split (quoted above) and stating this is a clean-break move with no dual-dir fallback and nothing to migrate (no prior producer ever wrote approval-shaped files into `queue/`)

**assistant-agent PR #10** (send-gate writer side, 1 file, 1 line):
- `gate/surfaces/queue.ts` — `DEFAULT_QUEUE_DIR = join(homedir(), ".claude", "coderails-dashboard", "queue")` → `"approvals"`. `queue.test.ts`'s 6 tests are unaffected — they inject a temp dir rather than depend on the constant.

**Untouched, confirmed by both PRs' test plans and independently re-verified (2026-07-07 orchestrator pass):**
- `skills/workflow-audit/scripts/write_queue_entry.sh` — still writes `queue/`
- `skills/dashboard/runner/src/main.ts` — still reads `queue/`
- No `"queue"` string literal remains in either changed dashboard route file (no dual-dir fallback shim)

## Verification (orchestrator pass, 2026-07-07)

Independently re-derived from `origin/main` on both repos post-merge, not taken on the PR reports' word:
- `git show origin/main:gate/surfaces/queue.ts` — `DEFAULT_QUEUE_DIR` reads `approvals`, confirmed
- `git show origin/main:skills/dashboard/app/src/app/api/queue/route.ts` — `DEFAULT_QUEUE_DIR` reads `approvals`, confirmed
- `git grep coderails-dashboard` on both repos' `origin/main` — the only remaining `queue/` references are the workflow-audit script/skill-doc, the routines-runner plan doc, and `REFERENCE.md`'s queue-mode description, all of which are the *other* seam and correctly untouched
- Full test suites green on both repos at the merge SHAs (assistant-agent: 67/67; dashboard app/lib/runner: 250+48+70/368 per PR #62's own test plan)

## Residual doc drift found (not part of this PR, flagged for follow-up)

`assistant-agent/AGENTS.md` line 96 still describes the approval surface as "the dashboard queue file at `~/.claude/coderails-dashboard/queue/`" — this line predates PR #10 (last touched at an earlier commit) and was not updated by it. Low-stakes (one prose sentence in a docs file, not code), but genuine drift introduced by this move, owed as a follow-up. The frozen eval artifact `.claude/evals-wu2-pr.json` also names the old path but is an intentional point-in-time snapshot, not live documentation — not drift.

## See also

- [[assistant-link-send-gate-architecture]] — the queue seam this moves, including the `QueueFileEntry` shape and writer/reader split
- [[pr_43-44-46_workflow-audit-queue-seam]] — the proposal producer that shares `queue/` with the routines runner, whose collision risk motivated this move
- [[queue-contract-cross-pr-audit_2026-07-07]] — the earlier cross-PR audit of the queue contract, written before this split existed
- [[dashboard]]
