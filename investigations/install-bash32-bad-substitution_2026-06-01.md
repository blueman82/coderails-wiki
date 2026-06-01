---
title: "Investigation: install.sh bash 3.2 'bad substitution'"
type: investigation
created: 2026-06-01
last_updated: 2026-06-01
sources:
  - install.sh
tags: [install, bash, portability, bug, macos, fixed]
---

# Investigation: install.sh `bad substitution` on bash 3.2

## Symptom

On a second machine, `install.sh` printed at the command-conflict overwrite prompt:

```
install.sh: line 299: ${answer,,}: bad substitution
```

The prompt rendered, the user pressed `y`, then the script errored instead of
overwriting. (verified: error text pasted from the affected machine this session)

## Root cause

Line 299 used `${answer,,}` — bash's **case-modification expansion**, which lowercases
a variable. That syntax is **bash 4.0+ only**. macOS ships **bash 3.2.57** as
`/bin/bash` (frozen at the last GPLv2 release), and the script's shebang is
`#!/usr/bin/env bash`, so on a machine where PATH resolves `bash` to the system one,
the 4.x syntax is parsed by a 3.2 interpreter → "bad substitution". (verified:
`grep` confirmed line 299 was the sole bash-4-ism in the file; reproduced the
version split — this machine's PATH bash is 5.3, the affected machine's is 3.2)

Why it never showed up before: the original author's machine (and the primary dev
machine) resolve `bash` to a Homebrew bash 4+/5+, where `,,` works. The bug only
surfaces where `/bin/bash` 3.2 is first on PATH.

## Fix

Replace the bash-4 expansion with a portable `tr` lowercase, which works on 3.2+:

```bash
answer=$(printf '%s' "$answer" | tr '[:upper:]' '[:lower:]')
if [[ "$answer" == "y" ]]; then
```

Committed to `blueman82/coderails` `main` (`a312b28`). A comment was added at the
call site naming the bash-3.2 reason so it isn't "tidied" back to `,,`. (verified:
committed + pushed this session; `bash -n install.sh` parses, Y→overwrite / n→skip
behaviour confirmed)

## Scope check

`${answer,,}` was the **only** bash-4 case-modification expansion in `install.sh`,
and there were **none** in `uninstall.sh`. So this one change makes the installer
3.2-safe. No other `,,`/`^^`/`,`/`^` expansions remain. (verified: `grep -nE` scan
of both scripts this session)

## Lesson

Shebang `#!/usr/bin/env bash` does **not** guarantee a modern bash — it picks up
whatever is first on PATH, which on stock macOS is 3.2. Any installer meant to run
on an unprepared Mac must stay within bash 3.2 syntax (no `,,`/`^^`, no associative
arrays, no `mapfile`/`readarray`) or explicitly re-exec under a known-good bash.

## Cross-References

- [[install-and-cache-trap]] — the other install.sh gotcha (cache, not syntax)
- [[repo-hosting]] — where install.sh lives and how the other machine pulls the fix
