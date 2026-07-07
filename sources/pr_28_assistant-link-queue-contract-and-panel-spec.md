---
title: "PR 28 — ASSISTANT.LINK queue contract and panel spec"
type: source
created: 2026-07-07
last_updated: 2026-07-07
sources: []
tags: [source, dashboard, assistant-agent, queue, spec, agentic-os, sub-project-4-of-5]
---

# PR 28 — ASSISTANT.LINK queue contract and panel spec

Ingested by /wiki-ingest after merge. This is an immutable record of what changed.

## PR metadata

| Field | Value |
|---|---|
| PR number | #28 |
| Branch | `feat/wu4-assistant-link-seam` |
| Merged | 2026-07-06 |
| Merge SHA | `4a28bfc` |
| Repo | `blueman82/coderails` |
| Sub-project | 4 of 5 in the agentic-OS evolution sequence (observability → routines → workflow-audit → assistant-agent kernel integration → improvement loops) |

## Summary

A single new design/spec doc — `docs/coderails/specs/2026-07-06-assistant-link-panel-design.md` — no code. Defines the on-disk `QueueFileEntry` contract that assistant-agent's send-approval gate (PR assistant-agent#4) writes to and a future dashboard Approve/Deny button reads/mutates, plus the four-item ASSISTANT.LINK panel spec (D6). Written **normative-first**: at merge time assistant-agent's `gate/surfaces/queue.ts` didn't exist yet, so this document is the contract WU2 had to conform to, not a description of code already written. `skills/dashboard/` is explicitly untouched by this PR — the Approve/Deny button itself is deferred (see [[assistant-link-send-gate-architecture]]).

## Files changed

- `docs/coderails/specs/2026-07-06-assistant-link-panel-design.md` — the contract + panel spec (229 lines)

## Wiki pages updated

- [[assistant-link-send-gate-architecture]] — new design page covering the send-gate architecture, the queue seam this PR defines, and the stale-plugin-cache finding
- [[dashboard]] — cross-referenced; the "reserved slot" language this PR's panel spec targets
- [[pr_31_assistant-link-approve-button]] — the follow-up PR that implements the reader/writer half of this contract

## Caveats / gotchas

- Normative direction matters: if a future implementation diverges from this document's `QueueFileEntry` shape, that is a bug in the implementation, not staleness in this doc — the doc should only change if the owner explicitly re-resolves the contract.
- Three of the four panel slots (tasks due/overdue, email-last-checked, routine-runs) remain **unbuilt dependencies** as of this PR — only "sends + approvals log" (queue-backed) has a real data source. The panel spec deliberately does not fabricate a schema for the other three; see PR #31's implementation, which renders only the queue slice.
- `status` in `QueueFileEntry` has no `expired`/`timeout` value — a gate-side timeout is recorded in the audit log (JSONL), not the queue file; a stale `pending` file is left as-is pending a future (unbuilt) cleanup routine.
