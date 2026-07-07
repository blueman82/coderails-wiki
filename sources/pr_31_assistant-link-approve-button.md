---
title: "PR 31 — ASSISTANT.LINK dashboard Approve/Deny button"
type: source
created: 2026-07-07
last_updated: 2026-07-07
sources:
  - sources/pr_28_assistant-link-queue-contract-and-panel-spec.md
tags: [source, dashboard, assistant-agent, queue, security-fix, agentic-os, sub-project-4-of-5]
---

# PR 31 — ASSISTANT.LINK dashboard Approve/Deny button

Ingested by /wiki-ingest after merge. This is an immutable record of what changed.

## PR metadata

| Field | Value |
|---|---|
| PR number | #31 |
| Branch | `feat/wu5-approve-button` |
| Merged | 2026-07-06 |
| Merge SHA | `6fa5e34` |
| Repo | `blueman82/coderails` |
| Sub-project | 4 of 5 in the agentic-OS evolution sequence |

## Summary

Implements the dashboard-side half of the queue contract PR #28 defined: a `collectQueue` collector (`skills/dashboard/app/src/lib/collect/queue.ts`), a new `AssistantLinkPanel` component rendering only the "sends + approvals log" pending-queue slice (the other three D6 panel slots stay unbuilt, per PR #28's own scoping), and a `POST /api/queue` endpoint performing an **in-place JSON status rewrite** of the target `<hash>.json` file — the simplest reading of "the only writer of the approved/denied transition," pre-authorized rather than inventing a separate decision-file mechanism. Zero files touched outside `skills/dashboard/`.

## Process

9/9 Tier-2 evals GO. Review caught **one Critical finding pre-merge**: the queue action endpoint's `resolveQueueEntry` built its file path via `join(queueDir, hash + ".json")` without validating `hash`'s shape first — a `hash` containing `../` segments could escape `queueDir` (path traversal). Fixed by validating `hash` against `/^[0-9a-f]{64}$/` (strictly 64 lowercase hex characters) at **two layers**: the API route (`skills/dashboard/app/src/app/api/queue/route.ts`) rejects a malformed hash before calling in, and `queueActions.ts`'s `resolveQueueEntry` itself re-validates rather than trusting the caller — documented as defense-in-depth, not redundant. assistant-agent's `hashInput` (`gate/sendGate.ts`, `createHash("sha256").update(canonicalise(toolInput)).digest("hex")`) was confirmed to conform to this pattern.

## Files changed

- `skills/dashboard/app/src/lib/collect/queue.ts` + `queueActions.ts` — collector (list+parse, degrade to `[]`) and the approve/deny mutator (`resolveQueueEntry`, throws `QueueActionError`, never silently succeeds or defaults to approved)
- `skills/dashboard/app/src/app/api/queue/route.ts` — `POST /api/queue`, token + Origin/Host guarded (mirrors `api/run/route.ts`'s pattern), hash-pattern-validated
- `skills/dashboard/app/src/components/AssistantLinkPanel.tsx` — renders pending entries; `toolInput` shown only via `JSON.stringify` preview, never destructured (queue is generic across all gated tools)
- `skills/dashboard/app/src/components/{DashboardProvider,RailRight}.tsx`, `src/hooks/useDashboardState.ts`, `src/lib/collect/index.ts` — wiring (queue into `Snapshot`/`AggregatorDeps`, panel mounted in `RailRight` replacing the old "Reserved — Sub-Project 4" placeholder row)
- `skills/dashboard/app/test/*` — collector, queue-action, route, and panel tests (including `DashboardContextTestProvider.tsx`, a new test-only fixed-snapshot provider; `DashboardContext` itself was exported for this purpose)

## Wiki pages updated

- [[assistant-link-send-gate-architecture]] — the path-traversal fix and the hash-validation contract it depends on
- [[dashboard]] — "reserved slot" language superseded; panel now real, scoped to one of the four D6 items

## Caveats / gotchas

- **Residual debt (owner-flagged, not fixed by this PR):** no live browser E2E against the real send-gate exists — only unit/component tests with a fixed test snapshot. A concurrent approve/deny race (two requests for the same hash) is also untested; `resolveQueueEntry`'s read-modify-write is not atomic.
- The panel deliberately renders only entries with `status === "pending"`; a component test explicitly asserts approved/denied entries are excluded from that view and that the three unbuilt D6 slots (tasks, email, routines) do not leak into this component's output.
- `DashboardContext` (previously module-private) is now exported for test use only — production code must still go through `DashboardProvider`/`useDashboardContext`, never the raw context.
