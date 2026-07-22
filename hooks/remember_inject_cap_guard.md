---
title: "remember_inject_cap_guard.sh"
type: hook
created: 2026-07-22
last_updated: 2026-07-22
sources: [sources/pr_259_remember-inject-cap-guard.md]
tags: [hook, SessionStart, remember, plugin-boundary, warn-only, opt-in, token-burn, fail-open]
---

# remember_inject_cap_guard.sh

`SessionStart` hook that reports — and, only on explicit opt-in, re-applies —
the memory-injection byte cap on the **remember** plugin's
`session-start-hook.sh`. The only coderails hook that writes outside this repo,
and it does so only when asked. (verified — PR #259)

## Event and mode

| Field | Value |
|---|---|
| Event | `SessionStart` |
| Mode | silent when nothing to do; **notify** (never block) otherwise; writes only under opt-in |
| Timeout | (default); `async: false` |
| Position | second in the `SessionStart` array, after [[inject_bootstrap]] |
| Exit code | **always 0** — fail-open throughout, never blocks session start |

## What the cap is

Unpatched, the remember plugin concatenates `identity.md`, `core-memories.md`,
`today-*.md`, `now.md`, `recent.md` and `archive.md` into every session's context
verbatim with no size limit. The patch truncates each to
`REMEMBER_INJECT_MAX_BYTES` (default 8000) and prints a truncation marker naming
the full file's path. It was **row 5 of the 7-measure 2026-07-17 token-burn
effort** — the only one never committed anywhere. See
[[pr_259_remember-inject-cap-guard]] for that history and
[[pr_228_229_230_token-burn-reduction-and-agents-split]] for its six siblings.

## Logic summary

Skip gates in order, cheapest first (verified — reading
`hooks/scripts/remember_inject_cap_guard.sh`):

1. **Resolve the target.** `REMEMBER_HOOK_FILE` overrides everything (test
   seam). Otherwise read `~/.claude/plugins/installed_plugins.json` via `jq`,
   take the `remember@…` entries, prefer scope `user`, else highest version by
   `sort -V`. No manifest or no `jq` → glob `cache/*/remember/*/` and take the
   highest by `sort -V` (version-aware: `0.10.0` outranks `0.8.3`; plain
   lexicographic sort gets that backwards).
2. **Plugin not installed at all → exit 0 silently.** A machine without remember
   must never be nagged.
3. **Target missing or unreadable → notify, never write.** The plugin's layout
   changed; say so rather than doing nothing silently.
4. **Cap already present → exit 0 silently.** Keyed on `CAP_EVIDENCE`, see below.
5. **`REMEMBER_INJECT_CAP_AUTOWRITE != "1"` (the default) → notify once per
   plugin version, change nothing, exit 0.** This gate sits *before* every
   write-path check, which is what makes the suppression sound — see below.
6. Opt-in path only: patch text readable → block shape check → backup → awk
   rewrite to a temp file → sanity gate → mode preserve → `mv`.

### Detection keys on evidence, not on the token

```
CAP_EVIDENCE='head -c "$REMEMBER_INJECT_MAX_BYTES"'
```

A bare substring search for `REMEMBER_INJECT_MAX_BYTES` is satisfied by *any*
mention — a comment, a changelog line, the residue of a half-applied edit — so
the guard would conclude "capped" and exit silently while the truncation code was
absent, **swallowing the exact condition it exists to detect**. That was the
shipped behaviour until review caught it (PR #259 review finding 1). The
truncation call is the one line that cannot be present unless the patch actually
applied. The same constant re-gates the rewritten output before it replaces
anything. Generalisable: **detect on evidence of the effect, not on a token that
names it.** See [[enforcement-model]].

### The block match is a line sequence, not a line count

The shape check must find the vendor block **exactly once**: 0 means the vendor
rewrote it, >1 is ambiguous, only 1 is safe to replace. It is implemented as an
awk contiguous-line-sequence match, *not* a count of marker lines — the block's
last line is a bare `fi`, which appears 8 times in the real
`session-start-hook.sh`, so a per-line count refuses to patch the very file the
hook exists for. That bug shipped once and was hidden by small fixtures carrying
a single `fi`; the test now asserts multi-`fi` as a fixture precondition.
(verified — PR #259)

Whole-block literal match is deliberate over line-number or context-diff
tooling: diff tooling drifts *silently* across a vendor bump, a literal block
match fails **cleanly and loudly**.

## Block condition

None. This hook never blocks and never exits non-zero.

## Notice channels

One JSON document carrying **both** channels — two concatenated top-level objects
would not parse (the same note appears in `loop_stall_guard.sh`):

- `systemMessage` — the human-visible channel ([[loop_stall_guard]]'s idiom).
- `hookSpecificOutput.additionalContext` — the model-visible `SessionStart`
  channel ([[inject_bootstrap]]'s idiom).

Both matter: the human needs to know a plugin update wiped a hand patch, and the
agent needs to know its memory injection was just re-capped. Falls back to a
plain stderr line if `jq` is unavailable, so a notice is never lost to a missing
JSON encoder.

## Warn suppression, and what is never suppressed

The opt-in notice is stamped once per plugin version at
`$HOME/.claude/coderails/remember_inject_cap_warned` (override with
`REMEMBER_INJECT_STATE_DIR`). A plugin bump changes the stamp's contents and
warns afresh.

Because the opt-in gate sits before every write-path check, the opt-in notice is
the **only** message reachable in warn-only mode. Every other notice —
unrecognised shape, failed backup, unverified rewrite, failed swap — names a real
fault on a machine that explicitly asked to be written to, and those fire every
session, unsuppressed. That is a placement property, not a separate rule.

The stamp is deliberately **not** under `~/.claude/plugins/`: that tree belongs
to Claude Code's plugin installer, and writing coderails' own bookkeeping there
would be the same boundary violation the warn-only default exists to prevent. It
is also wiped by a plugin reinstall, which would silently reset suppression. See
[[plugin-boundary-writes]].

The stamp write is best-effort and wrapped as one `{ …; } 2>/dev/null` group,
not per-command redirects: a failing *output redirection* is diagnosed by the
shell before the command runs, so `printf … > "$f" 2>/dev/null` still prints to
the real stderr — and stray shell diagnostics interleaved with this hook's JSON
stdout break Claude Code's parse, turning a best-effort stamp into a lost notice.

## Write safety (opt-in path only)

- One **rolling** timestamped backup `<target>.coderails-bak-<ts>`; earlier ones
  are reaped after each successful backup, so a repeatedly-failing guard cannot
  litter the plugin cache. The name stays timestamped because the notices quote
  the path.
- A failed `cp` may leave a truncated partial; it is `rm -f`'d.
- Rewrite goes to `<target>.coderails-tmp.$$`, is sanity-gated (non-empty **and**
  carries `CAP_EVIDENCE`), gets the original mode restored (BSD `stat -f %Lp`,
  then GNU `stat -c %a`, then `chmod +x` — `chmod --reference` is GNU-only and
  absent on macOS), then `mv`-swapped. The target is never left half-patched.
- The success notice reports `${REMEMBER_INJECT_MAX_BYTES:-8000}`, not a
  hardcoded 8000 — an environment override would make the literal false.

## Environment variables

| Variable | Meaning |
|---|---|
| `REMEMBER_INJECT_CAP_AUTOWRITE` | `"1"` permits writing. Anything else, unset included, is warn-only. **Default OFF.** |
| `REMEMBER_INJECT_MAX_BYTES` | The cap itself, read by the *patched plugin file*, default 8000; also quoted in the success notice |
| `REMEMBER_HOOK_FILE` | Test seam — target file, bypassing all version resolution |
| `REMEMBER_PATCH_DIR` | Test seam — directory holding the canonical patch text |
| `REMEMBER_INJECT_STATE_DIR` | Test seam — where the once-per-version stamp lives |
| `CLAUDE_PLUGINS_DIR` | Test seam — plugin root, default `$HOME/.claude/plugins` |
| `REMEMBER_PLUGIN_VERSION` | Version string used in the notice |

No `$CLAUDE_DISCIPLINE_LOG` entry — this is not a discipline hook.

## Known caveats

- **Not enabled on this machine.** `REMEMBER_INJECT_CAP_AUTOWRITE` was never
  added to `~/.claude/settings.json` (a harness gate blocks agent edits to that
  file — see [[no_edit_on_main]]), so writing is OFF locally and the cap will not
  auto-reapply after a remember bump until it is added by hand.
- **The reported version is best-effort.** Neither resolution path can prove
  which install Claude Code is running. A wrong guess patches an inactive copy —
  inert, but the notice names a version the session is not using.
- **Editing `patched.txt`'s NOTE comment does not propagate** to an
  already-patched install, because detection keys on the truncation call.

## See also

[[plugin-boundary-writes]] — the design decision behind warn-only  
[[pr_259_remember-inject-cap-guard]] — the source record, the row-5 history, the review findings  
[[install-and-cache-trap]] — the version-pinned-cache mechanism that makes the patch perishable  
[[inject_bootstrap]] — the sibling `SessionStart` hook, and the `additionalContext` idiom  
[[enforcement-model]] — hooks vs commands; detection-evidence framing  
[[test_gate]] — the glob-discovery gate that a self-skipping test defeated  
`coderails/hooks/scripts/remember_inject_cap_guard.sh`  
`coderails/hooks/patches/README.md`
