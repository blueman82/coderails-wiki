---
title: "PR #290 — SSE aggregator teardown on abort, and the jq valid-scalar object guard"
type: source
origin: "coderails PR #290 (merged f28d7b7, 2026-07-24; head 8ed1dff)"
created: 2026-07-24
last_updated: 2026-07-24
sources: [skills/dashboard/app/src/app/api/events/route.ts, skills/dashboard/app/test/eventsTeardown.test.ts, launchd/com.coderails.dashboard.plist, hooks/scripts/lib/discipline_common.sh, hooks/scripts/tests/discipline_common.test.sh, skills/dashboard/app/src/lib/collect/contextTrend.ts, skills/dashboard/app/src/components/ContextTrendPanel.tsx]
tags: [dashboard, sse, teardown, fd-leak, launchd, maxfiles, abortsignal, idempotent-release, jq, object-guard, valid-scalar, discipline-hooks, context-trend, constant-justification]
---

# PR #290 — SSE aggregator teardown on abort, and the jq valid-scalar object guard

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #290 |
| Merged | 2026-07-24 (00:15:25Z) |
| Merge commit | `f28d7b7ece9e259290a72d144718a1d0c688ace3` |
| Head commit | `8ed1dff309a10e9952f385a6792a2cb43d75c199` |
| JIRA ticket | — |

Three concerns in one PR, related only by the session that found them. Listed in
order of weight: the SSE file-descriptor leak (the title), the jq valid-scalar
object guard (which closes a wiki `⚠️ CONTRADICTION` open since 2026-07-17), and
a comment rewrite that retires a phantom spec citation.

---

## 1. The SSE aggregator leak — `cancel()` is not a disconnect signal

### The defect

