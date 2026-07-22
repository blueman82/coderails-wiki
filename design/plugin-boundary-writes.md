---
title: "Plugin-Boundary Writes"
type: design
created: 2026-07-22
last_updated: 2026-07-22
sources: [sources/pr_259_remember-inject-cap-guard.md]
tags: [design, plugin-boundary, warn-only, opt-in, consent, version-control, token-burn, provenance]
---

# Plugin-Boundary Writes

When one plugin modifies a file belonging to another plugin, the modification is
not a bug fix — it is an unrequested change to someone else's package on a
stranger's machine. coderails' rule: **warn by default, write only on explicit
opt-in, and never write bookkeeping into the other party's tree.** (verified —
PR #259, [[pr_259_remember-inject-cap-guard]])

## Context

The 2026-07-17 token-burn effort's row 5 capped the **remember** plugin's
session-start memory injection. Its target lives in the version-pinned plugin
cache, not in any repo, so a plugin bump installs a fresh unpatched copy and the
cap silently disappears — the [[install-and-cache-trap]] mechanism, but applied
to *another maintainer's* plugin rather than coderails' own. Making the cap
survive a bump means something has to notice its absence and act. What "act"
means is the whole decision.

## The rule

1. **Default is warn-only.** Report what is missing, name the opt-in, change
   nothing.
2. **Writing requires an explicit env-var opt-in** —
   `REMEMBER_INJECT_CAP_AUTOWRITE=1` in a `settings.json` `env` block. Not a
   config default, not a first-run prompt: an affirmative act by the machine's
   owner.
3. **A warning that cannot be silenced is a defect, not a feature.** Suppress to
   once per relevant version, with a fresh warn when that version changes.
4. **Suppression state goes in your own tree, never theirs.** coderails' stamp
   lives under `$HOME/.claude/coderails/`, never under `~/.claude/plugins/`.
5. **Genuine faults are never suppressed.** Only the opt-in notice is
   rate-limited; unrecognised shape, failed backup, unverified rewrite all fire
   every session, because those name a real problem on a machine that *asked* to
   be written to.
6. **The canonical text lives in git**, with honest provenance labelling if it
   was reconstructed rather than captured.

## Rationale

The guard was built to auto-write by default. Four review agents audited it and
one demonstrated **live** that auto-write was not defensible for a public
plugin. Two arguments, both concrete:

- **The consent argument.** coderails would silently rewrite a file belonging to
  `remember`, an official-marketplace plugin, on any user's machine. The 8000-byte
  cap is the repo owner's **personal tuning constant**, not a bug fix. A user who
  never asked for it would find their plugin quietly rewritten. "It's an
  improvement" is the maintainer's judgment, not the user's consent.
- **The unsilenceable-nag argument, which was worse.** On any plugin version
  whose file shape differs from the expected block, the guard warns but never
  writes its sentinel — so the *identical* warning fires every session forever,
  with no suppression and no opt-out. An auto-write design that degrades into a
  permanent nag on an unrecognised version is a bad default in both directions at
  once.

Rule 4 is the same principle applied one level down. Writing a suppression stamp
into `~/.claude/plugins/` would be exactly the boundary violation warn-only
exists to prevent — and it is wiped by a plugin reinstall, which would silently
reset suppression as a bonus defect.

## Where it is enforced

Mechanically, in [[remember_inject_cap_guard]] itself: the
`REMEMBER_INJECT_CAP_AUTOWRITE` gate sits *before* every write-path check, so in
warn-only mode the script never reaches the shape check, the backup, or the
rewrite. That placement is what makes rule 5 true as a property rather than as a
convention — the opt-in notice is the only message reachable in the default mode.

Nothing enforces this rule across *future* hooks. It is a design convention with
one implementation. (inferred — no cross-cutting check exists; grepping for a
plugin-boundary guard would find only this hook.)

## The companion lesson: outside version control is outside the record

Row 5 was applied by hand and committed nowhere. On 2026-07-22, reconstructing
what it even was required searching every Claude Code transcript on disk — it
appeared in no PR, no issue, no commit message and no repo file. Worse, the
surviving prose in `skills/agentic-loop/SKILL.md` had been **renumbered to "row
N of 4"**, erasing the record that there were ever seven measures. It was
recovered only from a planning-session transcript.

**A measure applied outside version control has no record and cannot be
audited.** The perishability against a plugin bump is the obvious cost. The real
cost is that it is invisible to every review, every diff, every doc-drift check —
and to the renumbering that quietly rewrote its siblings' denominator without
anyone noticing a gap. If a change cannot live in the repo it modifies, its
canonical text must live in *some* repo, alongside the mechanism that re-applies
it. See [[pr_259_remember-inject-cap-guard]] and, for the sibling measures that
did land in git, [[pr_228_229_230_token-burn-reduction-and-agents-split]].

## Known caveats / edge cases

- **Reconstructed provenance.** The canonical `vendor.txt` is not an authentic
  upstream artifact — no untouched original was kept, so it was recovered by
  reverse-applying the hand patch to the live file. It is validated by a
  round-trip inverse test, and labelled as reconstructed in its own README rather
  than presented as pristine. Honest labelling of a derived canonical text is
  part of this rule, not an aside.
- **Opt-in is not enabled on this machine.** The env var could not be written to
  `~/.claude/settings.json` because a harness gate blocks agent edits to that
  file ([[no_edit_on_main]]). The design is live; the local activation is not.
- **The rule costs the thing it protects.** With writing off by default, the cap
  does *not* come back after a remember bump — the token burn returns until
  someone acts. That is the accepted price of not writing into another
  maintainer's package unasked.

## See also

[[remember_inject_cap_guard]] — the sole implementation  
[[pr_259_remember-inject-cap-guard]] — the source record and review findings  
[[install-and-cache-trap]] — the version-pinned-cache mechanism, applied here to a *foreign* plugin  
[[enforcement-model]] — hooks enforce, commands advise; what a warn-only hook can and cannot guarantee  
[[trust-floor]] — the adjacent question of what coderails may do without asking
