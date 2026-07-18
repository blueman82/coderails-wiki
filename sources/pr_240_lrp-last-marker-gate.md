---
title: "PR #240 — loop-retro-promotion-weekly last-marker gate"
type: source
created: 2026-07-18
last_updated: 2026-07-18
sources: []
tags: [routines, artifact-gate, last-marker, loop-retro-promotion, false-green]
---

# PR #240 — loop-retro-promotion-weekly last-marker gate

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #240 |
| Branch | — (see commit below; local branch name not recorded in `gh pr view`'s default fields) |
| Merged | 2026-07-18T00:03:42Z |
| Merge SHA | `27d0c5ad857a45bbc93b1fa07b859ccd91b7b6dc` |
| JIRA ticket | — |

## Summary

Closes an append-log false-green gap in the `loop-retro-promotion-weekly` routine, the same defect *class* [[pr_207_209_docs-sync-nightly-and-drift-fix|PR #227]] closed for `sync-docs-nightly`'s run log a day earlier: an append-only, per-repo log file that accumulates many runs' terminal markers, gated by a predicate that can't tell "this run's own outcome" from "some earlier run's stale success line still sitting in the file."

Before this PR, `loop-retro-promotion-weekly`'s `expectedArtifact.predicate` was `{ "kind": "exists" }` — file present and fresh, nothing more. That predicate cannot distinguish a genuinely completed run from a run that entered delivery (mining → drafting → push → review → merge) and died partway through: the log file from a *prior* successful run is still present and still fresh, so `exists` reads PASSED regardless of what the *current* run actually did. PR #240 switches the predicate to `{ "kind": "last-marker", "success": "run=ok", "failures": ["abort=", "delivery=started"] }`, and amends the skill itself (`skills/loop-retro-promotion/SKILL.md`) to write the markers that predicate depends on:

- **Dormant-stop terminal (§1, predicate=unmet branch):** now appends `run=ok` — a dormant stop is a correct, successful no-op, and the gate must read it as green, not merely as "file exists."
- **`delivery=started` marker (§1, predicate=met branch):** written immediately after the `predicate=met...` line, **before** §2 Mining starts — not at §4 Delivery's entry. This is deliberate: it is the fail-safe in-progress marker for the *entire* met-path (mining, drafting, AND delivery), so a death anywhere after met-determination — not just a death during the delivery gate chain — leaves `delivery=started` as the log's last terminal marker, and the last-marker gate reads that run RED.
- **`run=ok` on successful delivery (§4, after step 9/merge):** the only point that writes the success terminal for a delivery run; it must come after the per-stage "merge" line so it remains the log's true *last* terminal marker for a completed run.

`examples/dashboard-config.json` (the documentation-only example config — see [[routines]] for why the live config at `~/.claude/coderails-dashboard.json` is the one that actually matters at runtime) was updated to match the new predicate shape.

## Files changed

- `examples/dashboard-config.json` — `loop-retro-promotion-weekly` routine's `expectedArtifact.predicate`: `{kind:"exists"}` → `{kind:"last-marker", success:"run=ok", failures:["abort=","delivery=started"]}`.
- `skills/loop-retro-promotion/SKILL.md` — §1 (dormant-stop `run=ok` write; `delivery=started` write ahead of §2 Mining) and §4 (clarifying note that `delivery=started` was already written in §1 and covers this section too; `run=ok` write after step 9/merge).
- `skills/dashboard/runner/test/artifactGate.test.ts` — five new RED-LOCK fixtures exercising this routine's own marker shape (`run=ok` / `delivery=started` / `abort=`), distinct from the pre-existing `docs-sync` `lastMarker` fixtures (whose failure set is `["abort=","refused="]`, no `delivery=started`). Two of the five carry an explicit **discrimination proof**: the identical fixture file re-read under the OLD `{kind:"exists"}` predicate reads PASSED — the exact false-green this PR closes — so the new-predicate assertion is proven to be a real regression lock, not a check that would have passed under either predicate shape.

## The declined sibling: memory-consolidation-weekly

The PR's own scope included analysing the *other* append-log-shaped routine, `memory-consolidation-weekly`, for the same defect class. It was **declined as correctly not-broken**, not silently skipped: [[memory-consolidation]]'s Step 5 writes one **unconditional**, date-keyed report file (`report-{date}.md`) per run — never an append to a shared multi-run log. Because each run gets its own freshly-named artifact, file-absence *is* an adequate failure signal for this routine; there is no stale-prior-run marker for a fresh file's `exists` check to be confused by. The two routines share superficial append-log DNA (both are weekly, both gate on a file under the dashboard config) but not the actual failure geometry PR #240 and PR #227 both close — `last-marker` would be a no-op change here, not a fix.

## Wiki pages updated

- [[loop-retro-promotion]] — dormant-stop `run=ok` write, `delivery=started` fail-safe marker (written pre-Mining, covers the whole met-path), `run=ok` post-merge write, and the predicate-kind switch from `exists` to `last-marker`.
- [[memory-consolidation]] — records the decline: analysed for the same defect class, correctly left on `exists` because it writes one unconditional per-run artifact rather than an append-only shared log.
- [[routines]] — `RoutineDef` field contract's predicate-kind enumeration corrected from three kinds to four (see below); `loop-retro-promotion-weekly`'s own routine-table row updated from "gates on an `exists` predicate" to the last-marker shape.
- [[dashboard-runner]] — `checkArtifact()`'s predicate-kind list corrected from three to four, `last-marker` described.

## Caveats / gotchas

**This PR also closed a pre-existing wiki staleness, found in pre-flight, unrelated to PR #240's own diff but adjacent to it.** [[dashboard-runner]] and [[routines]] both described the artifact-gate predicate evaluator as supporting exactly three kinds (`exists` / `contains` / `json-field`). `last-marker` was actually added a day earlier by a **different** PR — #227 (`26a5d69`, merged 2026-07-17T19:30:51Z, "Fix docs-sync same-date false-green: order-aware last-marker gate predicate"), which introduced the predicate kind and `docs-sync`'s own use of it — but no wiki source page or page edit ever recorded that addition. `last-marker` was therefore completely undocumented in the vault (zero hits, checked directly) until this ingest, despite already being live in `artifactGate.ts` and already used by two routines' worth of production config. This ingest's page updates cover both: `last-marker`'s existence and mechanics (attributable to PR #227) and its second application to `loop-retro-promotion-weekly` (PR #240 itself). No dedicated PR #227 source page was created — out of scope for this ingest, which covers PR #240 — its mechanics are described inline on the design pages instead.

**`last-marker` semantics, precisely** (verified directly against `skills/dashboard/runner/src/artifactGate.ts`): scans the file's lines for anything matching the marker set (`success` ∪ `failures`, substring match via `.includes`), and the **last matching line wins** — not the literal last line of the file, since a routine may append a non-terminal trailing note (e.g. `note=...`) after its real terminal marker within the same run. Passes iff that last terminal marker is the success marker; a file with no terminal marker at all reads NOT passed with an explicit "no terminal marker" reason, distinct from a plain "does not exist" failure.

**Not a contradiction with the pre-existing `⚠️ Corrected` note on [[loop-retro-promotion]] about a "gate can never pass" claim.** That earlier note (dated 2026-07-17, from [[pr_201_202_203_routine-followups]]) is about `maxAgeSeconds` staleness against a stale example-config path — an orthogonal axis to the predicate-*kind* switch this PR makes. Both are simultaneously true: the routine's `artifactPath`/`maxAgeSeconds` freshness question was resolved 2026-07-17 (live config, correct path, currently fresh), and separately, the predicate *shape* used to evaluate that fresh file's content was `exists` until this PR and is `last-marker` as of this PR.
