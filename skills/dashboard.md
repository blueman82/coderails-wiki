---
title: "Skill: dashboard"
type: skill
created: 2026-07-06
last_updated: 2026-07-15
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
  - sources/pr_175-176_crack-on-gate-and-inbox-brief-button.md
  - sources/pr_179_dashboard-lan-access.md
tags: [skill, dashboard, observability, nextjs, r3f, sse, obsidian, agentic-os, sub-project-1-of-5, queue-contract, builder, ask-button, argv, run-output, streaming, launchd, reboot-persistence, npm-ci, lockfile, permission-mode, enter-to-submit, decisions-absorbed, loop-decisions-tile, deck-trim, multi-loop, gates-freshness, headless-exemption, inbox-brief, rachel, lan-access, dashboard-host, request-guard, security-posture]
---

# Skill: dashboard

Launches the coderails observability dashboard — a live local web HUD showing sessions, agentic loops, PR gate states, runs, and memory activity — plus a companion Obsidian command centre. Sub-project 1 of a 5-part agentic-OS evolution sequence (observability → routines → workflow-audit → assistant-agent kernel integration → improvement loops); see [[routines]], [[workflow-audit]], [[assistant-link-send-gate-architecture]], and [[pr_118-123_self-improving-loops]] for the other four.

Source: `coderails/skills/dashboard/SKILL.md`
Invoked as: `coderails:dashboard`

> Design spec and plan file paths formerly cited here (`docs/coderails/specs/2026-07-06-observability-dashboard-design.md`, `docs/coderails/plans/2026-07-06-observability-dashboard-plan.md`) no longer exist — removed from repo tracking by [[pr_138_remove-specs-plans-tracking]] (2026-07-11). This page's own content is now the durable record.

## What it shows

