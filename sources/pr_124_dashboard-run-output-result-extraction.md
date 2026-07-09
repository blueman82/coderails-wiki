---
title: "PR #124 — dashboard Run Output panel: extract final result instead of raw stream-json"
type: source
created: 2026-07-09
last_updated: 2026-07-09
sources: []
tags: [source, dashboard, run-log, streaming, bugfix, task-evals]
---

# PR #124 — dashboard Run Output panel: extract final result instead of raw stream-json

Ingested by `/wiki-ingest` after merge. Immutable record of what changed.

Closes the caveat [[pr_80-82_dashboard-stream-run-output-viewer]] flagged at the time: "the
stream-json parser's output is not consumed for anything beyond well-formed-or-gracefully-skipped
verification today... nothing in this cluster requires it." A live user report supplied the
"requires it" case — asking the dashboard's `ask` button "what time is it in Dublin" produced a
genuinely correct answer that was unfindable on screen, because the Run Output panel rendered the
entire raw `stream-json` transcript (every hook/system/assistant event, thousands of characters)
with the actual answer buried in a nested JSON blob at the very end.

## PR metadata

| Field | Value |
|---|---|
| PR #124 | `fix/dashboard-run-output-parse-result`, merged 2026-07-09, SHA `1005d6bc7a15a061748b49a5de4c9b584300063b` (merge commit `c9db6da`) |
| Repo | `blueman82/coderails` |
| Process | `systematic-debugging` (root cause, prior session) → `task-evals` (pr scope, tier 1, frozen before implementation) → `test-driven-development` → `push` → `review-pr` → `post-review` → `post-evals` → `merge` |

## Summary

**Root cause** (confirmed prior session, not re-litigated here): `GET /api/run/output`
(`skills/dashboard/app/src/app/api/run/output/route.ts`) read a completed run's `.log` file and
returned its full raw content as `output`, with zero parsing. The file is
`claude -p --output-format stream-json` JSONL — per [[pr_80-82_dashboard-stream-run-output-viewer]],
the final answer lives in the `result` field of the last `type:"result"` line, but nothing read
that field; the client-side `OutputViewerPanel.tsx` just dumped whatever the API gave it into a
`<pre>` block verbatim.

**Fix.** A new module-private `extractResultText(rawLog: string): string` helper in `route.ts`
reuses the existing `parseStreamJsonLine` from `skills/dashboard/app/src/lib/streamJson.ts`
(shipped in PR #80, previously unused beyond the never-throws guarantee) — the first real
consumer of its parsed `{ok, value}` shape. Splits the log on newlines, scans **backwards**, and
returns the first line where `parsed.ok && value.type === "result" && typeof value.result ===
"string"`. Falls back to the raw log content if no such line is found — deliberately: a
crashed/killed run that never reached a `type:"result"` event should still show whatever partial
output exists rather than an empty panel, and this preserves the pre-fix behavior for that case
by construction (same fallback value).

```ts
function extractResultText(rawLog: string): string {
  const lines = rawLog.split("\n").filter((line) => line.trim() !== "");
  for (let i = lines.length - 1; i >= 0; i--) {
    const parsed = parseStreamJsonLine(lines[i]);
    if (parsed.ok && parsed.value.type === "result" && typeof parsed.value.result === "string") {
      return parsed.value.result;
    }
  }
  return rawLog;
}
```

Call site: `output = extractResultText(readFileSync(record.outputPath, "utf-8"))`, replacing the
prior bare `readFileSync(...)` assignment. One import added, one function, one call-site swap —
no other code touched.

## Task-evals (tier 1, GO)

Frozen before implementation per `task-evals` (pr scope). 3 P0 + 2 P1, all pass:
- E1 — extracts the last `type:"result"` line's `result` field, not raw content. Negative control
  confirmed genuine failure against pre-fix code.
- E2 — output contains no raw JSONL markers (`"type":"system"` etc). Negative control confirmed.
- E3 — no-result-line log falls back cleanly, doesn't throw (intentionally identical pre/post-fix).
- E4 — full existing `runOutputRoute.test.ts` suite green, no regression in auth/origin/
  validation/404/409 paths.
- E5 — the pre-existing path-traversal regression-guard test (no `join(...runId...)` in source)
  still passes; the fix adds no new path-join.

One amendment recorded pre-merge: a frozen eval's `-t` filter referenced a placeholder test name
that didn't match; corrected to the real name at grading time, assertion unchanged.

## Review findings, addressed pre-merge

`pr-review-toolkit:review-pr` ran `code-reviewer` + `pr-test-analyzer` in parallel. Both
independently surfaced the same two gaps in the initial (14-test) submission — the frozen evals'
fixtures only ever had a single `type:"result"` line, so two branches of `extractResultText` were
unexercised:

1. **Multi-result-line last-match selection** — a log with more than one `type:"result"` line
   (e.g. a resumed/retried run) must return the *last* one, not the first.
2. **Non-string `result` field** — a `type:"result"` line with a non-string `result` (e.g. an
   `error_during_execution` subtype, `result: null`) must fall through to the raw-content
   fallback, not crash or silently coerce.

Both closed same session: two tests added to `runOutputRoute.test.ts`, each first verified against
a naive buggy implementation (forward-scan-first-match, no `typeof` guard) to prove they weren't
vacuous, then confirmed green against the real fix (16/16 total). This is the second time in this
project's history two independently-run review agents converged on the identical gap
unprompted — worth noting as evidence the parallel-review pattern catches real coverage holes a
single reviewer's fixture choices can miss.

## Files changed

- `skills/dashboard/app/src/app/api/run/output/route.ts` — `extractResultText` helper + call-site
  swap
- `skills/dashboard/app/test/runOutputRoute.test.ts` — 1 existing test rewritten (was asserting
  raw-passthrough, now asserts parsed-result), 5 new tests (result extraction, marker
  non-leakage, no-result fallback, multi-result last-match, non-string-result fallback)

## Wiki pages updated

- [[pr_80-82_dashboard-stream-run-output-viewer]] — caveat section note added: closed by this PR
- [[dashboard]] — Run Output panel description updated to reflect parsed-result display

## Caveats / gotchas

- `extractResultText` is module-private (not exported) — no other call site exists or is
  anticipated; if the dashboard ever needs the same extraction elsewhere (e.g. a future
  routines-runner summary view), this logic should move to a shared lib rather than being
  duplicated, per the project's DRY convention.
- The fallback-to-raw-content behavior means a genuinely crashed run still shows the full
  stream-json dump in the panel — this PR only fixes the common case (a settled run that reached
  a result event). Left as-is: out of scope for the reported bug, and arguably correct (partial
  raw output beats an empty panel for debugging a crash).
