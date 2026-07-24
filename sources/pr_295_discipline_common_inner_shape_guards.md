---
title: "PR #295 — inner-shape guards + an rc-capture net for discipline_common's jq slurps"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - hooks/scripts/lib/discipline_common.sh
  - hooks/scripts/tests/discipline_common.test.sh
  - sources/pr_290_sse_teardown_and_jq_object_guard.md
tags: [source, hooks, discipline-common, jq-slurp, fail-open, check-verify-loop, inert-scaffolding]
---

# PR #295 — inner-shape guards + an rc-capture net for discipline_common's jq slurps

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #295 |
| Branch | `fix/discipline-common-rc-capture` |
| Merged | 2026-07-24 (`2026-07-24T08:33:04Z`) |
| Merge commit | `4a5c2375757128a98a64cebb59f59d02a9c52625` |
| Head commit | `87a064737b6efaa4db2d1b6bf5f0baf3d7a5fbe3` |
| JIRA ticket | — |

## Summary

Closes **part 3** of the jq-slurp fragility family in
`hooks/scripts/lib/discipline_common.sh`: the **wrong-inner-shape** hazard.
Two files, `dc_file_count` and `dc_extract_last_text` plus their tests.

This is the hazard [[discipline-loop]] recorded as *open on `origin/main`* and
[[pr_290_sse_teardown_and_jq_object_guard|PR #290]]'s page recorded as *a fix
in flight, unmerged*. Both records are now closed — see
[[discipline-loop]]'s Part 3 for the current status.

**It extends #290's guard, it does not supersede it.** #290 added
`select(type == "object")`, which stops a top-level **scalar** line aborting
the stage-2 slurp. #295's own comment states plainly that this is "necessary,
not sufficient": a line can be a valid JSON **object** and still abort, if its
*inner* shape is wrong — `{"type":"assistant","message":"oops"}` passes the
object guard, then `.message.content` indexes a bare string and jq errors out,
taking the whole slurp with it. Same silent signature as parts 1 and 2: stderr
discarded, count coerced to 0, extraction returns empty, and the caller reads
"no files touched" / "no text yet" rather than "the parse failed".

## The two-layer shape

The PR's durable contribution is the **layering**, not either guard alone.

**Layer 1 — recovery, and the primary mechanism.** Inline shape guards where
the slurp indexes: `((.message | type) == "object")` before touching
`.message.content` (in both functions, and inside `is_genuine_user` on the
user-line path), plus `select(type == "object")` on each *content element*
before indexing `.type`/`.name`. These are **per-line recovery**: the slurp
completes and the count/text is recovered from the surviving lines, so one bad
line costs one line.

**Layer 2 — a trigger-independent net, explicitly not the primary.** Stage 2
was split into a captured intermediate (`$tolerant`) and its own command
substitution so `agg_rc=$?` can be read at all — the shape [[pr_274_tier_gate_observability_fixes|PR #274]]'s
fix 3 called *capture-first*, and the same shape PR #293 gave
`ulg_count_dispatch_turns`. It is **whole-slurp**, not per-line: on an abort
the surviving lines are lost too, which the PR states as a deliberate trade.
Its purpose is the hazards nobody has enumerated — tests (p) and (q) reach it
with shapes no Layer 1 guard covers (a `tool_use` whose `.input` is a bare
string; a text block whose `.text` is an object).

## Why Layer 2 writes nothing to stderr

`ulg_count_dispatch_turns`'s equivalent net sets `ULG_PARSE_REASON` and echoes
it. #295 deliberately does **neither** here, and the reason is a real
interaction, not a style choice: `dc_file_count` is called **unconditionally
on every Stop-hook turn** from `check_verify_loop.sh`, *ahead of* that hook's
own block-message write to the **same stderr stream** (`>&2` + `exit 2`). An
attribution echo would land concatenated ahead of the model-facing block
message on every blocked turn where the hazard existed anywhere in the
transcript — visible noise for zero observability gain, since nothing consumes
the token. `dc_file_count` has exactly one direct caller and nothing reads a
reason global. (verified — the function's own comment, and
`check_verify_loop.sh`'s call site)

The generalisable point: **an attribution channel is only worth adding where
something reads it, and where it doesn't collide with a channel that is already
load-bearing.** The same fix shape (rc capture) was correct in
`unregistered_loop_guard.sh` — a nudge-only hook that writes nothing else to
stderr — and wrong here.

## Open finding — Layer 2 is behaviourally inert as shipped

Recorded because it is exactly the class of thing a green test suite hides.

With the stderr write dropped, the abort branch reduces to `printf '0'; return`
(and `printf ''; return` in `dc_extract_last_text`). That is **byte-identical
to what the fall-through already produces**: on an abort `n` is empty, and the
trailing `case "$n" in (''|*[!0-9]*) n=0;; esac` laundering coerces it to `0`
anyway; `dc_extract_last_text`'s `text` is already `""`. The branch has no
observable effect.

Verified by mutation against merged `origin/main`, not by reading: deleting
**both** early-return branches outright and re-running the exact fixtures tests
(p) and (q) use gives identical output — `dc_file_count` → `0` and
`dc_extract_last_text` → empty, before and after (verified — executed at ingest
2026-07-24 against `git show origin/main:hooks/scripts/lib/discipline_common.sh`).

So tests (p) and (q) do pass, and they do describe the right *behaviour*
(fail open, no stderr), but they **do not discriminate the mechanism they are
named for** — they would pass with the Layer 2 scaffolding removed. This is
the [[pr_216_217_safe-routes-and-cost-miner-diagnostics|"a check that cannot
both pass and fail is not a check"]] shape, one level up: the check is real,
the *attribution* is not.

Contrast `ulg_count_dispatch_turns`, where the same net **is** live: it returns
before the laundering `case` **and** sets `ULG_PARSE_REASON`, so its abort is
distinguishable from a genuinely quiet session.

**Disposition owed** (not decided here, this is a source record): either remove
the dead scaffolding and keep only the Layer 1 guards, or give Layer 2 an
observable effect that doesn't collide with `check_verify_loop.sh`'s stderr
(a distinct exit path, or a reason global something actually reads). Relabelling
tests (p)/(q) as "fail-open behaviour" rather than "Layer 2 net" would at least
stop them attesting to a mechanism they don't exercise. Not a live defect — the
fail-open outcome is correct either way, which is precisely why it went
unnoticed.

## Tests

Seven new checks, (m) through (q). The distinction worth keeping is the one
between the two hazard depths, which the PR's own test comments spell out:
(m)/(o)/(n) exercise a wrong-shaped **`.message`**; (o2)/(n2) exercise a
wrong-shaped **content element** *inside* an otherwise-valid array — a
different guard, and each was mutation-proved to flip the corresponding test to
FAIL when removed alone. (p)/(q) reach Layer 2. Test (n)'s comment also corrects
an earlier comment that claimed to cover the element case when it covered the
`.message` case.

## Wiki pages updated

- [[discipline-loop]] — Part 3 flipped from open to closed; the two-layer shape
  and the inert-Layer-2 finding recorded there
- [[check_verify_loop]] — the stderr-collision constraint on its own caller path
- [[pr_290_sse_teardown_and_jq_object_guard]] — dated resolution note appended

## Caveats / gotchas

- **Family closure is not uniform, and saying "the family is closed" would
  repeat an over-claim this vault has already caught twice.** After #295 the
  two `discipline_common.sh` functions carry **both** Layer 1 inner-shape
  guards and a Layer 2 rc capture; `ulg_count_dispatch_turns` carries the
  scalar object guard (#293) and a **live** Layer 2, but **no** inner-shape
  Layer 1 guards. Three functions, three different treatments.
- The inert-Layer-2 finding above means the *effective* protection in
  `discipline_common.sh` today is Layer 1 alone.

## See also

- [[discipline-loop]] — the jq-slurp fragility family, parts 1-3
- [[check_verify_loop]] — the sole caller of both functions
- [[pr_290_sse_teardown_and_jq_object_guard]] — part 2, the valid-scalar object guard
- [[pr_156_dnv-presence-check]] — part 1, the malformed-line tolerant parse in `dc_file_count`
- [[pr_206_208_loop-state-common-docs-and-robustness]] — the sibling guard in `als_extract_last_text`
