---
title: "Investigation: dashboard run-output rendering — the wiki lags the code by 5 merged PRs"
type: investigation
created: 2026-07-15
last_updated: 2026-07-15
sources:
  - skills/dashboard/app/src/components/OutputViewerPanel.tsx
  - skills/dashboard/app/src/components/RunOutputOverlay.tsx
  - skills/dashboard/app/src/app/api/run/route.ts
  - skills/dashboard/app/src/lib/collect/sessions.ts
tags: [investigation, dashboard, run-output, streaming, wiki-drift, security]
---

# Investigation: dashboard run-output rendering — the wiki lags the code by 5 merged PRs

> Filed: 2026-07-15. Point-in-time snapshot — may be superseded.

## Question

A query asked what the wiki covers about the dashboard's run-output rendering, the
inbox-brief feature, the run route / `streamJson` projection, and the button→claude
spawn model — plus cross-PR constraints, gaps, superseded decisions, and plan-time
assumptions not actually enforced in code.

## Evidence

- [[dashboard]] (skill page, `last_updated: 2026-07-15`) still describes the Run Output
  viewer via [[pr_80-82_dashboard-stream-run-output-viewer]],
  [[pr_124_dashboard-run-output-result-extraction]], and
  [[pr_139-141_dashboard-ask-enter-clean-output]] as the current state: an inline
  `.hud-output-viewer` `<pre>` block with a live-only `showRaw` raw/clean toggle.
- `gh pr list --state merged` for #169-#177 (the PR range immediately following
  [[pr_163-168_dashboard-rethink]], same `dashboard-rethink` loop, session `0767967d`)
  shows **five merged PRs touching `skills/dashboard/*` with zero wiki coverage**:
  #170 (docs panel drift, `SKILL.md` only), #171 (t9 no-truncation), #172 (t10
  run-output modal), #173 (t11 mobile responsive), #174 (t12 run-output GFM markdown).
  `grep -rn "RunOutputOverlay|react-markdown|t10-runoutput|t12-runoutput|PR #172|PR #174"`
  over the whole vault returns zero hits.
- Read `OutputViewerPanel.tsx` (204 lines, current `main`) and `RunOutputOverlay.tsx`
  (111 lines, current `main`) in full.
- Read `run/route.ts:250-260` to confirm the `CODERAILS_HEADLESS_RUN` set-site and the
  literal `spawnImpl("claude", argv, ...)` call — not config-derived.
- `git worktree list` / `git branch --merged origin/main` to separate merged-but-
  unwikied work (t9/t10/t11/t12) from genuinely open work (PR #179 `dashboard-lan-access`,
  still `OPEN`, touching `requestGuard.ts`).

## Findings

**1. The wiki's central run-output claim is now false — not just missing a page.**
`(verified)` PR #172 (`rethink/t10-runoutput-modal`, merged 2026-07-14) replaced the
inline `<pre>` viewer entirely with `RunOutputOverlay.tsx`, a portal-rendered modal
opened by clicking a run-history row. `OutputViewerPanel.tsx`'s own header comment says
so explicitly: "Clicking a row opens an in-page OVERLAY... the retired inline `<pre>`
viewer squished that same text into a small box below this list, which is what Task T10
replaced." The wiki ([[dashboard]], [[pr_139-141_dashboard-ask-enter-clean-output]])
still documents the retired inline box as current.

**2. The live/settled raw-toggle from PR #139-141 no longer exists.** `(verified)` The
overlay component has no `showRaw` state and no toggle button at all — it always
renders `output` (already projected to prose by the panel) as sanitized markdown via
`react-markdown`. `OutputViewerPanel.tsx:151` calls `projectAssistantText(rawOpenOutput)`
unconditionally for **both** the live and settled paths (a change from #124/#139-141's
model, where the settled path's server-side `extractResultText` was assumed to already
be clean prose needing no further client projection — the comment at lines 142-148
explains the settled path still needs `projectAssistantText` too, because a crashed run
with no `result` line falls back to raw stream-json server-side). There is no way to
view raw stream-json in the UI anymore. A plan that assumes the raw/clean toggle exists,
or that only the live path needs client-side projection, is building on a retired
feature.

**3. PR #174 adds markdown rendering with real, undocumented security engineering.**
`(verified)` `RunOutputOverlay.tsx` renders output via `<ReactMarkdown remarkPlugins={[remarkGfm]}>`
with **two** deliberate security layers, per its own inline comments: (a) react-markdown
itself is the sanitizer — raw HTML in the source renders as escaped text, no `rehype-raw`,
no `dangerouslySetInnerHTML`, because "run output is untrusted"; (b) an `img` component
override renders every image as its alt text instead of a live `<img>`, closing a
tracking-beacon/SSRF-from-the-viewer vector a bare CommonMark image would otherwise open
(a GET fires on render with no click). Links are left alone, relying on
`defaultUrlTransform` to inert `javascript:` URLs. None of this is in the wiki's dashboard
security model, which today only documents the Origin/Host guard, token non-leakage, and
`safePrUrl` (assistant-link panel). A future plan touching run-output rendering could
reintroduce `rehype-raw` or live images without realizing it's undoing a considered
security decision, because that decision is currently only visible in source comments.

