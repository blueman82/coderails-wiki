---
title: "ASSISTANT.LINK send-gate architecture"
type: design
created: 2026-07-07
last_updated: 2026-07-07
sources:
  - sources/pr_28_assistant-link-queue-contract-and-panel-spec.md
  - sources/pr_31_assistant-link-approve-button.md
  - sources/pr_62_10_approvals-dir-move.md
  - sources/pr_36-41-33-53-65_verified-routines.md
tags: [design, assistant-agent, send-gate, queue, approvals, dashboard, agentic-os, sub-project-4-of-5, plugin-cache]
---

# ASSISTANT.LINK send-gate architecture

How assistant-agent (Gary's personal secretary, `~/Github/assistant-agent`) puts the coderails kernel's enforcement discipline in front of its own send-capable tools (Slack, Calendar, Gmail), and how that gate's queue surface connects to the coderails dashboard.

## Context

Sub-project 4 of the agentic-OS evolution sequence puts the secretary on the coderails kernel: installs the plugin, aligns the assistant-agent wiki to the coderails wiki schema, and — the highest-leverage piece — promotes the secretary's prompt-level "confirm before send" rules to a deterministic enforcement mechanism. This page documents the two load-bearing architecture decisions and the queue seam that connects assistant-agent to the dashboard (sub-project 1, [[dashboard]]).

## The rule

**The send-gate lives inside the host process as an SDK `PreToolUse` hook callback, not as a bash hook script — because a bash `PreToolUse` hook shares the agent's own user/filesystem and is forgeable by the agent itself.** A bash hook could be defeated by an agent editing an approval-marker file it also controls; the host-process callback runs outside the agent's trust domain entirely, on a `permissionDecision` control-plane path that does not depend on `allowedTools`.

**The gate must self-enforce fail-closed, because the SDK is fail-open on hook error.** A live spike (assistant-agent PR #4) proved the SDK does **not** cut off a hook that throws mid-execution or exceeds its matcher-level timeout — in both cases, absent the hook wrapping itself, the tool call would proceed unapproved. `gate/sendGate.ts`'s entire body runs inside a `try`/`catch` that resolves to deny on any exception (verified, `gate/sendGate.ts`), and an internal deny-timeout race (`INTERNAL_DENY_TIMEOUT_MS`, 60s, strictly shorter than any matcher-level timeout) is the actual enforcement of fail-closed-on-timeout — not the SDK's own timeout handling, which does not fail closed on its own.

These two facts compound: neither is sufficient alone. A host-process hook that trusted the SDK to fail closed on error would still be bypassable by a thrown exception; a bash hook that self-wrapped in try/catch would still be forgeable by the agent it's meant to constrain. The architecture needs both.

## Gate semantics (D2)

- **Per-item, exact-input-bound approval.** An approval is bound to the sha256 hash of the canonicalised (sorted-key, deep-recursive `JSON.stringify`) tool input (`gate/sendGate.ts` `hashInput`/`canonicalise`). Approving send X does not approve send Y, even for the same tool.
- **One-shot.** An approval is consumed on use; an identical resend requires a fresh approval — no replay.
- **Fail-closed.** No approval within the internal timeout denies, with a message telling the agent to redraft or ask.
- **Gated tool list** (verified, `gate/sendGate.ts` `GATED_TOOL_NAMES`): `mcp__claude_ai_Slack__slack_send_message`, and the Google Calendar `create_event`/`update_event`/`delete_event`/`respond_to_event` tools. No Gmail send tool exists live (confirmed at build time) — nothing to gate there yet. Draft-creation tools (`slack_send_message_draft`, Gmail `create_draft`) are deliberately **ungated** — gating a draft breaks the draft-first UX for zero security benefit.
- **Non-MCP send routes are a stated residual risk, not silently ignored.** Bare `Bash` (e.g. `curl` to `slack.com/api`) and `mcp__claude-in-chrome__*` (driving the Slack/Gmail web UI) are both send-capable and stay allowed by tool-name matching alone. The same `PreToolUse` hook additionally pattern-matches `Bash` commands against known send-API hosts (`slack.com/api`, `api.telegram.org`, Gmail/Calendar `googleapis.com` endpoints) and denies with a redirect-to-MCP message — but the browser-automation route is **not** preventable by pattern-matching and is documented as an accepted residual, with the audit log as the detection layer rather than a prevention layer.
- **Every attempt and decision is audited.** An append-only JSONL log records event type (`attempt`/`decision`), tool name, hash, surface, and timestamp for every gated call — this is also the data source for the dashboard's "sends + approvals log" panel slot (see the queue seam below).

## The queue seam (dashboard connection)

The gate's third approval surface (alongside terminal y/n and a Telegram bot) is a **dashboard queue**: a pending-approval JSON file written to `~/.claude/coderails-dashboard/approvals/<hash>.json`, one file per pending-or-resolved approval, named by the same hex sha256 hash the approval binding uses.

**Moved from `queue/` to `approvals/` on 2026-07-07** ([[pr_62_10_approvals-dir-move]]): `queue/` is shared by the routines runner sweep and workflow-audit's `propose-skill` proposals ([[pr_43-44-46_workflow-audit-queue-seam]]), both of which glob every `.json` file there and quarantine/reject anything that isn't their own expected shape — a `QueueFileEntry` approval file looked like malformed input to both. `approvals/` is a sibling directory, not nested inside `queue/`; the split is a clean break with no dual-dir fallback.

```typescript
export interface QueueFileEntry {
  hash: string;           // sha256(canonicalise(toolInput)), hex — also the filename stem
  toolName: string;
  toolInput: unknown;     // opaque — never destructured by assumed shape
  createdAt: number;      // epoch ms
  status: "pending" | "approved" | "denied";
}
```

This shape was defined **normative-first** in coderails PR #28 (design doc only, no code) before assistant-agent's `gate/surfaces/queue.ts` existed — the contract WU2 had to conform to, not a description of code already written. `status` has no `expired`/`timeout` value: a gate-side timeout is recorded in the audit log, not the queue file; a stale `pending` file is left as-is pending a future cleanup routine (unbuilt).

**Writer/reader split:**
- The secretary's gate (`gate/surfaces/queue.ts`) is the **only writer** of `pending` entries, and polls its own written file for `status` to change.
- The dashboard's `collectQueue` (`skills/dashboard/app/src/lib/collect/queue.ts`, coderails PR #31) lists and parses the directory for the panel, degrading to `[]` on any read error — same idiom as `collectMemoryTrail`.
- The dashboard's `POST /api/queue` (coderails PR #31) is the **only writer** of the `approved`/`denied` transition, via an **in-place JSON rewrite** of the target file's `status` field (`resolveQueueEntry`) — not a separate decision-file mechanism. This was the pre-authorized simplest reading of the contract.

### Path-traversal fix (PR #31, Critical, fixed pre-merge)

`resolveQueueEntry` originally built its file path via `join(queueDir, hash + ".json")` without validating `hash`'s shape — a `hash` containing `../` segments could escape `queueDir` entirely. Fixed with a strict pattern, enforced at **two layers**: the API route rejects a malformed hash before calling in, and `resolveQueueEntry` itself re-validates rather than trusting the caller (defense-in-depth, not redundant, per the fix's own comment):

```typescript
const HASH_PATTERN = /^[0-9a-f]{64}$/;
```

assistant-agent's `hashInput` (`createHash("sha256").update(canonicalise(toolInput)).digest("hex")`) was confirmed to conform to this pattern — `digest("hex")` always produces exactly 64 lowercase hex characters, so a legitimate hash can never itself fail the check.

## The stale-plugin-cache finding

Separate from (but discovered in the same sub-project as) the send-gate work: assistant-agent's live hook-fire probe found the coderails plugin **as installed** in `~/.claude/plugins/cache/coderails/coderails/` was stale — version `1.0.0` (installed 2026-06-24, `autoUpdates: false` at the time), missing `no_edit_on_main.sh`, `enforce_pr_workflow.sh`, and the loop-guard hooks that the coderails **source checkout** at `~/Github/coderails` (version 1.1.0) already had. This is a different mechanism from [[install-and-cache-trap]]'s repo-vs-cache gap (which is about coderails' *own* local-directory dev loop): this is a **marketplace-installed, auto-updating plugin consumer** (assistant-agent) running against whatever cached snapshot the marketplace last delivered, which can silently lag the current source by weeks with no local signal.

The plugin self-updated 1.0.0 → 1.1.0 overnight on 2026-07-07, closing the gap without any manual action. assistant-agent's `hooks/scripts/tests/probe_conventions.test.sh` — the regression test written to document the original finding — initially hardcoded the `1.0.0` path and asserted the two hooks MISSING; that assertion silently kept validating the superseded path even after the update, because the old `1.0.0` cache directory continues to coexist on disk alongside `1.1.0` (both are simply present under the cache root). Fixed (assistant-agent PR #6) to resolve the active install path dynamically via `jq` against `installed_plugins.json`'s `.plugins["coderails@coderails"][0].installPath`, with a version-sorted (`sort -V`, not mtime — cache directories can share an identical install timestamp) fallback if jq or the key is unavailable. Detection of this class of drift now lives permanently in assistant-agent's own test suite, rather than being a one-time finding.

**General lesson, extending [[install-and-cache-trap]]'s theme to marketplace-installed consumers:** a plugin's installed cache and its published source can diverge for any consumer, not just coderails' own dev loop — and unlike the local-directory case, an auto-updating marketplace install can heal itself silently, which means a test asserting "the gap exists" needs to re-validate against the *live* install path each run, not a path frozen at the time the gap was discovered.

**Addendum (found by the orchestrator, 2026-07-07): the same 1.0.0→1.1.0 update also silently changed the agentic-loop state key.** The 1.0.0 cache predates [[agentic-loop-path-keying|PR #24]] entirely — verified: it has no `agentic_loop_path.sh` file at all, so any session running under it used the older cwd-only slug. Once the plugin auto-updated to 1.1.0, `progress.json`'s path authority switched to the git-common-dir-keyed slug (e.g. `-Users-harrison-Github-coderails-.git` instead of the plain cwd slug), and — unlike the same-version mid-loop re-key case [[agentic-loop-path-keying]] already documents — there is no loop-side signal that a re-key is imminent on a plugin upgrade. A session with a faithfully maintained `progress.json` under the old slug was reported as an *unregistered* loop by the newer guards (now live post-update) despite doing nothing wrong. Workaround applied live: a symlink from the new slug to the old state directory. This is a **migration gap, not fixed here** — see [[agentic-loop-path-keying]]'s own caveats section for the full detail and candidate fixes (legacy-slug fallback, or a plugin-update migration step).

## Where it is enforced

- `~/Github/assistant-agent/gate/sendGate.ts` — `createSendGateHook`, the `PreToolUse` callback; self-wrapped fail-closed try/catch.
- `~/Github/assistant-agent/secretary.ts` (lines ~9-12, ~67, ~122-129, verified) — wires `sendGateHook` into the SDK's `hooks: { PreToolUse: [...] }` option alongside the three approval surfaces (terminal, Telegram, queue).
- `~/Github/assistant-agent/gate/bashPatterns.ts` — the non-MCP Bash send-API pattern layer.
- `~/Github/assistant-agent/gate/auditLog.ts` — the append-only JSONL audit log.
- `skills/dashboard/app/src/lib/collect/queue.ts` + `queueActions.ts` (coderails PR #31) — the dashboard-side reader/writer.
- `~/Github/assistant-agent/hooks/scripts/tests/probe_conventions.test.sh` (assistant-agent PR #6) — the stale-cache detection, now dynamic.

## Known caveats / edge cases

- **Browser-automation send route is an accepted residual risk**, not prevented — see the non-MCP send routes note above. The audit log is the detection layer for this specific gap.
- **No live browser E2E exists** against the real send-gate for the dashboard Approve/Deny button — only unit/component tests against a fixed test snapshot (owner-flagged residual debt, PR #31).
- **Concurrent approve/deny race is untested** — `resolveQueueEntry`'s read-modify-write of the queue file is not atomic (owner-flagged residual debt, PR #31).
- **Three of the four ASSISTANT.LINK panel slots remain unbuilt dependencies**: tasks due/overdue needs `assistant-agent/tasks/*.md` to actually be populated before a parse format can be derived (the directory was empty at spec time); email-last-checked needs a secretary-side sweep-timestamp state file that no work unit in this sub-project commits to building; routine-runs is explicitly deferred to sub-project 2. Only "sends + approvals log" (queue-backed) has a real data source as of this cluster.

## See also

- [[install-and-cache-trap]] — the sibling repo-vs-cache phenomenon for coderails' own local-directory dev loop; this page's stale-plugin-cache finding is the marketplace-consumer analogue
- [[agentic-loop-path-keying]] — the loop-state re-keying gap the same plugin update triggered; documents the mechanism (`--git-common-dir` slug vs. legacy cwd slug) and the workaround/candidate fixes
- [[enforcement-model]] — the hooks-vs-advisory law this gate follows (host-process hook chosen specifically because a bash hook's enforcement ceiling would be forgeable here)
- [[dashboard]] — sub-project 1; the panel this gate's queue surface feeds
- [[pr_28_assistant-link-queue-contract-and-panel-spec]] — the normative queue contract + panel spec
- [[pr_31_assistant-link-approve-button]] — the dashboard-side implementation + path-traversal fix
- [[pr_62_10_approvals-dir-move]] — the `queue/` → `approvals/` split and why it was needed
