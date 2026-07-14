---
title: "Skill: dashboard"
type: skill
created: 2026-07-06
last_updated: 2026-07-12
sources:
  - sources/pr_25_observability-dashboard.md
  - sources/pr_43-44-46_workflow-audit-queue-seam.md
  - sources/pr_36-41-33-53-65_verified-routines.md
  - sources/pr_55-60-64-66-67_approve-build-runner.md
  - sources/pr_89-100-104-106_approve-build-e5-live.md
  - sources/pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements.md
  - sources/pr_80-82_dashboard-stream-run-output-viewer.md
  - sources/pr_88_93_dashboard-launchd.md
  - sources/pr_86-107_2026-07-08_loop-lib-residuals.md
  - sources/pr_124_dashboard-run-output-result-extraction.md
  - sources/pr_128_dashboard-ask-button-auto-profile.md
  - sources/pr_130-136_dashboard-right-rail-ux.md
  - sources/pr_139-141_dashboard-ask-enter-clean-output.md
  - investigations/dashboard-lockfile-emnapi-drift_2026-07-08.md
  - sources/pr_144-149_agentic-loop-hardening-from-loop-engineering.md
  - sources/pr_163-168_dashboard-rethink.md
tags: [skill, dashboard, observability, nextjs, r3f, sse, obsidian, agentic-os, sub-project-1-of-5, queue-contract, builder, ask-button, argv, run-output, streaming, launchd, reboot-persistence, npm-ci, lockfile, permission-mode, enter-to-submit, decisions-absorbed, loop-decisions-tile, deck-trim, multi-loop, gates-freshness, headless-exemption]
---

# Skill: dashboard

Launches the coderails observability dashboard — a live local web HUD showing sessions, agentic loops, PR gate states, runs, and memory activity — plus a companion Obsidian command centre. Sub-project 1 of a 5-part agentic-OS evolution sequence (observability → routines → workflow-audit → assistant-agent kernel integration → improvement loops); see [[routines]], [[workflow-audit]], [[assistant-link-send-gate-architecture]], and [[pr_118-123_self-improving-loops]] for the other four.

Source: `coderails/skills/dashboard/SKILL.md`
Invoked as: `coderails:dashboard`

> Design spec and plan file paths formerly cited here (`docs/coderails/specs/2026-07-06-observability-dashboard-design.md`, `docs/coderails/plans/2026-07-06-observability-dashboard-plan.md`) no longer exist — removed from repo tracking by [[pr_138_remove-specs-plans-tracking]] (2026-07-11). This page's own content is now the durable record.

## What it shows