Six panels as of the [[pr_163-168_dashboard-rethink|dashboard-rethink cluster]] (2026-07-14; was seven before PR #166 removed DOCUMENTS/MEMORY.TRAIL — see below), all reading state the kernel already produces (no new services, no telemetry leaving the machine): SYSTEM VITALS (usage/hooks/lint sparklines), DIRECTIVES (now **one card per live loop**, not a single active-loop checklist — see "Multi-loop Directives panel" below), COMMAND DECK (declared one-click buttons, trimmed to 3 visible by default as of PR #163 — see "Deck trim" below), PR GATES (merge-ready / blocked / stale, from the marker-grammar libs; polled every 30s as of PR #164, was 120s — see "Gates freshness" below), a bottom-centre hero stat (now follows the freshest **live** loop, not a single tracked "active loop" — PR #168), and ASSISTANT.LINK — originally a reserved placeholder for sub-project 4 (assistant-agent kernel integration), now a real panel (coderails PR #31) rendering the pending send-approval queue for assistant-agent's send-gate; see [[assistant-link-send-gate-architecture]].

### Deck trim (PR #163, 2026-07-13/14)

`DashboardButton` gained an optional `hidden?: boolean` field (`lib/config.ts`); config validation rejects a non-boolean value; the button list is filtered (`config.buttons.filter((b) => !b.hidden)`) before rendering. Applied live in the per-user `~/.claude/coderails-dashboard.json` (not in-repo) to trim the deck from 7 buttons to 3 visible (DEEP RESEARCH / WIKI QUERY / ASK), hiding 4 routine-trigger buttons (`wiki-lint`, `sync-docs-weekly`, `memory-consolidation-weekly`, `loop-retro-promotion`) that remain in config as live `buttonRef` targets for the routines runner — hiding is display-only, not removal. See [[pr_163-168_dashboard-rethink]].

### Memory.Trail panel removed (PR #166, clean-break, 2026-07-14)

The DOCUMENTS/MEMORY.TRAIL panel, its collector, and its state slice were removed end-to-end, along with the dead Command-Deck run-history list and the orphaned `memoryPaths` config field from both `DashboardConfig` declarations (app and shared lib) and all fixtures/docs. Clean-break disposition — no deprecated-but-present field, no back-compat shim. Every "seven panels" / "wiki + memory mtime feed" description predating 2026-07-14 is now stale by construction; treat six panels as current. See [[pr_163-168_dashboard-rethink]].

### Multi-loop Directives panel (PR #168, clean-break, 2026-07-14)

DIRECTIVES now renders **one card per live loop** instead of a single tracked "active loop." `LOOP_LIVE_WINDOW_MS = 60 * 60_000` (`hooks/useDashboardState.ts`) — a loop counts as live when `status !== "complete"` and it was updated within the last 60 minutes; `liveLoops()`/`stalledLoops()` partition the non-complete set on that window, both newest-first. A `useNow(30_000)` ticking-clock hook re-evaluates live/stalled status every 30s without a page reload (consumed by both `RailLeft.tsx` and `BottomHero.tsx`). Stalled loops render as a dim sub-list with no cards; each live card gets a `Live.N` header suffix, an evals-frozen footer, and a decisions sub-list (the same `decisions_absorbed` feed from PR #148/#149, now per-card). `BottomHero` follows `liveLoops(...)[0]`, the freshest live loop — `selectActiveLoop` is retired clean-break, not deprecated. This builds directly on the PR #165 collector rewrite: `LoopInfo` gained `title` (resolution chain `loop` field → `authorising_prompt_raw` → slug fallback), `lastUpdatedMs` (`last_updated` field wins, mtime fallback), and `units[]` with a `"done"|"in-flight"|"pending"` status union, replacing the old `unitTitles`/`done` boolean pair; both current (keyed-object) and legacy (array) `progress.json` schemas parse. See [[pr_163-168_dashboard-rethink]].

### Gates freshness (PR #164, 2026-07-14)

`DEFAULT_GATES_POLL_MS` dropped from 120s to 30s (`lib/collect/index.ts`). A new `GATES_RUNS_DEBOUNCE_MS = 3_000` debounces an additional gates refresh off runs-directory changes — a run finishing now updates PR GATES within ~3s of the runs-dir write, on top of (not instead of) the 30s poll floor. Shipped sonnet after 2 failed haiku attempts (Phase 2.8 model-routing fallback). See [[pr_163-168_dashboard-rethink]].

## Architecture

One Next.js process serves both the React Three Fiber HUD frontend and the API routes: an SSE stream (`/api/events`) and a token + Origin-guarded run trigger (`/api/run`). Binds `127.0.0.1` by default — loopback-only until [[pr_179_dashboard-lan-access|PR #179]] added an opt-in `DASHBOARD_HOST` override (see "LAN access (opt-in)" under Starting/stopping below); unset/empty is unchanged from before that PR. Collectors read: `~/.claude/projects/*` session dirs, agentic-loop `progress.json` (via the existing `agentic_loop_path.sh` SSOT — see [[agentic-loop]]), `gh`-polled PR state parsed with the same marker grammar as `review-artifact.sh`/`eval-artifact.sh` (see [[review-artifact-seam]] and [[task-evals-gate]]), hook logs, and wiki/memory mtimes.

### Button / run model — the security-load-bearing piece

A button is a declared, bounded run, never a free prompt box. `POST /run` looks up a button name in `~/.claude/coderails-dashboard.json` and refuses anything undeclared. `buildArgv()` in `src/lib/argv.ts` is the **single** profile→flag mapping (`read-only` → `--allowedTools Read Grep Glob`; `standard` → inherit target project's allowlists; `auto` → `--permission-mode auto`, added [[pr_128_dashboard-ask-button-auto-profile|PR #128]]; `bypass` → `--dangerously-skip-permissions`, opt-in per button, warning badge in both UIs) — the Obsidian plugin must reuse it, never re-implement it.

**`auto` closes a headless-only hang, not tool-call non-determinism ([[pr_128_dashboard-ask-button-auto-profile|PR #128]], 2026-07-10).** In headless `claude -p` mode, `standard` adds no permission flags, so a tool needing a fresh permission grant (e.g. `WebSearch`) blocks indefinitely on an unanswerable prompt — the model then silently falls back to answering from memory with no tool call ever attempted. `--permission-mode auto` auto-approves the permission decision instead of blocking on it, closing that specific hang. It does **not** fix the separate, already-documented non-determinism of *whether the model attempts the tool call at all*: direct empirical testing (both in the PR's own `argv.ts:40-49` code comment and independently reproduced this session) found only 1 real tool call in 3 consecutive identical runs, the other 2 falling back to "I don't have internet access" with no attempt and no permission-block message. See [[pr_128_dashboard-ask-button-auto-profile]] for the full finding.

**Flag-smuggling closed by two independent layers**, found and fixed during review (Tier-2 eval / review cycle, not caught by the original implementation): a leading-dash input like `--dangerously-skip-permissions` would otherwise be parsed by the `claude` CLI as a real flag rather than literal prompt text (confirmed empirically: `claude -p "--version"` prints the version banner instead of answering the prompt). The fix: (1) `input` starting with `-` (checked against the **trimmed** value, so whitespace can't hide a leading dash from this check — closed 2026-07-07, PR #70) is rejected outright before any sentinel is trusted, and (2) a literal `--` end-of-options sentinel is always inserted before the prompt regardless, confirmed on-machine to make the CLI treat everything after it as literal text. Neither layer alone was judged sufficient — the review explicitly wanted the reject-outright check as a second wall, not just the sentinel.

**Input-delivery bug, found and fixed 2026-07-07 (PR #70 — see [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]):** the original argv shape pushed `input` as **a separate positional argv element** after `btn.command` and the sentinel (`["-p", btn.command, ...profileFlags, "--", input]`). The `claude` CLI's `-p`/`--print` only consumes ONE positional prompt — it never merges a second positional into the prompt already consumed, so `input` silently never reached the model on **any** `inputAllowed` button, since the button model's inception (`verify-q`, `deep-research`). Caught by an intake smoke test observing the *live* dashboard's response, not by any argv-shape unit test — every prior test checked shape only, none checked that input was actually delivered, so the pre-fix code passed all of them. Fixed by merging `command` and `input` into **one** prompt string before the sentinel: `btn.command ? \`${btn.command} ${input}\` : input`, giving the current shape `["-p", ...profileFlags, "--", mergedPrompt]` when input is present (profile flags precede the sentinel; the merged prompt is the sole positional after it). Without input, the original single-token shape is unchanged. Empty/whitespace-only input is now normalised to "no input" inside `buildArgv` itself (not just at the UI layer), and `buildArgv` throws on an empty effective prompt (no command and no input) — `route.ts`'s existing catch surfaces this as a 400.

A new **"ask" button pattern** follows directly from this fix: `command: ""`, `inputAllowed: true` — free text becomes the entire prompt inside the button's declared envelope (profile flags still apply). Live in the owner's per-user config; documented in `examples/dashboard-config.json` (4th button, first time exercised by `lib/test/config.test.ts`). Originally shipped on `profile: "standard"`; switched to `profile: "auto"` by [[pr_128_dashboard-ask-button-auto-profile|PR #128]] (2026-07-10) to avoid `standard`'s headless permission-block hang on tools like `WebSearch` — see the caveat above on what `auto` does and does not fix.

**`inbox-brief` button — a thin dispatcher to a second entrypoint (PR #176, 2026-07-14).** `examples/dashboard-config.json` gained a `profile: "bypass"`, `bypassPermissions: true` button with `cwd: "/Users/harrison/Github/assistant-agent"` whose declared `command` instructs the spawned `claude` to run exactly one Bash command and report its output: `bin/rachel "Read tasks/inbox-brief.md and follow it." < /dev/null`. This still respects the button/spawn boundary above: `route.ts`'s `POST /run` hardcodes `spawn('claude', ...)`, so a button can never invoke `bin/rachel` (assistant-agent's secretary entrypoint) directly — reaching it requires a `claude` instance whose prompt shells out via Bash. `< /dev/null` starves the spawned process of stdin, the same non-interactive posture every other spawned-`claude` button already assumes. Unlike every prior bypass/auto button, whose spawned `claude` does its own reasoning inline, this one is a pure pass-through to `rachel` (Gary's secretary; see [[assistant-link-send-gate-architecture]]) — the first button in this config shaped that way. See [[pr_175-176_crack-on-gate-and-inbox-brief-button]] for the full record.

Every run is single-flight per button, JSONL-recorded (`~/.claude/coderails-dashboard/runs/`) with argv/cwd/profile/exit code, and CSRF-token-guarded (both `/api/events` and `/api/run` also reject an `Origin`/`Host` outside the allowed set — loopback always, plus the one exact `DASHBOARD_HOST` LAN host when configured; see [[pr_179_dashboard-lan-access|PR #179]]).

**Headless discipline-hook exemption (PR #167, 2026-07-14).** `route.ts`'s `spawn(...)` call sets `CODERAILS_HEADLESS_RUN=1` (spread-preserving: `{ ...process.env, CODERAILS_HEADLESS_RUN: "1" }`) — the **sole** set-site in the codebase. `check_confidence_labels.sh` and `check_verify_loop.sh` both skip enforcement when this flag is present, but **only on the `Stop` event** — `SubagentStop` still blocks unconditionally, matching the documented invariant that worker output always blocks. Rationale: a headless `claude -p` run has no interactive turn left to satisfy a repair-turn block, so without the exemption the discipline gate would displace the run's actual answer with gate-repair text instead of the wiki/ask response the user asked for. See [[enforcement-model]] for the ceiling framing (env-triggered, inside the agent's own trust domain, not a privilege boundary) and [[pr_163-168_dashboard-rethink]] for the source record. A related same-loop finding (t7, no PR): slash-command-as-prompt runs show 0 model turns in run logs because the slash command executes as a CLI local command — the outer session genuinely takes no model turns, so the write path is faithful and this isn't a logging bug.

### Run Output viewer — streaming capture + modal playback (PR #80–82, superseded/extended by #171–174, #181, #183)

> Rewritten 2026-07-15 — the previous text here described a retired inline `<pre>` viewer
> with a live-only raw/clean toggle. Neither exists anymore; see
> [[dashboard-run-output-rendering-gap_2026-07-15]] for the investigation that caught the
> wiki 5 PRs behind the code before this rewrite.

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
event-type allowlist; parse failures never affect what's appended/published. **This streaming
backend is unchanged by everything below** — #171-174/#181/#183 are all client-rendering
follow-ups; the 2026-07-15 investigation confirmed neither #172 nor #174's diffs touch
`api/run/output/route.ts`, `runOutputBus.ts`, or any file-path/token boundary.

The panel live-streams a still-active run from the accumulated `DashboardState.runOutput` map;
once a run ends, it fetches the full settled output once from `GET /api/run/output`, which
resolves `runId` (format-validated `/^[0-9a-f]{16}$/`) to its `RunRecord` in `runs.jsonl` and
reads *that record's own* `outputPath` — `runId` is never itself joined into a filesystem path,
going one step further than the strict-format-validation precedent
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

**Client-side, `projectAssistantText()` now runs unconditionally on both live and settled
paths ([[pr_139-141_dashboard-ask-enter-clean-output]], 2026-07-11; behavior since changed
again — see below).** The exported `projectAssistantText(raw)` in `streamJson.ts` reduces a raw
stream-json log to just the assistant's readable prose: prefers the last `type:"result"` line's
`result` field (last-wins if more than one appears); else concatenates `text_delta` values from
`stream_event`/`content_block_delta` events (covers a still-streaming run with no result line
yet); else returns the raw input unchanged so a run that produced real output never renders
empty.

**The inline `<pre>` viewer and its live-only raw/clean toggle were retired outright by
PR #172 (`rethink/t10-runoutput-modal`, merged 2026-07-14) — do not describe either as current.**
`(verified)` `RunOutputOverlay.tsx` — a portal-rendered modal opened by clicking a run-history
row, not an always-visible inline box — replaced the old `.hud-output-viewer`/`showRaw` toggle
entirely. There is no `showRaw` state anywhere in the current overlay and **no way to view raw
stream-json in the UI at all** — `OutputViewerPanel.tsx:151` now calls
`projectAssistantText(rawOpenOutput)` unconditionally for *both* the live and settled paths (a
change from the #124/#139-141 model, where the settled path's server-side `extractResultText`
was assumed already-clean prose needing no further client projection — it still needs
`projectAssistantText` too, because a crashed run with no `result` line falls back to raw
stream-json server-side).

**PR #174 (`rethink/t12-runoutput-markdown`, merged 2026-07-14) — GFM markdown rendering with
two deliberate security layers, previously undocumented here.** `(verified)` The modal renders
output via `<ReactMarkdown remarkPlugins={[remarkGfm]}>` with: (a) react-markdown itself as the
sanitizer — raw HTML in the source renders as escaped text, no `rehype-raw`, no
`dangerouslySetInnerHTML`, because run output is untrusted; (b) an `img` component override that
renders every image as its alt text instead of a live `<img>` — closing a tracking-beacon/SSRF-
from-the-viewer vector a bare CommonMark image would otherwise open (a GET fires on render with
no click). Links are left alone, relying on `defaultUrlTransform` to inert `javascript:` URLs.
A future change to run-output rendering that reintroduces `rehype-raw` or live images would be
undoing this considered decision, not adding a feature.

**PR #181 (merged 2026-07-15) — long lines/fenced code blocks now wrap.** `.hud-markdown pre`
gained `white-space: pre-wrap; overflow-wrap: anywhere; word-break: break-word;` (kept
`overflow-x: auto` as a fallback) — previously a long unbroken line overflowed the modal
horizontally. See [[pr_181_183_dashboard-run-output-wrap-and-inbox-brief-clean-modal]].

**PR #183 (merged 2026-07-15) — inbox-brief button's modal output is now the clean brief, not
Rachel's execution trace.** The button's `command` was rewritten so the outer spawned `claude`
agent reads the scratch file named on a `BRIEF FILE: <path>` sentinel line (added to
`tasks/inbox-brief.md` by the paired assistant-agent PR #27) and makes its entire final message
that file's verbatim contents — necessary because the modal renders the outer agent's `result`
field, and merely changing what Rachel outputs (the inner layer) can't reach it. See
[[pr_181_183_dashboard-run-output-wrap-and-inbox-brief-clean-modal]] for the full two-repo
mechanism.

**Enter-to-submit ([[pr_139-141_dashboard-ask-enter-clean-output]], PR #139) — still current,
unaffected by the modal rewrite.** The `.hud-cmd-input` behind any `inputAllowed: true` button
(currently just "ask") gained an `onKeyDown` handler: `Enter` without `Shift` submits the same
run `handleClick(btn)` would; `Shift+Enter` is a reserved no-op (the input is a single-line
`<input>`, not a `<textarea>` — no multi-line behavior exists yet).

**PR #171 (`rethink/t9-no-truncation`, merged 2026-07-14) — the 80-char title truncation and
two-line description clamp documented elsewhere in this page (Directives panel) no longer
apply.** `(verified)` `readTitle()` in `sessions.ts` now returns the full trimmed prompt with no
80-char cap; `.hud-unit-desc`'s CSS switched from `-webkit-line-clamp: 2` to
`overflow-wrap: break-word` (no clamp at all). Not run-output-specific, but caught by the same
2026-07-15 investigation and noted here since it's the same PR range.

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

### LAN access (opt-in) — PR #179, 2026-07-15

Both serving paths (`start-dashboard.sh` for a manual run, `runner/bin/dashboard-server.sh` for the launchd agent) read one new env var, `DASHBOARD_HOST`. Unset/empty is byte-for-byte the pre-PR behaviour: bind `127.0.0.1`, guard accepts loopback only. Set to a concrete LAN IP (e.g. `192.168.50.140`), two things change together from that single variable: the server binds that address instead of loopback, **and** `requestGuard.ts` (`isLocalOrigin`, via a new `isAllowedHost()` helper) additionally exact-matches that one host — and only that host — for both the `Host` and `Origin` headers, alongside loopback which is always still allowed. From another device on the LAN: `http://<LAN-IP>:4173`. For the launchd agent, `com.coderails.dashboard.plist` now ships an `EnvironmentVariables` dict with an empty `DASHBOARD_HOST` entry — fill in the LAN IP and reinstall the agent (`launchd/install-dashboard-agent.sh`) for it to take effect.

**Validated, fail-loud, not silently-broken.** Both scripts reject a wildcard bind (`0.0.0.0`, `::`, `*`), a `host:port` form, and a bare hostname with a non-zero exit before `npm run start` is ever invoked — the guard exact-matches one literal host string, so a wildcard bind would silently 403 every real LAN request that reached it. Under the launchd agent (`KeepAlive` + `ThrottleInterval 60`), a rejected `DASHBOARD_HOST` means the agent respawns and fails every 60s until fixed — correct fail-loud behaviour for a misconfiguration, not a new bug class.

**Security posture — deliberate, not an oversight.** The dashboard's COMMAND DECK `POST /api/run` and the workflow-audit Approve/Deny queue are an unauthenticated command-execution surface: no login, no per-request auth of any kind, by original design (see "Button / run model" above). The `Host`/`Origin` guard defends against a hostile web page or DNS-rebinding attack reaching the dashboard from a browser already open on the machine — it does **not** authenticate LAN devices. Once `DASHBOARD_HOST` is set, **any device on that LAN that can reach the port can trigger declared runs.** This was decided autonomously under a crack-on envelope, scoped explicitly to trusted-home-network use, with device-level auth deferred rather than built — see [[enforcement-model]] for how this project frames env-triggered/scope-gated capability expansions that stay inside an existing trust boundary rather than crossing one. Full test coverage for the allow/reject matrix (loopback, configured LAN host, arbitrary host, DNS-rebinding-shaped substring match, cross-origin, mismatched second LAN host, bracketed IPv6 literal) is in `skills/dashboard/app/test/requestGuard.test.ts`. See [[pr_179_dashboard-lan-access]] for the full source record.

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
- [[crack_on_gate]] / [[pr_175-176_crack-on-gate-and-inbox-brief-button]] — the sibling PR merged the same day (#175/#176): a new discipline hook plus the `inbox-brief` button, this config's first pure-dispatcher button (spawns `claude` only to shell out to `bin/rachel`, not to reason inline)
- [[task-evals-gate]] — the `grading`/`UNSTAMPED` mechanism `readEvalsFrozen` mirrors, and the "one schema, two seams" (now three) framing
- [[pr_139-141_dashboard-ask-enter-clean-output]] — Enter-to-submit on the ask input (still current) and `projectAssistantText()`'s clean-by-default projection; the live-only raw/clean toggle and `.hud-output-viewer`/`.hud-output-toggle` CSS this PR introduced were retired outright by PR #172 — see the Run Output section above and [[dashboard-run-output-rendering-gap_2026-07-15]]
- [[pr_163-168_dashboard-rethink]] — the six-PR dashboard-rethink cluster (2026-07-14): deck trim to 3 visible buttons via a `hidden` field, Memory.Trail panel/collector/slice clean-break removal, multi-loop collector rewrite (`title`/`lastUpdatedMs`/`units[]`), the multi-loop Directives panel + hero (live/stalled partition, `selectActiveLoop` retired), gates poll 120s→30s plus a 3s runs-dir debounce, and the `CODERAILS_HEADLESS_RUN` Stop-only hook exemption
- [[dashboard-run-output-rendering-gap_2026-07-15]] — investigation that found this page's Run Output section 5 PRs stale (retired inline `<pre>` viewer + raw/clean toggle documented as current); the modal rewrite above (PRs #171-174, #181, #183) is the ingest that closes it
- [[pr_181_183_dashboard-run-output-wrap-and-inbox-brief-clean-modal]] — the modal-wrap CSS fix (#181) and the two-repo `BRIEF FILE:` sentinel mechanism (#183 + assistant-agent #27) that makes the inbox-brief button's modal output the clean brief instead of Rachel's execution trace
- [[enforcement-model]] — the headless-run exemption's ceiling framing (env-triggered, Stop-event-only, inside the agent's own trust domain; `SubagentStop` unaffected); also the framing used for the LAN-access opt-in below (scope expansion inside an existing trust boundary, not a new one)
- [[pr_179_dashboard-lan-access]] — opt-in LAN exposure via `DASHBOARD_HOST`: bind + request-guard change together from one var, fail-loud validation in both serving scripts, launchd plist's new `EnvironmentVariables` dict, and the deliberate unauthenticated-command-exec security posture (Host/Origin guard blocks hostile-web-page/DNS-rebinding, does not authenticate LAN devices; trusted-home-network-only)
