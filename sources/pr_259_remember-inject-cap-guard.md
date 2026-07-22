---
title: "PR #259 — remember-plugin memory-injection cap guard; the lost 7th token-burn measure"
type: source
created: 2026-07-22
last_updated: 2026-07-22
sources:
  - hooks/scripts/remember_inject_cap_guard.sh
  - hooks/patches/README.md
  - hooks/patches/remember_inject_cap.vendor.txt
  - hooks/patches/remember_inject_cap.patched.txt
  - hooks/scripts/tests/remember_inject_cap_guard.test.sh
  - hooks/hooks.json
  - AGENTS.md
  - README.md
  - docs/REFERENCE.md
tags: [hook, SessionStart, remember, plugin-boundary, token-burn, warn-only, opt-in, patch-provenance, test-skip, detection-evidence]
---

# PR #259 — remember-plugin memory-injection cap guard

## PR metadata

| Field | Value |
|---|---|
| PR number | #259 |
| Branch | `feat/remember-inject-cap-guard` |
| Merged | 2026-07-22 (15:35:02Z) |
| Merge SHA | `3de997b` (branch HEAD `c660b1d`) |
| JIRA ticket | — |

10 files, +1146/-0 — no deletions, nothing pre-existing modified beyond three
doc tables and a `hooks.json` array append. (verified: `git show 3de997b --stat`)

## Summary

Puts a hand-applied, never-committed patch into version control and gives it a
`SessionStart` hook that can detect its absence — and, only on explicit opt-in,
re-apply it. The patch caps the **remember** plugin's session-start memory
injection at `REMEMBER_INJECT_MAX_BYTES` (default 8000) per file. See
[[remember_inject_cap_guard]] for the hook's own contract and
[[plugin-boundary-writes]] for the design decision that made it warn-only.

## The history: row 5 of 7, and how it was nearly lost

The 2026-07-17 "token-burn reduction" effort shipped **seven** measures, numbered
rows 1–7 in the planning session. Six of them went into git as PRs:

| Row | Where it landed |
|---|---|
| 1–4 | PR #228 (`feat/tokenA-skill`) — see [[pr_228_229_230_token-burn-reduction-and-agents-split]] |
| 5 | **nowhere** — applied by hand to a file inside the remember plugin's version-pinned cache directory |
| 6 | PR #229 (`feat/tokenB-agents`) |
| 7 | PR #230 (`feat/tokenC-loopcost`) |

Row 5's target — `~/.claude/plugins/cache/<marketplace>/remember/<version>/scripts/session-start-hook.sh`
— is not in any repo. Unpatched, that script concatenates `identity.md`,
`core-memories.md`, `today-*.md`, `now.md`, `recent.md` and `archive.md` into
every session's context verbatim, with no size limit. The patch truncates each
to `REMEMBER_INJECT_MAX_BYTES`. (verified: `hooks/patches/remember_inject_cap.vendor.txt`
vs `remember_inject_cap.patched.txt`)

On 2026-07-22 row 5 was nearly lost outright. Reconstructing *what it even was*
required searching every Claude Code transcript on disk: it appeared in no PR,
no issue, no commit message and no repo file. The surviving prose in
`skills/agentic-loop/SKILL.md` had been renumbered to **"row N of 4"**, erasing
the record that there were ever seven measures. It was finally identified from a
planning-session transcript dated 2026-07-17. (verified: stated in the PR's own
premise and consistent with the guard script's `WHY THIS EXISTS` header, which
records "shipped seven measures. Six went in via PRs.")

> ⚠️ CONTRADICTION: [[pr_228_229_230_token-burn-reduction-and-agents-split]] and
> the `skills/agentic-loop/SKILL.md` text it quotes describe the token-burn rules
> as "row N of **4**". That numbering is the erasure, not the original scheme.
> PR #228 genuinely only shipped four rules — the "of 4" denominator is what is
> wrong. The effort was rows 1–7; #228 carried rows 1–4, row 5 was the
> uncommitted ops leg recovered by this PR, #229 was row 6, #230 was row 7.
> Read that page's rule table as "rows 1–4 of 7".

