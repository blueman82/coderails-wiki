---
title: "Session: verify-loop total enforcement"
type: source
origin: direct-edit session (no PR)
created: 2026-06-01
last_updated: 2026-06-01
tags: [source, session, hook, check_verify_loop, did-not-verify, enforcement]
---

# Session 2026-06-01 — verify-loop total enforcement

A direct-edit session (no PR in the plugin repo, which carries no git history) that
escalated `check_verify_loop.sh` from selective to **total** enforcement. Recorded
as a session source because the change was made directly to the working tree.

## The gap this closed

The 2026-05-31 hardening (see [[session_2026-05-31_verify-loop-hardening]]) made a
DNV bullet block **only** when it named a `file.ext` or `file:line` token, with a
`meta_pattern` allowlist to drop false positives. The consequence: a prose claim
that named no file — "the dedup regression test catches the bug", "IMS refresh-token
rotation still works" — passed unpoliced even though it was just as checkable. The
user observed exactly this: three prose bullets after a restart went unverified, and
none named a file token, so the old regex correctly let them through. The selective
rule was the wrong rule, not a bug in it.

## Changes to `check_verify_loop.sh`

1. **Removed the source-token regex and the `meta_pattern` allowlist.** Both encoded
   the file-naming requirement and its false-positive escape phrases. (verified)
2. **Total enforcement.** Now *any* `## Did Not Verify` bullet blocks unless tagged.
   Untagged bullets are found by inverting the tag match (`grep -ivE`), then counting
   bullets with non-whitespace content (`grep -cE '^- *[^[:space:]]'` — a bare `- `
   is not a claim). Any count `> 0` exits 2. (verified: check_verify_loop.sh:136–139,143)
3. **Single escape hatch.** A bullet passes only if its leading clause is an explicit
   `(unverifiable: <reason>)` tag, anchored by `^- *\(unverifiable:` so it can't be
   sprinkled mid-sentence. This replaced the fuzzy keyword allowlist. (verified:135)
4. **Loop-guard makes it safe.** `stop_hook_active` blocks once then allows the
   re-stop, so total enforcement can never deadlock. (verified:56–60)

## Honest boundary (recorded deliberately)

Mechanical enforcement can force resolve-or-tag. It **cannot** force the tag to be
honest — tagging a checkable item is cheaper than checking it, and the hook can't
tell. The guarantee is "nothing silently deferred," auditable via the greppable tag,
not "everything verified." This boundary is written into the script header
(check_verify_loop.sh:10–12) so a future contributor doesn't mistake the tag for a
proof of verification.

## Verification done this session

- Logic-replication test (7 cases) and a synthetic-transcript run against the real
  script: untagged → block, tagged → allow. (verified: run this session)
- A **live** block: an untagged prose bullet with no filename was caught by the hook
  after reinstall + restart, proving total enforcement works on prose. (verified)

## Install/cache note

The hook fires from the cached copy at
`~/.claude/plugins/cache/coderails/coderails/1.0.0/`, not the repo.
`/reload-plugins` does not re-copy script files; only `/plugin uninstall` + `install`
repopulates the cache, and a **restart** is additionally required for the reinstalled
hook to be active in-session. See [[install-and-cache-trap]].

## Downstream doc sync

The authoritative `instructions/self-checking-discipline.md` and the live
`~/.claude/CLAUDE.md` discipline section were both updated to the total-enforcement
DNV wording so the prompt-level rule matches the hook. (verified this session)

## Related

- [[check_verify_loop]]
- [[session_2026-05-31_verify-loop-hardening]]
- [[enforcement-model]]
- [[install-and-cache-trap]]