Seven panels, all reading state the kernel already produces (no new services, no telemetry leaving the machine): SYSTEM VITALS (usage/hooks/lint sparklines), DIRECTIVES (active loop's `progress.json` work units as a checklist, plus — since PR #148/#149 — the loop's last 5 `decisions_absorbed` entries), DOCUMENTS/MEMORY.TRAIL (wiki + memory mtime feed), COMMAND DECK (declared one-click buttons + run history), PR GATES (merge-ready / blocked / stale, from the marker-grammar libs), a bottom-centre hero stat, and ASSISTANT.LINK — originally a reserved placeholder for sub-project 4 (assistant-agent kernel integration), now a real panel (coderails PR #31) rendering the pending send-approval queue for assistant-agent's send-gate; see [[assistant-link-send-gate-architecture]].

## Architecture

One Next.js process serves both the React Three Fiber HUD frontend and the API routes: an SSE stream (`/api/events`) and a token + Origin-guarded run trigger (`/api/run`). Binds `127.0.0.1` only. Collectors read: `~/.claude/projects/*` session dirs, agentic-loop `progress.json` (via the existing `agentic_loop_path.sh` SSOT — see [[agentic-loop]]), `gh`-polled PR state parsed with the same marker grammar as `review-artifact.sh`/`eval-artifact.sh` (see [[review-artifact-seam]] and [[task-evals-gate]]), hook logs, and wiki/memory mtimes.

### Button / run model — the security-load-bearing piece

A button is a declared, bounded run, never a free prompt box. `POST /run` looks up a button name in `~/.claude/coderails-dashboard.json` and refuses anything undeclared. `buildArgv()` in `src/lib/argv.ts` is the **single** profile→flag mapping (`read-only` → `--allowedTools Read Grep Glob`; `standard` → inherit target project's allowlists; `auto` → `--permission-mode auto`, added [[pr_128_dashboard-ask-button-auto-profile|PR #128]]; `bypass` → `--dangerously-skip-permissions`, opt-in per button, warning badge in both UIs) — the Obsidian plugin must reuse it, never re-implement it.

**`auto` closes a headless-only hang, not tool-call non-determinism ([[pr_128_dashboard-ask-button-auto-profile|PR #128]], 2026-07-10).** In headless `claude -p` mode, `standard` adds no permission flags, so a tool needing a fresh permission grant (e.g. `WebSearch`) blocks indefinitely on an unanswerable prompt — the model then silently falls back to answering from memory with no tool call ever attempted. `--permission-mode auto` auto-approves the permission decision instead of blocking on it, closing that specific hang. It does **not** fix the separate, already-documented non-determinism of *whether the model attempts the tool call at all*: direct empirical testing (both in the PR's own `argv.ts:40-49` code comment and independently reproduced this session) found only 1 real tool call in 3 consecutive identical runs, the other 2 falling back to "I don't have internet access" with no attempt and no permission-block message. See [[pr_128_dashboard-ask-button-auto-profile]] for the full finding.

**Flag-smuggling closed by two independent layers**, found and fixed during review (Tier-2 eval / review cycle, not caught by the original implementation): a leading-dash input like `--dangerously-skip-permissions` would otherwise be parsed by the `claude` CLI as a real flag rather than literal prompt text (confirmed empirically: `claude -p "--version"` prints the version banner instead of answering the prompt). The fix: (1) `input` starting with `-` (checked against the **trimmed** value, so whitespace can't hide a leading dash from this check — closed 2026-07-07, PR #70) is rejected outright before any sentinel is trusted, and (2) a literal `--` end-of-options sentinel is always inserted before the prompt regardless, confirmed on-machine to make the CLI treat everything after it as literal text. Neither layer alone was judged sufficient — the review explicitly wanted the reject-outright check as a second wall, not just the sentinel.

**Input-delivery bug, found and fixed 2026-07-07 (PR #70 — see [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]):** the original argv shape pushed `input` as **a separate positional argv element** after `btn.command` and the sentinel (`["-p", btn.command, ...profileFlags, "--", input]`). The `claude` CLI's `-p`/`--print` only consumes ONE positional prompt — it never merges a second positional into the prompt already consumed, so `input` silently never reached the model on **any** `inputAllowed` button, since the button model's inception (`verify-q`, `deep-research`). Caught by an intake smoke test observing the *live* dashboard's response, not by any argv-shape unit test — every prior test checked shape only, none checked that input was actually delivered, so the pre-fix code passed all of them. Fixed by merging `command` and `input` into **one** prompt string before the sentinel: `btn.command ? \`${btn.command} ${input}\` : input`, giving the current shape `["-p", ...profileFlags, "--", mergedPrompt]` when input is present (profile flags precede the sentinel; the merged prompt is the sole positional after it). Without input, the original single-token shape is unchanged. Empty/whitespace-only input is now normalised to "no input" inside `buildArgv` itself (not just at the UI layer), and `buildArgv` throws on an empty effective prompt (no command and no input) — `route.ts`'s existing catch surfaces this as a 400.

A new **"ask" button pattern** follows directly from this fix: `command: ""`, `inputAllowed: true` — free text becomes the entire prompt inside the button's declared envelope (profile flags still apply). Live in the owner's per-user config; documented in `examples/dashboard-config.json` (4th button, first time exercised by `lib/test/config.test.ts`). Originally shipped on `profile: "standard"`; switched to `profile: "auto"` by [[pr_128_dashboard-ask-button-auto-profile|PR #128]] (2026-07-10) to avoid `standard`'s headless permission-block hang on tools like `WebSearch` — see the caveat above on what `auto` does and does not fix.

Every run is single-flight per button, JSONL-recorded (`~/.claude/coderails-dashboard/runs/`) with argv/cwd/profile/exit code, and CSRF-token-guarded (both `/api/events` and `/api/run` also reject non-localhost `Origin`/`Host`).

### Run Output viewer — live-streaming + settled playback (PR #80–82)

The COMMAND DECK's 4th panel, `OutputViewerPanel.tsx`, closes a gap
[[dashboard-run-log-streaming-viewer-gap_2026-07-07]] documented the same day: the per-run
`.log` file used to be write-only (one post-exit `appendFileSync`), never read anywhere. As of
[[pr_80-82_dashboard-stream-run-output-viewer]], `POST /api/run` spawns `claude` with
`--output-format stream-json --include-partial-messages --verbose` and streams each
stdout/stderr chunk incrementally: appended to `runs/<runId>.log` as it arrives, and published
on a new in-process pub/sub (`lib/runOutputBus.ts`) that the `/api/events` aggregator forwards
as a `"run-output"` SSE event (`{runId, chunk}`) — deliberately riding the **existing** single
SSE connection rather than opening a second one, per the single-SSE-provider rule established
in PR #25's own fix-loop. A new non-throwing, forward-compatible parser (`lib/streamJson.ts`)
validates each line is at least well-formed-or-gracefully-skipped without maintaining an
event-type allowlist; parse failures never affect what's appended/published.

The panel live-streams a still-active run from the accumulated `DashboardState.runOutput` map;
once a run ends, it fetches the full settled output once from a new `GET /api/run/output` route,
which resolves `runId` (format-validated `/^[0-9a-f]{16}$/`) to its `RunRecord` in `runs.jsonl`
and reads *that record's own* `outputPath` — `runId` is never itself joined into a filesystem
path, going one step further than the strict-format-validation precedent
([[pr_31_assistant-link-approve-button]]) that motivated the check in the first place. Two
Criticals were found and fixed in review: a missing `child.on("error", ...)` handler (a spawn
failure like `ENOENT` would otherwise hang the request forever and leak the per-button lock),
and a `fetchSettledOutput` that silently swallowed all fetch/parse errors (now a discriminated
result type with a distinct in-progress/error/ok case and a visible retry button).

**Settled output is parsed, not dumped raw ([[pr_124_dashboard-run-output-result-extraction]],
2026-07-09).** PR #80–82 shipped `lib/streamJson.ts`'s parser but nothing consumed its parsed
`{ok, value}` shape beyond the never-throws guarantee — `GET /api/run/output` returned the
entire raw JSONL log file as `output`, so the panel rendered every hook/system/assistant event
verbatim with the actual answer buried in a nested blob at the end (surfaced by a real user
report: a correct "what time is it in Dublin" answer was unfindable on screen). Fixed with a
module-private `extractResultText()` in `route.ts` that scans the log backwards for the last
`type:"result"` line and returns its `result` field, falling back to the raw content if no valid
result line exists (crashed run, or a non-string `result` e.g. an `error_during_execution`
subtype) — same fallback value as the old behavior, so that path is unchanged by construction.

**Live output is now projected clean by default, with a live-only raw toggle
([[pr_139-141_dashboard-ask-enter-clean-output]], 2026-07-11).** A new exported
`projectAssistantText(raw)` in `streamJson.ts` reduces a raw stream-json log to just the
assistant's readable prose: prefers the last `type:"result"` line's `result` field
(last-wins if more than one appears, e.g. across multiple turns); else concatenates `text_delta`
values from `stream_event`/`content_block_delta` events (covers a still-streaming run with no
result line yet, ignoring `input_json_delta` tool-use noise); else returns the raw input
unchanged so a run that produced real output never renders empty. `OutputViewerPanel.tsx` renders
the projected text by default behind a `showRaw` toggle (`.hud-output-toggle`, styled in the same
PR's CSS sibling #141 alongside a `.hud-output-viewer` readability bump — font 9.5px→12px,
max-height 220px→320px). **The toggle only appears for a still-live run** — the settled path
(above) is already server-extracted with no raw JSONL left client-side to toggle to, so rendering
the button for a settled run would be a no-op control; `OutputViewerPanel.test.ts` encodes this as
an explicit negative assertion. This live/settled asymmetry predates this PR (established by
[[pr_80-82_dashboard-stream-run-output-viewer]] and [[pr_124_dashboard-run-output-result-extraction]]);
this cluster's contribution is scoping the new toggle to respect it, caught by review rather than
present in the original design.

**Enter-to-submit ([[pr_139-141_dashboard-ask-enter-clean-output]], PR #139).** The `.hud-cmd-input`
behind any `inputAllowed: true` button (currently just "ask") gained an `onKeyDown` handler:
`Enter` without `Shift` submits the same run `handleClick(btn)` would; `Shift+Enter` is a reserved
no-op (the input is a single-line `<input>`, not a `<textarea>` — no multi-line behavior exists yet).

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

**Honest build feedback (added by [[pr_89-100-104-106_approve-build-e5-live]],
PR #106):** the bare `building` (opaque for up to 45 minutes) is replaced with
live progress. The builder writes a coarse phase — one of `authoring | testing
| pushing | opening_pr` — to `builds/<hash>/phase`; `collectBuilds`
closed-set-validates that word **before it reaches the client** (same
reject-never-default discipline as `state`), and the panel shows `building`
followed by the phase, an elapsed timer (from `startedAt`), and heartbeat
freshness ("last active Ns ago"). After a build's PR leaves the dashboard's
open-PR set (merged or closed) the panel shows **`PR resolved`** instead of a
stale `awaiting your merge` — the builder now writes `pr_url` immediately after
`gh pr create` so the join can happen. A **null-guard skips reconciliation**
(falls back to `awaiting`) whenever the open-PR set is untrustworthy — gates
not loaded, poll failed, or a repo degraded to an error entry — so an open PR
is never falsely marked resolved. `buildPrompt` gained a `buildDir` parameter
to interpolate the concrete phase-file path.

### Obsidian command centre

A native Obsidian plugin (official TypeScript template) registering a code-block processor (`agentic-os`) that renders dashboard state inside a real markdown note, sharing the same button config as the web deck. The routines sub-project (#2, now shipped — see [[intent-queue-runner-contract]] and [[dashboard-runner]]) defines the queue+runner contract this seam was frozen against; the plugin writes intent files to `~/.claude/coderails-dashboard/queue/` per that contract, but still also keeps its interim direct-exec path — [[dashboard-runner]] existing doesn't yet make the plugin stop invoking `claude` itself, so the plugin has not yet been updated to rely solely on the now-real runner. Ships a committed, reproducible `dist/main.js` build (same precedent as `wiki-init`'s committed Marp assets).

**`pressButton` buildArgv error-handling parity ([[pr_86-107_2026-07-08_loop-lib-residuals|PR #86]], 2026-07-08):** the interim direct-exec path's `pressButton` function (`src/exec.ts`) now wraps its `buildArgv(button, input)` call in a `try`/`catch`, returning `{ ok: false, reason: "invalid-input" }` on throw instead of letting the exception propagate uncaught — matching the web app's `route.ts`, which already had this catch. `buildArgv` throws on an empty effective prompt (no command and no input) or a leading-whitespace-then-dash flag-smuggling shape; pre-#86, either input crashed the Obsidian plugin's press instead of resolving to a `PressResult`.

## Starting / stopping

`skills/dashboard/scripts/start-dashboard.sh` (npm ci → build → start → open, idempotent re-launch) and `stop-dashboard.sh` (kills the pidfile'd process). Port overridable via `DASHBOARD_PORT`.

**First-run gotcha:** `npm ci` is exact-lock — unlike `npm install`, it fails hard (`EUSAGE`) rather than silently repairing if `package-lock.json` has drifted from `package.json`, including drift in optional/transitive native deps (e.g. `@emnapi/*`). See [[dashboard-lockfile-emnapi-drift_2026-07-08]] for a case that blocked first run this way and the general fix (`npm install --package-lock-only`).

### Surviving reboots (launchd)

[[pr_88_93_dashboard-launchd]] gives the dashboard the same reboot-survival mechanism [[routines]] already has: a launchd LaunchAgent (`launchd/com.coderails.dashboard.plist`, `RunAtLoad`+`KeepAlive`+`ThrottleInterval 60`) instead of the manually-started, pidfile-tracked process above. A thin exec wrapper, `skills/dashboard/runner/bin/dashboard-server.sh`, execs `npm run start` in the foreground (sibling to `install-routines.sh`'s `bin/*.sh` pattern — a background+PID-file model here would just leave launchd babysitting an empty shell). It deliberately duplicates ~13 lines of `start-dashboard.sh`'s build-if-stale logic rather than sharing it (accepted YAGNI; unify only if the copies drift), extended with a fail-safe check the manual script doesn't need: staleness also compares `package.json`/`package-lock.json`/`next.config.mjs` against `.next`, not just `src/`, since a daemon has no operator to notice a stale build.

`launchd/install-dashboard-agent.sh` / `uninstall-dashboard-agent.sh` copy the plist into `~/Library/LaunchAgents/` and bootstrap from that copy — the same load-bearing fix [[routines]]' own boot-persistence section documents discovering first (a bootstrap from the repo path silently unloads on reboot; launchd only auto-loads plists living in `~/Library/LaunchAgents/`). The installer also refuses to bootstrap if port 4173 is already held (`lsof` pre-flight), avoiding an EADDRINUSE crash-loop against a live manual server.

**Once the agent is installed, `stop-dashboard.sh` cannot stop it** (no pidfile for the agent-owned process) — use `launchctl bootout gui/$(id -u)/com.coderails.dashboard`. Stop any manual server first; installing the agent while a manual server holds the port causes a `ThrottleInterval`-rate-limited (60s) crash-loop until one side stops.

**Uninstall race, fixed same day ([[pr_88_93_dashboard-launchd]], PR #93):** `launchctl bootout` is asynchronous for a running `KeepAlive` job (~2s observed to actually unload) — a single immediate `launchctl print` re-check spuriously reported "still loaded" and bailed before removing the LaunchAgents copy. The uninstaller now polls up to 10×1s before declaring failure, and only removes the copy after the loaded-check passes.

**Stale-build gotcha, observed live 2026-07-11.** A production dashboard server (`npm run start`, no pidfile — started ad hoc, not via `start-dashboard.sh`) kept running for ~7 hours while 7 PRs merged to `main`, including all 6 right-rail UX fixes below. Screenshotting it after the merges showed no visible change, because `next build`'s output isn't hot-reloaded — the `.next/BUILD_ID` timestamp (16:16) predated the last merge (23:39) by over 7 hours. The untracked orphan process (no pidfile, so `stop-dashboard.sh` couldn't find it) had to be killed manually and `start-dashboard.sh` re-run to pick up current `main`. Lesson: before trusting a running dashboard's screenshot as evidence of a shipped UI change, check `.next/BUILD_ID`'s mtime against the last relevant merge commit's timestamp, not just whether the server answers HTTP 200.

## Right-rail UX cluster (PRs #130-136, 2026-07-10)

[[pr_130-136_dashboard-right-rail-ux]] — six independent PRs fixing UX/IA findings on the right rail: panel separation (bordered cards + rose accent spine on the shared `.hud-block` class, affecting both rails), input affordance (boxed input fields + an "ARG" tag on input-capable buttons), label wrapping (ellipsis truncation instead of layout-breaking two-line wrap), run-history structure (filled/hollow status glyphs replacing a uniform dot, in both of the two intentionally-duplicated run-history list implementations), output-viewer context (a header bar identifying which run produced the currently-shown output), and button-state differentiation (a new `lastOutcome` field driving a transient green/rose bullet-flash on run completion — the only finding needing new component state, and the first PR in this codebase to add React Testing Library / jsdom test infrastructure). Shipped under a registered [[agentic-loop]] run with a mid-loop skill edit — see [[pr_134_agentic-loop-retry-until-green]].

## Left-rail loop-decisions tile (PRs #148, #149, 2026-07-12)

`collectLoops` (`skills/dashboard/app/src/lib/collect/sessions.ts`) gained a `decisions: string[]` field on `LoopInfo`, populated by a new `readDecisions()` helper: reads `progress.json`'s `decisions_absorbed` array (added by [[agentic-loop]] Phase 2.5/2.6/5/6, see [[loop-progress-fields]]), filters to well-formed `{phase, decision}` string-pair entries — degrade-don't-throw on a non-array or malformed entries, same stance as the pre-existing `readUnitTitles` helper — formats each `"<phase>: <decision>"`, and returns the last 5, newest-first. `RailLeft.tsx` renders these as `.hud-decision-item` entries under the existing Directives card (the same card already showing the active loop's work-unit checklist): one line per decision, duplicates rendered as distinct entries (not collapsed), an empty array rendering no sub-list rather than an empty placeholder.

**`readEvalsFrozen` gained the `grade-loop` stamp check**, mirroring the bash `als_read_loop_evals_result` reader's `UNSTAMPED` demotion (see [[task-evals-gate]]): a `GO`/`TIER0` verdict now additionally requires `data.grading.by` and `data.grading.checksum` both present and non-empty before reading as frozen, and an explicit `result: "NO-GO"` wins over a tier-0 exemption regardless of grading presence, matching the bash reader's precedence. This check is **presence-only** — no checksum recomputation, an explicit KISS trade-off for a display surface that can't block anything, documented inline in the source.

**#149 is a same-day follow-up**, found after #148 merged: the first cut only required `.grading.checksum` non-empty, not `.grading.by` — weaker than the bash reader's `[ -z "$stamped_by" ] || [ -z "$stamped_checksum" ]` check, so a partially-stamped artifact (checksum present, `by` absent) could show "frozen" here while the hook would correctly block it as `UNSTAMPED`. #149 tightened the TS check to require both fields non-empty (after trim). This is a live illustration of `sessions.ts` being a **third, independent consumer** of the loop-scope `evals.json` schema (writer: `post_evals.sh grade-loop`; enforcing reader: `loop_state_guard`) that must stay in lockstep with the bash SSOT even though it's read-only — see [[task-evals-gate]]'s "one schema, two seams" section, updated by this cluster to name three.

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
- [[pr_89-100-104-106_approve-build-e5-live]] — the live-fire close of that pipeline: first real skill built ([[verify-merged-pr]]) + this panel's honest build feedback (phase/timer/heartbeat, `PR resolved` reconciliation)
- [[assistant-link-send-gate-architecture]] — sub-project 4's send-gate design, the queue seam this panel reads/mutates, and the ASSISTANT.LINK panel's four D6 slots (only "sends + approvals log" has a real data source so far)
- [[pr_28_assistant-link-queue-contract-and-panel-spec]] / [[pr_31_assistant-link-approve-button]] — the contract spec and the panel's implementation + path-traversal fix
- [[intent-queue-runner-contract]] / [[dashboard-runner]] / [[routines]] / [[memory-consolidation]] — sub-project 2, the routines cluster: the queue schema/lifecycle, the sole-executor runner, the scheduling convention, and one shipped routine's skill
- [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] — the argv merged-prompt fix (button input was silently dropped on every `inputAllowed` button since inception) and the "ask" button pattern this fix enabled
- [[pr_128_dashboard-ask-button-auto-profile]] — the "ask" button's `profile: "standard"` → `"auto"` switch: closes the headless permission-block hang on tools like `WebSearch`, but does not fix the separate, still-open tool-call-attempt non-determinism; third occurrence of independent reviewers converging on an identical finding
- [[voice_announce]] — the sibling PR in the same cluster: a new observe-only Stop hook announcing loop lifecycle events via macOS `say`, unrelated to the dashboard's own code but merged in the same session
- [[pr_80-82_dashboard-stream-run-output-viewer]] — the Run Output viewer: incremental stream-json capture, the `runOutputBus`/`"run-output"` SSE event, the path-traversal-safe `GET /api/run/output` route, and the two Critical review fixes (missing spawn-error handler, silently-swallowed fetch errors)
- [[dashboard-run-log-streaming-viewer-gap_2026-07-07]] — the investigation this cluster closes; documents the pre-fix write-only log model and the cross-PR constraints (single-SSE-provider, strict-ID-validation, never-throw, token non-leakage) the fix had to respect
- [[pr_130-136_dashboard-right-rail-ux]] — the six-PR right-rail UX cluster (panel separation, input affordance, label wrapping, run-history glyphs, output-viewer header, button-state flash)
- [[pr_134_agentic-loop-retry-until-green]] — the mid-loop agentic-loop skill edit that shipped as a 7th PR in the same loop as the UX cluster
- [[pr_138_remove-specs-plans-tracking]] — removes this and every other skill's design-spec/plan file paths from repo tracking; this page's design-spec/plan lines above were updated in place as a direct consequence
- [[pr_144-149_agentic-loop-hardening-from-loop-engineering]] — PRs #148/#149 source record: the left-rail loop-decisions tile and the `readEvalsFrozen` third-seam drift fix
- [[loop-progress-fields]] — `decisions_absorbed`, the `progress.json` field this tile reads
- [[task-evals-gate]] — the `grading`/`UNSTAMPED` mechanism `readEvalsFrozen` mirrors, and the "one schema, two seams" (now three) framing
- [[pr_139-141_dashboard-ask-enter-clean-output]] — Enter-to-submit on the ask input, `projectAssistantText()`'s clean-by-default projection of the live run-output view (result-line-preferred, delta-concatenation fallback), the live-only raw/clean toggle, and the `.hud-output-viewer`/`.hud-output-toggle` readability CSS
