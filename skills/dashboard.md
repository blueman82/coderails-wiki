---
title: "Skill: dashboard"
type: skill
created: 2026-07-06
last_updated: 2026-07-07
sources:
  - sources/pr_25_observability-dashboard.md
  - sources/pr_43-44-46_workflow-audit-queue-seam.md
  - sources/pr_36-41-33-53-65_verified-routines.md
  - sources/pr_55-60-64-66-67_approve-build-runner.md
  - sources/pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements.md
tags: [skill, dashboard, observability, nextjs, r3f, sse, obsidian, agentic-os, sub-project-1-of-5, queue-contract, builder, ask-button, argv]
---

# Skill: dashboard

Launches the coderails observability dashboard — a live local web HUD showing sessions, agentic loops, PR gate states, runs, and memory activity — plus a companion Obsidian command centre. Sub-project 1 of a 5-part agentic-OS evolution sequence (observability → routines → workflow-audit → assistant-agent kernel integration → improvement loops); see the design spec's decisions log for the other four.

Source: `coderails/skills/dashboard/SKILL.md`
Design spec: `coderails/docs/coderails/specs/2026-07-06-observability-dashboard-design.md`
Plan: `coderails/docs/coderails/plans/2026-07-06-observability-dashboard-plan.md`
Invoked as: `coderails:dashboard`

## What it shows