**The durable lesson:** a measure applied outside version control has no record
and cannot be audited. It is not merely fragile against a plugin bump — it is
invisible to every review, every diff, every doc-drift check, and to the
renumbering that quietly rewrote its siblings. Recorded on
[[plugin-boundary-writes]].

## Files changed

- **`hooks/scripts/remember_inject_cap_guard.sh`** (338 lines, new) — the
  `SessionStart` hook. Full treatment on [[remember_inject_cap_guard]].
- **`hooks/patches/`** (new directory) — the canonical patch text, now in git:
  `remember_inject_cap.vendor.txt` (the search block, the vendor's unpatched
  `MEMORY` block) and `remember_inject_cap.patched.txt` (the replace block).
  Both are **byte-exact match keys, not documentation** — the guard does a
  whole-block literal line-sequence search/replace, so one added line or changed
  indent stops the match. `README.md` in that directory exists precisely so no
  provenance banner is ever written *inside* the `.txt` files. (verified:
  `hooks/patches/README.md`)
- **`hooks/hooks.json`** — appends the guard to the `SessionStart` array, after
  [[inject_bootstrap]]. `async: false`.
- **`hooks/scripts/tests/remember_inject_cap_guard.test.sh`** (695 lines, new).
- **`hooks/scripts/tests/exec_bit_invariant.test.sh`** (+2) — registers the new
  script in the exec-bit invariant.
- **`AGENTS.md` / `README.md` / `docs/REFERENCE.md`** (+1 line each) — one row
  added to each file's hook table.

### `vendor.txt` is reconstructed, not authentic

The unpatched original was never kept — the cap was applied by hand directly to
the live cache file. `vendor.txt` was recovered by **reverse-applying** the hand
patch to the live file; `patched.txt` was taken from the live file as it stood
after it. Both were then validated by round trip (reverse → re-apply reproduces
the live file byte-for-byte), and the test suite asserts that inverse
relationship hermetically on every run. (verified: `hooks/patches/README.md`
"Provenance") This is honest provenance labelling of a derived artifact — worth
copying whenever a canonical text is recovered rather than captured.

## The design decision: warn-only by default

Built initially to **auto-write by default**. Four review agents audited it; one
demonstrated live that auto-write was not defensible for a public plugin. The
resolution and its reasoning live on [[plugin-boundary-writes]]. In short:
writing now requires `REMEMBER_INJECT_CAP_AUTOWRITE=1`, and the warn is
suppressed to once per plugin version via a stamp under
`$HOME/.claude/coderails/` — deliberately **not** under `~/.claude/plugins/`.

## Review findings worth keeping as engineering lessons

### 1. Detection was a bare substring grep — the check swallowed the condition

The guard originally decided "is the cap present?" by grepping for the token
`REMEMBER_INJECT_MAX_BYTES` anywhere in the target. Any mention satisfies that —
a comment, a changelog line, the residue of a half-applied edit — so the hook
would exit silently concluding the cap was present while the truncation code was
absent. **That is the exact condition the hook exists to detect, swallowed by the
check meant to detect it.**

Sharpening the point: the PR's own test used a *stronger* definition than
production did — it grepped the truncation line, while the shipped code grepped
the token. A test more discriminating than the implementation it guards is a
standing tell.

Fixed to key on **evidence of the patch**: `head -c "$REMEMBER_INJECT_MAX_BYTES"`,
the one line that cannot be present unless the patch actually applied. (verified:
`CAP_EVIDENCE` in `remember_inject_cap_guard.sh`, used at both the
already-capped early exit and the post-rewrite sanity gate.) Generalised on
[[remember_inject_cap_guard]] and [[enforcement-model]]: **detect on evidence of
the effect, not on a token that merely names it.**

### 2. A regression test that skipped itself into uselessness

