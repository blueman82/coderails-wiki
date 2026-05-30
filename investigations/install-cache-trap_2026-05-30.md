---
title: "Investigation: Install Cache Trap (2026-05-30)"
type: investigation
created: 2026-05-30
last_updated: 2026-05-30
sources:
  - .claude-plugin/marketplace.json
  - hooks/scripts/check_verify_loop.sh
  - ~/.claude/settings.json
tags: [investigation, install, cache, plugin, hook, verify-loop]
---

# Investigation: Install Cache Trap (2026-05-30)

A session-level debugging trace. Preserved so a future session does not rediscover this the hard way.

## Finding summary

During a session editing coderails hooks, a Stop hook edit had no effect at runtime. Root cause was three divergent copies of the plugin plus a gate threshold that skipped the hook regardless.

## Three divergent copies

At the time of the session the plugin existed in three places simultaneously: (inferred: session knowledge — the three-copy state was the session's finding; current state below is verified)

| Copy | Path | Role during the session |
|---|---|---|
| Working repo | `/Users/harrison/Documents/Github/coderails` | Edits landed here |
| Stale download | `/Users/harrison/Downloads/coderails-test/coderails` | Stale — marketplace source pointed here |
| Runtime cache | Claude Code plugin cache | Loaded from the stale download, not the working repo |

The stale download copy still exists as of this writing (verified: `ls /Users/harrison/Downloads/coderails-test/coderails` — directory present). Its `marketplace.json` is a full copy of the plugin with `"source": "./"` — a relative path pointing within that stale tree, not the working repo. (verified: Downloads/coderails-test/coderails/.claude-plugin/marketplace.json:17)

## Why edits had no effect

Three compounding causes:

**a) Stale marketplace source.** Claude Code's `extraKnownMarketplaces.coderails.source.path` pointed at the Downloads copy, not Documents/Github/coderails. So `/reload-plugins` loaded the unedited stale tree.

**b) Runtime cache.** Even with a corrected source, the cached plugin version continued to run until a clean reinstall forced a reload from the corrected path.

**c) Gate threshold.** `check_verify_loop.sh` skipped when `file_count -lt 1` (verified: check_verify_loop.sh:48). During the session the threshold was `< 3`, meaning responses touching fewer than three files were never policed. A hook that never fires validates nothing.

## Resolution applied

1. **Repoint marketplace**: `extraKnownMarketplaces.coderails.source.path` now correctly points to `/Users/harrison/Documents/Github/coderails`. (verified: `~/.claude/settings.json` — `extraKnownMarketplaces.coderails.source.path` = `/Users/harrison/Documents/Github/coderails`)
2. **Clean reinstall**: ran `install.sh` against the correct path, forcing a full cache reload.
3. **Lower threshold**: `check_verify_loop.sh` now skips at `file_count -lt 1` — a single edited file is enough to bring the response in scope. (verified: check_verify_loop.sh:48)

## What to check if hooks stop working again

1. `cat ~/.claude/settings.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['extraKnownMarketplaces'])"` — confirm source.path is the Documents/Github copy.
2. Compare `hooks/scripts/check_verify_loop.sh` between the working repo and `Downloads/coderails-test/coderails/` — if they diverge, the stale copy was loaded.
3. Check `~/.claude/discipline.log` for recent `hook=verify_loop` entries. If the log is silent after file-editing turns, the hook is not running.

## Related

- [[install-and-cache-trap]]
- [[check_verify_loop]]
- [[discipline-loop]]
- [[enforcement-model]]