**4. PR #171 removed a truncation behavior the wiki still documents as current.**
`(verified)` [[pr_163-168_dashboard-rethink]] (t3/t4, #165/#168) and [[dashboard]]
describe `LoopInfo.title` falling back to "`authorising_prompt_raw` (first 80 chars,
trimmed, "…")" and unit descriptions as CSS-clamped to two lines. PR #171 (`rethink/t9-no-truncation`,
merged 2026-07-14, same loop) removed both: `readTitle()` in `sessions.ts` now returns
the full trimmed prompt text with no 80-char cap, and `.hud-unit-desc`'s CSS switched from
`-webkit-line-clamp: 2` to `overflow-wrap: break-word` (no clamp at all). This is a
straightforward superseded-decision gap, not security-relevant, but a plan referencing
"truncated to 80 chars" or "clamped to two lines" for Directives-panel content would be
wrong.

**5. `#179` (`dashboard-lan-access`, OPEN, not yet merged) adds an opt-in LAN-access mode —
read in full, not merely inferred from the file list.** `(verified)` The diff adds a new
`DASHBOARD_HOST` env var. Unset (the default, unchanged from today): `isAllowedHost()` in
`requestGuard.ts` degrades to exactly `isLocalhost()` — behaviour identical to current main.
Set: it allows exactly **one** additional exact host string (never "any non-loopback host")
on top of loopback, for both the `Host` and `Origin` header checks; a mismatched, rebinding,
or substring-spoofed host is still rejected (the PR's own new test file,
`requestGuard.test.ts`, asserts all three attack shapes explicitly). `SKILL.md`'s new "LAN
access (opt-in)" section adds an explicit security note: the Host/Origin guard defends
against a hostile web page / DNS-rebinding reaching the dashboard from the browser, but does
**not** authenticate LAN devices — any device on the LAN that can reach the port can trigger
declared runs once `DASHBOARD_HOST` is set. **Empirically re-confirmed against the actually-
running dashboard server** (PID 83512, `127.0.0.1:4173`, this machine, this session):
`curl` with `Origin: http://evil.com` returns `403`; a same-origin request returns `200`;
`ps eww` on the live process shows no `DASHBOARD_HOST` in its environment — matching the
"unset = today's behaviour, unchanged" claim exactly, on the real running process, not just
read from source. A plan that assumes the dashboard is unreachable off-localhost is safe
under today's default and under #179 as designed (opt-in, single-exact-host, tested against
rebinding) — the invariant is preserved, not broken, once #179 lands; it becomes user-toggleable
rather than fixed.

**6. The button→claude spawn boundary is enforced by a hardcoded literal, confirmed.**
`(verified)` `run/route.ts` calls `spawnImpl("claude", argv, {...})` with a literal string,
not a config-derived value — this is what makes the "a button can only ever spawn `claude`,
never `bin/rachel` or any other binary directly" claim in [[dashboard]] and
[[pr_175-176_crack-on-gate-and-inbox-brief-button]] a real code invariant, not just a
documented convention. The inbox-brief button's "thin dispatcher" design (spawn `claude`
→ prompt tells it to Bash-shell into `bin/rachel`) is the only way around this, exactly as
the wiki says.

**7. Cross-PR constraints from the closed 2026-07-07 investigation
([[dashboard-run-log-streaming-viewer-gap_2026-07-07]]) still hold and were re-verified
here for the T10/T12 work, informally:** the overlay reads already-projected `output` text
passed down as a prop — it does not itself touch `runId`, the SSE bus, or any file path,
so the strict-ID-validation and token-non-leakage constraints that page established are
untouched by the modal/markdown refactor. This was not a formal re-audit; noted as a
reasonable inference from reading the component, not a line-by-line re-verification of
every constraint against #172/#174's full diff.

## Adversarial review

Not run as a separate phase — this is a read-only wiki-query answer, not a design
requiring Pre-Parade/Premortem/Red Team. The advisor pass (used mid-investigation)
caught that the raw/clean toggle needed direct verification rather than being assumed
still-current from the pre-T10 wiki text; that check is folded into Finding 2 above.

## Resolution

Not fixed here — this page documents the gap for `/wiki-ingest` to close. The five
merged PRs (#170-#174) need proper source pages and a rewrite of [[dashboard]]'s Run
Output viewer section; `/wiki-lint` should also catch the stale 80-char-truncation and
raw-toggle claims once ingest runs. Filed as an investigation, not actioned as an ingest,
per this session's scope (answer a query, not perform ingest).

## See also

- [[dashboard]] — the skill page whose Run Output viewer section is now stale
- [[pr_80-82_dashboard-stream-run-output-viewer]] — the original streaming build; still
  accurate for the SSE/`streamJson.ts`/`runOutputBus` backend plumbing, which #172/#174
  did not touch
- [[pr_124_dashboard-run-output-result-extraction]] — still-accurate server-side
  `extractResultText` behavior; the client no longer skips re-projecting its output
- [[pr_139-141_dashboard-ask-enter-clean-output]] — superseded by #172: the `showRaw`
  toggle and inline `<pre>` box it introduced no longer exist
- [[pr_163-168_dashboard-rethink]] — the loop these five orphaned PRs belong to; that
  page documents only t1-t6 (#163-#168), not t9-t12 (#171-#174)
- [[pr_175-176_crack-on-gate-and-inbox-brief-button]] — the inbox-brief button; its
  "profile-bypass-can-only-spawn-claude" claim reconfirmed directly against
  `route.ts`'s literal `spawnImpl("claude", ...)` call
- [[dashboard-run-log-streaming-viewer-gap_2026-07-07]] — the original cross-PR
  constraints (Origin guard, strict-ID validation, never-throw, token non-leakage,
  long-lived-POST/lock coupling) this investigation spot-checked against the newer work
- [[enforcement-model]] — the headless-run exemption's ceiling framing, referenced for
  the `CODERAILS_HEADLESS_RUN` ceiling note, unaffected by this investigation's findings