Seven panels, all reading state the kernel already produces (no new services, no telemetry leaving the machine): SYSTEM VITALS (usage/hooks/lint sparklines), DIRECTIVES (active loop's `progress.json` work units as a checklist), DOCUMENTS/MEMORY.TRAIL (wiki + memory mtime feed), COMMAND DECK (declared one-click buttons + run history), PR GATES (merge-ready / blocked / stale, from the marker-grammar libs), a bottom-centre hero stat, and ASSISTANT.LINK — originally a reserved placeholder for sub-project 4 (assistant-agent kernel integration), now a real panel (coderails PR #31) rendering the pending send-approval queue for assistant-agent's send-gate; see [[assistant-link-send-gate-architecture]].

## Architecture

One Next.js process serves both the React Three Fiber HUD frontend and the API routes: an SSE stream (`/api/events`) and a token + Origin-guarded run trigger (`/api/run`). Binds `127.0.0.1` only. Collectors read: `~/.claude/projects/*` session dirs, agentic-loop `progress.json` (via the existing `agentic_loop_path.sh` SSOT — see [[agentic-loop]]), `gh`-polled PR state parsed with the same marker grammar as `review-artifact.sh`/`eval-artifact.sh` (see [[review-artifact-seam]] and [[task-evals-gate]]), hook logs, and wiki/memory mtimes.

### Button / run model — the security-load-bearing piece

A button is a declared, bounded run, never a free prompt box. `POST /run` looks up a button name in `~/.claude/coderails-dashboard.json` and refuses anything undeclared. `buildArgv()` in `src/lib/argv.ts` is the **single** profile→flag mapping (`read-only` → `--allowedTools Read Grep Glob`; `standard` → inherit target project's allowlists; `bypass` → `--dangerously-skip-permissions`, opt-in per button, warning badge in both UIs) — the Obsidian plugin must reuse it, never re-implement it.

**Flag-smuggling closed by two independent layers**, found and fixed during review (Tier-2 eval / review cycle, not caught by the original implementation): a leading-dash input like `--dangerously-skip-permissions` would otherwise be parsed by the `claude` CLI as a real flag rather than literal prompt text (confirmed empirically: `claude -p "--version"` prints the version banner instead of answering the prompt). The fix: (1) `input` starting with `-` (checked against the **trimmed** value, so whitespace can't hide a leading dash from this check — closed 2026-07-07, PR #70) is rejected outright before any sentinel is trusted, and (2) a literal `--` end-of-options sentinel is always inserted before the prompt regardless, confirmed on-machine to make the CLI treat everything after it as literal text. Neither layer alone was judged sufficient — the review explicitly wanted the reject-outright check as a second wall, not just the sentinel.

**Input-delivery bug, found and fixed 2026-07-07 (PR #70 — see [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]):** the original argv shape pushed `input` as **a separate positional argv element** after `btn.command` and the sentinel (`["-p", btn.command, ...profileFlags, "--", input]`). The `claude` CLI's `-p`/`--print` only consumes ONE positional prompt — it never merges a second positional into the prompt already consumed, so `input` silently never reached the model on **any** `inputAllowed` button, since the button model's inception (`verify-q`, `deep-research`). Caught by an intake smoke test observing the *live* dashboard's response, not by any argv-shape unit test — every prior test checked shape only, none checked that input was actually delivered, so the pre-fix code passed all of them. Fixed by merging `command` and `input` into **one** prompt string before the sentinel: `btn.command ? \`${btn.command} ${input}\` : input`, giving the current shape `["-p", ...profileFlags, "--", mergedPrompt]` when input is present (profile flags precede the sentinel; the merged prompt is the sole positional after it). Without input, the original single-token shape is unchanged. Empty/whitespace-only input is now normalised to "no input" inside `buildArgv` itself (not just at the UI layer), and `buildArgv` throws on an empty effective prompt (no command and no input) — `route.ts`'s existing catch surfaces this as a 400.

A new **"ask" button pattern** follows directly from this fix: `command: ""`, `profile: "standard"`, `inputAllowed: true` — free text becomes the entire prompt inside the button's declared envelope (profile flags still apply). Live in the owner's per-user config; documented in `examples/dashboard-config.json` (4th button, first time exercised by `lib/test/config.test.ts`).

Every run is single-flight per button, JSONL-recorded (`~/.claude/coderails-dashboard/runs/`) with argv/cwd/profile/exit code, and CSRF-token-guarded (both `/api/events` and `/api/run` also reject non-localhost `Origin`/`Host`).

### AssistantLinkPanel — per-producer readable rendering

`AssistantLinkPanel.tsx` (PR #31 — full build-out in
[[pr_31_assistant-link-approve-button]]) renders the queue directory's
pending entries via the same collector/Approve-Deny path regardless of
producer, treating `toolInput` opaquely by default (`JSON.stringify` +
truncate). As of [[pr_43-44-46_workflow-audit-queue-seam]] (PR #44), a
type-guarded render branch recognises `toolName ===
"workflow-audit:propose-skill"` (the [[workflow-audit]] skill's queue-mode
output) and displays it readably — proposed name / description / task
summary / session count — instead of the opaque fallback, which still covers
every other `toolName` (e.g. send-gate entries, per
[[assistant-link-send-gate-architecture]]). The Approve/Deny buttons and
`POST /api/queue` path are unchanged by this branch; only the preview differs
per producer.

**Build-state visibility (added by [[pr_55-60-64-66-67_approve-build-runner]],
PR #66):** a new `collectBuilds` collector reads `~/.claude/coderails-dashboard/builds/<hash>/state.json`
sidecars (closed-set `state` validation mirroring `queue.ts`'s discipline —
reject unknown/missing, never default), exposed as `Snapshot.builds`. The
panel hash-joins each approved queue entry to its build sidecar and renders
one of building / awaiting-your-merge / failed / builder-dead (on a stale
heartbeat). Includes a real XSS fix: `prUrl` is builder-session-controlled
data (read verbatim from `run-builder.sh`'s own `gh pr create` output, not
dashboard-generated), so a `safePrUrl()` guard only links `https:`-protocol
URLs — a `javascript:`/`data:` scheme would otherwise be a click-triggered
vector — falling back to a plain-text CTA otherwise.

### Obsidian command centre

A native Obsidian plugin (official TypeScript template) registering a code-block processor (`agentic-os`) that renders dashboard state inside a real markdown note, sharing the same button config as the web deck. The routines sub-project (#2, now shipped — see [[intent-queue-runner-contract]] and [[dashboard-runner]]) defines the queue+runner contract this seam was frozen against; the plugin writes intent files to `~/.claude/coderails-dashboard/queue/` per that contract, but still also keeps its interim direct-exec path — [[dashboard-runner]] existing doesn't yet make the plugin stop invoking `claude` itself, so the plugin has not yet been updated to rely solely on the now-real runner. Ships a committed, reproducible `dist/main.js` build (same precedent as `wiki-init`'s committed Marp assets).

## Starting / stopping

`skills/dashboard/scripts/start-dashboard.sh` (npm ci → build → start → open, idempotent re-launch) and `stop-dashboard.sh` (kills the pidfile'd process). Port overridable via `DASHBOARD_PORT`.

## Process record (this PR)

16 SDD tasks, each with an independent review. Five fix loops closed real findings: a lock TOCTOU (Critical), marker grammar drift, `hooksFired` today-scoping, a single-SSE-provider requirement, and the flag-smuggling vector above (High). The frozen Tier-2 eval suite (10 evals, required because this PR is both ≥3 work-units and touches an outward/irreversible surface — see [[task-evals]]'s tier rules) caught **two production bugs every review round missed**: the launch script reported false-success against a squatted port, and a statically-prerendered page baked an empty button config into the HTML (fixed by forcing the route dynamic). Both eval and review artifacts were posted SHA-bound on the PR and consumed by the merge gates ([[task-evals-gate]], [[review-artifact-seam]]).

## Why this matters beyond the feature

This is the first sub-project to give the task-evals gate a real production catch it can point to — prior task-evals coverage was about the gate's own mechanics (oracle independence, tier justification), not a demonstrated bug caught in the wild. See [[task-evals]] and [[task-evals-gate]] for the mechanism this exercised.

## See also

- [[agentic-loop]] — the loop this session ran under; registered with a loop-scope `evals.json` (Phase 2.7c), GO 4/4
- [[task-evals]] / [[task-evals-gate]] — the pr-scope Tier-2 gate this PR both used and validated with two real catches
- [[unregistered_loop_guard]] — born from a gap found in *this session's own* orchestration (see PR #17 in [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]]); linked here as the guard this kind of large multi-task session is designed to trip if it drifts unregistered
- [[review-artifact-seam]] — the SHA-bound review-comment gate this PR's merge consumed
- [[pr_25_observability-dashboard]] — the source record for this PR
- [[workflow-audit]] — sub-project 3 of the same agentic-OS evolution sequence this dashboard is sub-project 1 of; now a second queue producer via [[pr_43-44-46_workflow-audit-queue-seam]]
- [[pr_43-44-46_workflow-audit-queue-seam]] — the queue-mode integration source page (writer, `AssistantLinkPanel` render branch, consumption-seam contract)
- [[pr_55-60-64-66-67_approve-build-runner]] — the Approve-click → skill-creator builder pipeline: claim/spawn seam, wrapper state machine, injection-fenced prompt, build-state visibility, and the `safePrUrl` XSS fix
- [[assistant-link-send-gate-architecture]] — sub-project 4's send-gate design, the queue seam this panel reads/mutates, and the ASSISTANT.LINK panel's four D6 slots (only "sends + approvals log" has a real data source so far)
- [[pr_28_assistant-link-queue-contract-and-panel-spec]] / [[pr_31_assistant-link-approve-button]] — the contract spec and the panel's implementation + path-traversal fix
- [[intent-queue-runner-contract]] / [[dashboard-runner]] / [[routines]] / [[memory-consolidation]] — sub-project 2, the routines cluster: the queue schema/lifecycle, the sole-executor runner, the scheduling convention, and one shipped routine's skill
- [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] — the argv merged-prompt fix (button input was silently dropped on every `inputAllowed` button since inception) and the "ask" button pattern this fix enabled
- [[voice_announce]] — the sibling PR in the same cluster: a new observe-only Stop hook announcing loop lifecycle events via macOS `say`, unrelated to the dashboard's own code but merged in the same session