The test guarding a known historical bug was wrapped in a check for the live
plugin cache. Absent that cache, it printed `skip` — while the suite still
reported **PASS**. Because this repo's test gate auto-discovers `*.test.sh` by
glob, the gate reported green against the exact bug the test was written to
prevent, **on any fresh clone**.

Proven by mutation, not by argument: reintroduce the bug → 3 failures on a
normal run; point `HOME` at an empty dir → `skip` and PASS. Fixed by making the
coverage hermetic: `make_fixture` builds a fixture with multiple bare `fi` lines
and asserts the property on every machine, and a new hermetic inverse test (j2)
proves `vendor.txt`/`patched.txt` are exact inverses with no live cache
involved. The live-cache round trip survives as an explicitly-labelled **bonus**
check, correctly gated on the deployed copy still being current. (verified:
tests (b) and (j2) in `remember_inject_cap_guard.test.sh`)

The lesson is about the *gate*, not the test: an auto-discovering glob-based test
gate treats a self-skipping test as a passing one. A conditional skip in a
regression test converts a red into a green on precisely the machines least
likely to have the bug already fixed. See [[test_gate]].

### 3. The historical bug those tests guard

The shape check originally counted occurrences of the patch block's **last
line** — which is a bare `fi`. That line appears 8 times in the real
`session-start-hook.sh`, so the count came back >1, "ambiguous", and the guard
**refused to patch the very file it exists for**. Small hand-written test
fixtures containing one `fi` hid it completely.

Fixed by matching the whole block as a contiguous literal line *sequence* via
awk and counting sequence occurrences, which must be exactly 1. (verified:
`n_block` awk in the guard; the design note is in the script's `HOW THE PATCH IS
APPLIED` header.) Lesson: a fixture simpler than production can make a shape
check look correct while it is structurally wrong — the discriminating property
(here, "many bare `fi` lines") has to be an asserted precondition of the fixture.

## Wiki pages updated

- [[remember_inject_cap_guard]] — new hook page.
- [[plugin-boundary-writes]] — new design page (one plugin writing another
  plugin's files; why warn-only won; the outside-version-control lesson).
- [[install-and-cache-trap]] — cross-linked; this PR is the first coderails
  change that acts on *another* plugin's cache trap rather than its own.
- [[pr_228_229_230_token-burn-reduction-and-agents-split]] — contradiction
  notice added ("row N of 4" → rows 1–4 of 7).
- [[index]] hook table, [[log]].

## Caveats / gotchas

- **The opt-in is NOT enabled on this machine.** `REMEMBER_INJECT_CAP_AUTOWRITE`
  could not be written to `~/.claude/settings.json` — a harness gate blocks agent
  edits to that file (the same `no_edit_on_main` rule that blocks it on *any*
  branch; see [[no_edit_on_main]]). Writing is therefore currently **OFF**
  locally and the cap will **not** auto-reapply after a remember bump until Gary
  adds the env var by hand. (verified: stated in the ingest brief; not
  independently re-checked against the live `settings.json` by this ingest.)
- **Version resolution is best-effort and cannot be proven.** Neither the
  manifest read nor the glob fallback can establish which install Claude Code is
  actually running — the manifest lists one record per *scope*, so a
  project-scoped and a user-scoped install can both be on disk at different
  versions. The hook prefers scope `user`, else the highest by `sort -V`. If it
  guesses wrong it patches an inactive copy (inert), but the notice then names a
  version the running session is not using. Treat the reported version as
  best-effort. (verified: the script's `HOW THE TARGET IS RESOLVED` header)
- **Editing `patched.txt`'s NOTE comment does not propagate to an
  already-patched install.** Detection keys on the truncation call, not the
  comment, so a comment-only change leaves a capped file untouched until the next
  plugin bump wipes and re-applies the whole block. (verified:
  `hooks/patches/README.md`)
- **`vendor.txt`/`patched.txt` must never gain a header, banner or comment.**
  They are match keys. Legitimate upstream change means re-deriving both from the
  new vendor source, not hand-editing them.
