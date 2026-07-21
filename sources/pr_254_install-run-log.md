---
title: "PR #254 — install.sh verification run log"
type: source
created: 2026-07-21
last_updated: 2026-07-21
sources: []
tags: [install, docs, tier-gate, tier-0, judge-predicate, verification]
---

# PR #254 — install.sh verification run log

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR | [#254](https://github.com/blueman82/coderails/pull/254) |
| Title | docs: add install.sh verification run log |
| Branch | `docs/install-run-log` |
| Merged | 2026-07-21T13:46:01Z |
| Head at merge | `ddb8e47` |
| Tier | 1 (refiled from 0) |
| Files | `docs/INSTALL-RUN-LOG.md` (77 lines, 1 file) |

## What it adds

`docs/INSTALL-RUN-LOG.md` — a run log documenting a full end-to-end `install.sh` execution on
macOS. This closes one of the manual checks that had been outstanding since the public-release
work: the install path had been reasoned about but never recorded as actually run root-to-tip.

Split out of #252 specifically to fit under the tier-gate's tier-0 size cap (80 lines / 3 files,
enforced by `tg_prefilter` in `scripts/tier-gate/tier-gate-runner.sh`). At 77 lines / 1 file it
fits.

## The tier lesson — size is not sufficient for tier 0

The PR was **initially filed as tier 0 on size alone, and the tier-gate judge rejected it**
(`verdict=illegitimate`). The author's own correction, recorded in the PR body, is the durable
lesson:

> tier 0 also requires an EXISTING test/verify-criterion already covering the goal state
> (judge-prompt.md predicate 3), which a brand-new doc file structurally cannot satisfy no matter
> how small.

A new file has, by definition, no pre-existing coverage. So a brand-new doc can never be tier 0 —
not because it is risky, but because the third tier-0 predicate is structurally unsatisfiable for
it. Refiled as tier 1 and judged legitimate.

This matters because the size cap is the *visible* tier-0 constraint and the easy one to optimise
for. Passing it says nothing about the coverage predicate. A sibling PR, #255, was filed at tier 1
for the same reason.

> Contrast with [[pr_256_runner-transcript-persistence]], denied at the file-list stage with
> `verdict=self_edit` before any judging ran. #254 reached the judge and was rejected *on the
> predicate*; #256 never reached it at all. Two distinct refusal mechanisms.

See [[install-and-cache-trap]], [[pr_232_tier-review-gate]].