`skills/dashboard/app/src/app/api/events/route.ts` tore down its per-connection
aggregator **only** from `ReadableStream.cancel()`. That is the wrong hook for
the common case: `cancel()` fires when the response *consumer* cancels, and a
client that simply goes away — tab closed, network drop, `curl` killed — does
not reliably trigger it (verified — PR body's runtime probe; see below).

Each abandoned connection therefore leaked a whole aggregator: a recursive
`fs.watch` handle on **each** of `projectsDir`, `loopsDir`, `runsDir`,
`queueDir`, `buildsDir`, plus the gates `setInterval`. One full set per
connection, never released. (verified — `route.ts` diff + test-file header)

### Why it was fatal rather than untidy — the launchd descriptor ceiling

The compounding fact, and the reason this is a design note rather than a
bugfix: **a launchd-spawned process inherits `launchctl limit maxfiles`
(256 on a stock macOS), not the interactive shell's much higher soft limit.**
The dashboard runs under launchd ([[pr_88_93_dashboard-launchd]]), so the leak
had a ceiling of a few page loads, not thousands. (verified — plist comment)

The failure mode past that ceiling is *silent and misleading*: the process
still accepts TCP but can no longer open files or watches, so it wedges —
HTTP 000 on every route, zero SSE frames, panels stuck on "loading…" forever.
Nothing crashes and nothing logs; the supervisor still reports the job as
running. This is the same shape as the standing lesson that a supervisor's
"running" is not "serving" (see [[dashboard]]'s start/stop section) — here the
process is genuinely alive and genuinely useless.

### The fix — an idempotent `release()` reachable from three paths

```ts
let released = false;
const release = () => {
  if (released) return;
  released = true;
  unsubscribe?.();
  aggregator.stop();
};
request.signal?.addEventListener("abort", release, { once: true });
```

`cancel()` now delegates to `release()` rather than tearing down itself. The
guard flag is load-bearing because both paths can fire for one connection.

The third path is the subtle one, and it is the part worth remembering: **a
signal that was already aborted before the handler ran has fired its `abort`
event in the past, so a listener registered inside the handler never fires.**
The client can drop between connection-accept and handler execution. So the
route re-checks `request.signal?.aborted` at the end of `start()`. Placement is
deliberate and commented: checking any earlier would stop a not-yet-started
aggregator and leave the subscription dangling — the check must sit *after* the
aggregator is started and `unsubscribe` is bound, so `release()` has something
real to tear down. (verified — `route.ts` diff comment)

Generalises to: **registering a listener on an `AbortSignal` is not sufficient;
an already-aborted signal needs an explicit post-registration re-check.**

### Defence in depth — raising the ceiling too

`launchd/com.coderails.dashboard.plist` gains:

```xml
<key>SoftResourceLimits</key>
<dict><key>NumberOfFiles</key><integer>4096</integer></dict>
```

The plist comment is explicit that this is *defence in depth*, not the fix —
the per-connection leak that made 256 reachable is fixed in the route. Both
layers shipped together, which is the right disposition: the limit raise alone
would have converted a fast wedge into a slow one.

### Empirical proof, not just a green suite

The PR body records an instrumented run of the built server — 3 rounds of 6
abandoned SSE connections (18 total): fds flat at 39 every round, project-watch
fds returning to 0 every round, and a release firing for every connection
(7/13/19 cumulative). Measured against the **unfixed live server** for contrast:
114 fds held, 69 of them on the projects dir, climbing 115 → 120 → 125 across
connections and never returning — that server was serving HTTP 000 with zero SSE
frames, i.e. the wedge observed in the wild, not hypothesised.

A temporary probe also logged `signal present: true` and `RELEASE fired` on
client disconnect, which is the actual evidence for the two premises the fix
rests on: Next.js *does* abort `request.signal` for an abandoned connection, and
`ReadableStream.cancel` alone was *not* running. The probe was removed before
the final push. This is the [[verification-before-completion]] discipline applied
to a resource leak — a leak's absence is not observable from a passing unit test,
only from counting descriptors on a real process.

### Test coverage

New `skills/dashboard/app/test/eventsTeardown.test.ts` (115 lines) with four
cases, each pinned to a distinct path: abort with **no** stream cancel at all
(the exact path that leaked); abort *and* cancel both firing, asserting exactly
one `stop()`/`unsubscribe()`; an **already-aborted** signal at handler entry;
and a request carrying **no** signal, which must still tear down via `cancel()`
so the old path is not regressed away.

---

## 2. The jq object guard — a *valid* scalar is a different hazard from a malformed line

This half closes a `⚠️ CONTRADICTION` that had been open on
[[discipline-loop]] since 2026-07-17, re-verified as still-live by the
2026-07-24 full-vault lint. That flag said, correctly, that
`dc_extract_last_text` was **not** hardened by the earlier jq-slurp work
despite the page claiming it was.

### Two distinct failure modes, easily conflated

`hooks/scripts/lib/discipline_common.sh` parses transcripts with a two-stage
`jq -R 'fromjson? // empty' | jq -s ...` pipeline that defends against a
**malformed** line: `fromjson?` drops what does not parse, so one bad line no
longer collapses the whole slurp. That shape reached `dc_extract_last_text` in
[[pr_112-113_2026-07-08_jq-slurp-residuals-round2|PR #112]] and
`dc_file_count` in [[pr_156_dnv-presence-check|PR #156]] — so *both* functions
were already tolerant of malformed lines before #290, and the earlier work was
accurate about what it added.

It does **not** defend against a line that is *valid JSON but a bare scalar* —
`42`, or `"a bare json string"`. `fromjson?` **keeps** those, they reach
`.type`, and jq errors out (`Cannot index string with string "type"`). Because
stderr is discarded, that error is **silent**: the count comes back 0 or the
extraction comes back empty, and the caller reads it as "no files touched" /
"no text yet" rather than as a failure. (verified — diff comments + new tests)

So the earlier work and this work are not the same guard. The wiki's
CONTRADICTION was right that `dc_extract_last_text` was unguarded, and the page
it flagged was right that the function had been touched by an earlier PR — both
true, because #112 gave it the *malformed-line* tolerance and left the
*valid-scalar* hazard. Two hazards sharing one pipeline are easy to describe as
one.

### The fix

`select(type == "object")` added to both functions:

- `dc_file_count` — `. as $lines` → `map(select(type == "object")) as $lines`
- `dc_extract_last_text` — a `| select(type == "object")` before
  `select(.type == "assistant")`

Same guard, same reason, as `als_extract_last_text` in `loop_state_common.sh`,
which got it in [[pr_206_208_loop-state-common-docs-and-robustness|PR #208]].
PR #208 explicitly left `dc_extract_last_text` as "a separate, already-tracked,
parked concern" — #290 is the PR that finally lands it.

Two new tests, (k) and (l), each of which interleaves scalar lines (`42`,
`"a bare json string"`) with genuine assistant turns and asserts the real
answer survives — count 2 rather than 0, and `REAL ANSWER` rather than empty.

### Status of the family — and a scope correction worth recording

Verified against `origin/main` at ingest: the **object guard** is present in
`dc_file_count` (line 39) and `dc_extract_last_text` (line 74). Those are the
only two functions #290 touched.

`unregistered_loop_guard.sh`'s `ulg_count_dispatch_turns` carries its own
two-stage **malformed-line** tolerant parse (lines 65/91) from
[[pr_112-113_2026-07-08_jq-slurp-residuals-round2|PR #113]], but **no**
`select(type == "object")` — #290 did not touch that file.

> ⚠️ **OPEN DEFECT, found during this ingest (2026-07-24): the same
> valid-scalar hazard is still live in `ulg_count_dispatch_turns`.**
> Its stage-2 slurp is `[ .[]? | select(.type == "assistant") | ... ]` over an
> unfiltered `fromjson? // empty` stage 1 — the exact unguarded shape #290 just
> removed from both `discipline_common.sh` functions. Reproduced against a
> 3-line fixture (`{assistant/Agent}` / `42` / `{assistant/Agent}`): the
> function's real pipeline emits `jq: error … Cannot index number with string
> "type"` and returns **0**, while the same expression with
> `select(type == "object")` returns the correct **2**. (verified — executed at
> ingest) The failure is silent twice over: stderr is discarded, and the
> trailing `case "$n" in (''|*[!0-9]*) n=0;;` coerces the empty result to `0`,
> so `ULG_PARSE_REASON` stays **empty** — the guard reports "no dispatch turns"
> rather than a parse failure, which is precisely the attribution distinction
> PR #113 built. Impact is bounded: nudge-only hook, so the consequence is a
> missed nudge, not a gate failure. Not fixed here — #290 is the PR being
> ingested, not a place to land new code. **This is the third member of the
> family and it is not guarded.**

> ✅ **CLOSED 2026-07-24 by PR #293** — the note above was true when written and
> was overtaken within the hour. `gh pr view 293`: state **MERGED**, `mergedAt`
> `2026-07-24T01:02:56Z`, merge commit `4340c4b`, branch
> `fix/context-trend-panel-styles`, files `unregistered_loop_guard.sh` (+38/−1),
> `tests/unregistered_loop_guard.test.sh` (+39), `styles/hud.css` (+66)
> (verified). `git show origin/main:hooks/scripts/unregistered_loop_guard.sh`
> now carries `select(type == "object")` at **line 105**, inside
> `ulg_count_dispatch_turns`'s stage-2 slurp (verified). Re-run of **the same
> 3-line fixture this note cites** (`{assistant/Agent}` / `42` /
> `{assistant/Agent}`) against current `origin/main` returns **2**, not `0`
> (verified — executed 2026-07-24). The valid-scalar hazard in
> `ulg_count_dispatch_turns` is closed. Why #293 landed it: its title is the
> dashboard CSS work, so the guard is invisible from the PR title — the reason
> the note above, written from PR metadata, read the PR as unrelated.
>
> **This closes the valid-scalar hazard only. It is not family-wide closure**,
> and the distinction is the same one this page exists to keep. A valid JSON
> **object of the wrong inner shape** — e.g. `{"type":"assistant","message":"oops"}`,
> where `.message` is a bare string — passes `select(type == "object")` and then
> aborts the slurp on `.message.content`. That hazard is **still open on
> `origin/main`** in `dc_file_count` and `dc_extract_last_text`: fixture
> `{genuine assistant edit turn}` + `{"type":"assistant","message":"oops"}`
> gives `dc_file_count` → **0** (should be 1) and `dc_extract_last_text` →
> **empty** (should be the text), both silently (verified — executed against
> `git show origin/main:hooks/scripts/lib/discipline_common.sh`, 2026-07-24).
> A fix is in flight, unmerged at time of writing: **PR #295** ("discipline
> common rc capture", branch `fix/discipline-common-rc-capture`, opened
> `2026-07-24T01:43:54Z`, touching `discipline_common.sh` +94/−39 and its test
> file +73) (verified). No wiki-link here deliberately — an unmerged PR has no
> `sources/` page to link to.

**On the memory handoff `project_jq_slurp_round2_handoff`:** that handoff is
*not* about this hazard and is not closed by this PR. Read directly at ingest,
its "2 remaining bare `jq -s` slurps" are `dc_extract_last_text` **and**
`ulg_count_dispatch_turns` (not `dc_file_count`), and its stated fix shape is
the **malformed-line** two-stage parse mirroring `als_extract_last_text` — work
that PRs #112/#113 completed on 2026-07-08. #290 closes a *different, later*
hazard on one of the same functions. Recorded because the two are easy to
conflate by function name: same file, same pipeline, different failure.

---

## 3. Retiring a phantom spec — the ContextTrend constants get in-repo justification

[[pr_287_dashboard-context-trend-panel|PR #287]]'s ingest flagged that the
Context Trend collector and panel both cited `docs/TOKEN-REDUCTION-AUDIT.md` as
their spec, and that **no such file was tracked on any ref** — leaving
`CUTOVER_MS`, `WINDOW_START_MS` and `MIN_READABLE_N` with no in-repo
justification for a future reader re-tuning them. That was carried forward by
the 2026-07-23 lint as owed ingest-class work. #290 resolves it.

The rewrite does three things, and the *method* is the transferable part:

1. **Says the file never existed.** The new comment records plainly that "no
   such file has ever existed on any ref" — so a future reader does not repeat
   the hunt. Naming a dead reference as dead is cheaper than deleting it.
2. **Re-anchors the verdict to itself, not to a document.** Every "the audit's
   verdict is INDETERMINATE" is replaced with a first-person statement of the
   same epistemic state plus its *reason*: "whether those measures reduced
   token burn is NOT established: the before and after groups differ in size
   and composition, and no controlled comparison was run." The refusal to
   compute a headline saving is unchanged — it is now justified by the data's
   own shape rather than by an absent authority.
3. **Gives each constant its own checkable justification.** `CUTOVER_MS` now
   cites the three real merges (#228/#229/#230 at `20:22:29Z`, `20:25:46Z`,
   `20:27:27Z`) and names the command that checks it
   (`gh pr view 228 --json mergedAt`) — a repo event, not a chosen date.
   `WINDOW_START_MS` states what it buys (≈10 days back, a before-span
   comparable to the after-span, not an unbounded tail) and what it does not do
   (selects which sessions are plotted; never weights or adjusts one).
   `MIN_READABLE_N = 20` is re-labelled honestly as **a presentation threshold,
   not a statistical test** — a conservative round number sitting above the
   ~10-session after-group that first prompted the caveat.

The panel comment also swaps its "row 1 was documented as the largest saving"
phrasing for a checkable in-repo pointer: row 1 is *expected* to save the most
and has been inert since it shipped, **per PR #273, which removed its gate
after it fired zero times**.

Generalises to: **when a cited spec turns out not to exist, the repair is not
to delete the citation — it is to move each claim the citation was carrying
onto its own in-repo evidence, and to say the spec is gone.** A constant whose
only defence is "the audit said so" is unfalsifiable once the audit cannot be
read.

---

## What this PR did *not* do

- It did not add a regression guard for the launchd descriptor limit itself —
  `SoftResourceLimits` is a plist value with no test asserting it stays. If a
  future plist edit drops it, the leak fix still holds, but the defence in
  depth silently goes.
- It did not touch the other collectors' watch handles; the fix is at the
  aggregator lifecycle, so it covers them by construction rather than
  individually.
- It did not resolve the underlying question the Context Trend panel exists to
  ask. Whether the 2026-07-17 cutover reduced token burn remains **not
  established** — #290 makes that honest rather than deferred to a document,
  which is a different thing from answering it.

## See also

- [[dashboard]] — the skill page; Context Trend and SSE architecture sections
- [[discipline-loop]] — where the `dc_extract_last_text` CONTRADICTION lived
- [[pr_287_dashboard-context-trend-panel]] — the panel this PR revisits; same
  `/api/events` route, same constants
- [[pr_228_229_230_token-burn-reduction-and-agents-split]] — the cutover the
  collector bins against
- [[pr_206_208_loop-state-common-docs-and-robustness]] — the sibling object
  guard in `loop_state_common.sh`, landed 2 PRs' worth of lag earlier
- [[pr_156_dnv-presence-check]] — the two-stage tolerant parse this guard sits
  on top of
- [[pr_88_93_dashboard-launchd]] — the launchd supervision that supplies the
  256-descriptor ceiling
- [[check_verify_loop]], [[check_confidence_labels]] — the two consumers of
  `discipline_common.sh` that a silent-empty extraction would have degraded
