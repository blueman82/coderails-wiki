---
title: "PR #299 — .hud-rail min-width: 0, and the disproved premise it replaced"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - skills/dashboard/app/src/styles/hud.css
tags: [source, dashboard, css, responsive, grid-min-content, disproved-premise, measurement, cascade]
---

# PR #299 — .hud-rail min-width: 0, and the disproved premise it replaced

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #299 |
| Branch | `fix/hud-rail-min-width` |
| Merged | 2026-07-24 (`2026-07-24T08:34:08Z`) |
| Merge commit | `b4b078298b732ee5b67253ec1912d423faa9d2ac` |
| Head commit | `12c7244ab570d2d8b862d374cdf36a76f06e77f0` |
| JIRA ticket | — |

## Summary

One declaration: `min-width: 0` on `.hud-rail` (`hud.css`, now line 218), plus
a comment. One file, nothing else.

Below the `@media (max-width: 1100px)` breakpoint `.hud-stage` collapses to
`grid-template-columns: 1fr`, so `.hud-rail` becomes the grid item that has to
track the viewport. **A grid item's automatic minimum size is its min-content
width**, so with the default `min-width: auto` the column could not shrink past
the widest content in the rail — measured at **404.156px**. The column froze
there and the rail's content ran past the viewport edge on anything narrower.

`.hud-rail` already carried `min-height: 0` at the line above, with a comment
explaining why it's needed ("what lets a flex child shrink below its content
size"). **The identical reasoning on the width axis had simply never been
applied.** The fix is symmetric with a rule the file already justifies —
which is why it reads as one line rather than a design change.

Above 1100px nothing changes: those tracks are fixed-width (`330px 1fr 300px`),
so the rail's minimum never binds.

## The durable lesson: the reported cause was disproved by measurement

This is the compounding part, not the CSS.

The obvious-looking cause — and the one an earlier PR was built on — was
`.hud-trend-value`'s `white-space: nowrap`, since that value is the longest
string in the row. **Measured at a 320px viewport, changing it does nothing:**

| variant | computed `white-space` | value past viewport | grid column |
|---|---|---|---|
| unfixed | `nowrap` | +61.2px | 404.156px |
| `.hud-trend-value{white-space:normal}` | `normal` | **+61.2px** | **404.156px** |
| `.hud-rail{min-width:0}` | `nowrap` | **−138.4px** | **288px** |
| both | `normal` | −138.4px | 288px |

The white-space change **applies** — the computed value really does flip to
`normal` — and moves the geometry by **exactly zero**, because the trend value
is not what pins the layout; it sits 15px inside the rail at every width. The
`min-width: 0` fix works alone, and adding white-space on top makes no further
difference. So no white-space change is included.

**A change that visibly takes effect can still have zero effect on the thing
it was supposed to fix.** "The computed style changed" is not evidence the
cause was found — only a measurement of the symptom is.

## The cascade defect that hid it (belongs to #296, not this PR)

Recorded here because it is a CSS-specific instance of a general failure mode
this vault tracks: **a change that is present in the diff and does nothing.**

The superseded PR **#296** (`fix/context-trend-panel-width`, **CLOSED unmerged**
2026-07-24T08:00:53Z) targeted `.hud-trend-value`. Beyond resting on the
disproved premise above, its approach **could not take effect as written**: the
base `.hud-trend-value` rule sits *after* the `@media` block at **equal
specificity** (0,1,0), so the media-query override lost on **source order** and
the computed value stayed `nowrap` at every width.

Equal specificity means later-in-source wins, and a media query grants no
precedence of its own — so an override placed before its base rule is silently
dead. It looks correct in review, and it is inert. #296 later added a
regression test pinning declaration **source order** with a negative control,
which is the right instrument for this class — but that branch closed unmerged,
so **that test is not on `origin/main`** (verified — only `hudRailScroll.test.ts`
exists in `skills/dashboard/app/test/`). #299 ships the stylesheet change with
no test of its own.

## Regression sweep

`.hud-rail` affects every rail panel, not just Context Trend, so the sweep
checked each rail descendant's right edge against the rail's own (the rail has
`overflow-x: hidden`, so that is what would clip). Clean at 1400 / 1000 / 800 /
600 / 480 / 400 / 360 / 320 / 280: zero viewport spill, zero clipping, zero
scroll overflow.

Below 320px the rows degrade gracefully rather than breaking — the label wraps
to a second line and the `flex: 1` leader collapses to 0, while the value stays
intact and fully inside the rail.

## Method, and its stated limitation

Measured by varying an **iframe's** width, not by resizing the OS window,
because `resize_window` is unreliable on this machine: it returned "Successfully
resized" for 1400x900, 1440x900 and 900x800 while `window.innerWidth` stayed
677 every time. **That false success is worth knowing independently** — it is
the same shape as [[pr_274_tier_gate_observability_fixes|#274]]'s `curl`
exiting 0 on a 401: a tool reporting success for an operation it did not
perform.

An iframe is a real viewport for the document inside it —
`matchMedia('(max-width: 1100px)')` flips correctly with iframe width, so the
responsive path genuinely runs and layout is real. It is **not** an
element-width probe (no element's width is set directly), and it is **not** a
native OS resize; the unreproduced link is viewport→stage, which carries no
logic touching `.hud-trend-*`. Stated rather than papered over.

Before/after were measured on **different URLs** in the same session — BEFORE
on the deployed LAN dashboard (`192.168.50.140:4173`, `origin/main`), AFTER on
`127.0.0.1:4201` (`next dev` on the branch) — with both serving identical
populated data.

## Wiki pages updated

- [[dashboard]] — the responsive-rail fix and the disproved-premise record

## Caveats / gotchas

- **No test shipped with the fix.** #296's source-order regression test was the
  instrument for this class of defect and it closed unmerged with that branch.
- The pre-existing limit below ~240px is untouched: the rail begins clipping,
  first at `.hud-suffix` via its own `white-space: nowrap` — a different
  element, present on `origin/main`, deliberately out of scope.
- The launchd-managed dashboard on 4173 was not restarted or disturbed during
  verification (HTTP 200 throughout).

## See also

- [[dashboard]] — the dashboard skill and its panels
- [[pr_287_dashboard-context-trend-panel]] — the Context Trend panel whose value string looked like the cause
- [[pr_181_183_dashboard-run-output-wrap-and-inbox-brief-clean-modal]] — the earlier "wrap, don't clip" house rule in this stylesheet
