---
title: "PR #128 (2026-07-10) — dashboard 'auto' permission profile for the ask button"
type: source
created: 2026-07-10
last_updated: 2026-07-10
sources: []
tags: [dashboard, argv, config, permission-mode, ask-button, headless, tool-calls, reliability]
---

# PR #128 (2026-07-10) — dashboard "auto" permission profile for the ask button

Ingested by `/wiki-ingest` after merge. Immutable record of what changed.

## PR metadata

| Field | Value |
|---|---|
| Branch | `worktree-ask-button-auto-profile` |
| Merge commit | `7f60d96` (`blueman82/coderails`, main) |
| Files changed | `skills/dashboard/app/src/lib/config.ts`, `skills/dashboard/app/src/lib/argv.ts`, `examples/dashboard-config.json`, `skills/dashboard/app/test/argv.test.ts`, `skills/dashboard/app/test/config.test.ts` |

## What changed

`PermissionProfile` (`config.ts`) gains a fourth value: `"read-only" | "standard" | "auto" | "bypass"`
(previously three, no `"auto"`). `profileFlags()` in `argv.ts` maps it to a new CLI flag pair:

```ts
if (profile === "auto") return ["--permission-mode", "auto"];
```

`examples/dashboard-config.json`'s "ask" button (`command: ""`, `inputAllowed: true` — see
[[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] for how that pattern was
introduced) switches from `"profile": "standard"` to `"profile": "auto"`. Two new tests pin the
argv shape (`argv.test.ts`: with and without input) and config acceptance (`config.test.ts`:
`loadConfig` accepts a button declaring `profile: "auto"`).

## Why: the failure mode this closes

In headless `claude -p` mode, `"standard"` adds **no** permission flags at all — it was already
documented ([[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]) as effectively an
"inherit whatever the target project's settings allow" no-op for the dashboard's spawn. Any tool
requiring a **fresh** permission grant (e.g. `WebSearch`, first use in that context) then blocks
indefinitely on a prompt nobody can answer — headless, there's no terminal to answer it — and the
model silently falls back to answering from memory, with **no visible error and no tool call ever
attempted**. This is exactly the failure mode a real user report exposed for the "ask" button.
`--permission-mode auto` auto-approves permission decisions instead of blocking on them, closing
that specific hang.

## The honest limit: "auto" does not fix tool-call non-determinism

`argv.ts:40-49` carries this finding verbatim as a code comment, added in this PR:

> "auto" is not a guarantee of tool access — confirmed empirically on this machine 2026-07-10:
> three consecutive `claude -p --permission-mode auto` runs against a web-search-requiring
> question succeeded once and fell back to "I don't have internet access" (no tool call attempted
> at all) twice, with no permission-block message in either failing run. This mirrors the same
> non-deterministic "does the model attempt the tool call" behavior already documented for the
> "standard" profile — "auto" changes what happens if a tool call is attempted and needs a
> permission decision (auto-approved rather than blocked waiting on an unanswerable headless
> prompt), it does not make the model reliably choose to attempt the call.

Direct empirical terminal testing this session reproduced exactly that split: **1 real tool call
out of 3 consecutive identical runs**; the other 2 fell back to "I don't have internet access"
with no tool call attempted — the same silent-fallback symptom `"standard"` produces, just without
a permission-block as the cause. **This PR fixes "blocks on an unanswerable permission prompt."
It does not fix "the model sometimes doesn't try the tool call at all"** — that non-determinism
predates this PR and was already documented for the "ask" button and time-sensitive questions in
[[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]. Any future work claiming
"ask button tool calls are reliable" needs to close this second, still-open gap — this PR does not
close it.

## Superseded prior-state claims

Two existing wiki pages asserted the "ask" button ran on `profile: "standard"` — true at the time
they were written, now superseded by this PR:

- [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] (source page, left
  unmodified per immutable-record convention — see its own header note about not rewriting past
  state)
- [[dashboard]] (living page, updated by this ingest — see below)

## Review process: third occurrence of independent-reviewer convergence

Two independent PR review agents — `code-reviewer` and `pr-test-analyzer` — both flagged the same
test-coverage gap (no test exercising an `"auto"`-profile button combined with free-text input)
without seeing each other's output. This is the **third** time in this project's history that
independently-run review agents have converged on an identical finding unprompted:

1. [[pr_39_agentic-loop-slim-v2]] — main context + a blind Explore agent, both converged on "zero
   replaceable passages" against the same four-part test
2. [[pr_124_dashboard-run-output-result-extraction]] — two independent review agents converged on
   the identical `extractResultText` coverage gap (multi-result-line + non-string-result branches)
3. This PR — `code-reviewer` + `pr-test-analyzer` converged on the same auto+free-text
   test-coverage gap

Consistent with the prior two occurrences: evidence the parallel-review pattern reliably catches
real coverage holes a single reviewer's fixture/test choices can miss, not a one-off coincidence.
No dedicated wiki page exists for this meta-pattern; each occurrence is recorded on its own source
page and cross-linked, per this session's scope (not creating a new page for a pattern with three
data points spread across otherwise-unrelated PRs).

## Wiki pages updated

- [[dashboard]] — profile→flag mapping table gains `auto`; "ask" button description updated from
  `profile: "standard"` to `profile: "auto"`, with a forward-pointer note explaining the
  supersession and the tool-call non-determinism caveat

## See also

- [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] — introduced the "ask"
  button pattern (`command: ""`, `inputAllowed: true`) on `profile: "standard"`, and the argv
  merged-prompt fix this button pattern depends on
- [[dashboard]] — living architecture page, profile→flag mapping now includes `auto`
- [[pr_124_dashboard-run-output-result-extraction]] — the second occurrence of independent-reviewer
  convergence this page's finding is the third instance of
