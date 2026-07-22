---
title: "PRs #260, #263: dashboard security review — six Low findings, eight defences held, MERGED ≠ DEPLOYED"
type: source
created: 2026-07-22
last_updated: 2026-07-22
origin: PRs #260 (merge commit 900c740, merged 2026-07-22T18:28:08Z) and #263 (merge commit 7ff7967, merged 2026-07-22T18:37:25Z)
sources:
  - skills/dashboard/app/src/app/api/run/route.ts
  - skills/dashboard/runner/src/sweep.ts
  - skills/dashboard/app/src/lib/collect/queueActions.ts
  - skills/dashboard/app/src/lib/build/spawn.ts
  - skills/dashboard/app/src/lib/config.ts
  - skills/dashboard/app/src/lib/requestGuard.ts
  - docs/DASHBOARD-SECURITY-REVIEW.md (removed from tracking by PR #263, readable from history at b32686f)
tags: [source, dashboard, security-review, path-traversal, timing-safe-compare, permissions, threat-model, launchd, deployment-gap]
---

# PRs #260, #263 — dashboard security review

A security review of `skills/dashboard/` found six findings, all severity **Low**, all
same-user local-process, none deferred. PR #260 fixes all six. PR #263 removes the
review document itself from the repo. Both merged 2026-07-22, nine minutes apart.

## Threat-model finding that reframed the review

**Nothing crosses a privilege boundary.** `~/.claude` and
`~/.claude/coderails-dashboard/` are both `drwx------` — any process able to write
those files can already run `claude --dangerously-skip-permissions` directly. Routing
an attack through the dashboard's queue gains an attacker nothing beyond what they
already have. `(verified)` against the live filesystem — the reviewer checked this
rather than accepting the review brief's local-privilege-escalation framing, and
graded every finding **down** accordingly rather than inflating them. This is the same
posture [[enforcement-model]] documents for the tier-gate daemon: a boundary claim is
checked against the actual uid/permission state on the machine, not assumed from the
architecture diagram.

## The six findings (PR #260)

**F1 — queue path skipped the `inputAllowed` authorization check.**
`skills/dashboard/app/src/app/api/run/route.ts:163` enforced it; `skills/dashboard/runner/src/sweep.ts`
did not. A queue intent could carry arbitrary prompt text into a `bypass`-profile
button running under `--dangerously-skip-permissions`. Fixed by quarantining the
intent before `buildArgv` in `sweep.ts`, mirroring `route.ts`'s condition exactly —
same check, now in both call sites. See [[dashboard]]'s "Button / run model" section
for the `buildArgv`/profile-mapping contract this check protects.

**F2 — `entry.hash` path traversal.** `lib/collect/queueActions.ts` returned
`{...parsed, status}` — the object spread carried the *file's own* hash field, not
the validated parameter that had actually been checked — and `lib/build/spawn.ts`
did `join(buildsDir, entry.hash)` with that unvalidated value. A file-write
primitive, **not** code execution: `run-builder.sh` re-validates the hash and fails
closed before doing anything with it. Fixed at both layers: the validated hash is
now carried at the source (`queueActions.ts`), plus a second pre-join regex guard in
`spawn.ts` as defence-in-depth.

**F3 — config loader had no schema validation.** `lib/config.ts` iterated
`data.buttons` with no array guard (a missing key threw a raw `TypeError`) and no
type/charset check on a button's `name`, which becomes a lock **filename** via
`join(locksDir, name + ".lock")`. Fixed with an array guard, string-type checks on
each field, and an anchored `BUTTON_NAME_PATTERN`. A `cwd` allowlist was considered
and **explicitly not adopted** — the reviewer's stated reason: it buys nothing
against a same-user attacker, since config and code are the same trust tier for this
threat model (same conclusion as the F5 note below).

**F4 — directory/file permissions too loose.** Runtime dirs were `0755` and run logs
`0644`, sitting under a `0700` parent. Root cause: `mkdirSync`'s `mode` argument is a
no-op on an already-existing directory (Node behavior, not a coderails bug), so
specifying a tight mode only helps a fresh install — every live, already-installed
directory kept its original loose permissions forever. Fixed with explicit modes on
creation **plus** a `statSync`/`chmodSync` convergence block that runs on every
startup, so a live install tightens on its next launch rather than staying loose
until manually `chmod`'d.

**F5 — `artifactGate.escapesRoot` skips containment for templates lacking a
`{vault}` token.** Documentation-only disposition, no code change: config is treated
as the same trust tier as code for this threat model (same reasoning as F3's
rejected `cwd` allowlist) — a same-user attacker who could tamper with a template
already has write access to the code that would otherwise contain it.

**F6 — run-token compared with `!==` at three route sites.** Replaced with a shared
`tokensEqual` helper using `crypto.timingSafeEqual`, length-checked *first* — a
required guard, since `timingSafeEqual` throws on unequal-length buffers rather than
returning `false`. Honest severity assessment, stated directly rather than inflated:
the token is 256 bits **and** already embedded in page HTML, so a timing side-channel
was never the weak link here — this is hygiene, not a fix for an exploitable gap.

## Eight defences attacked and held (negative results, all verified at source)

The review didn't only find gaps — it attempted eight specific attacks against
existing code and confirmed each one holds:

1. **argv flag injection** — argv-array spawn, a trimmed leading-dash check on
   input, a `--` end-of-options sentinel, and a single concatenated prompt string.
   Could not be defeated. (Same mechanism [[dashboard]] already documents as
   "flag-smuggling closed by two independent layers" from PR #70's fix — this review
   re-attacked it independently and it still held.)
2. **DNS rebinding** — defended and explicitly tested, including the suffix-attack
   shape `192.168.50.140.evil.com` (a hostname containing the allowed LAN IP as a
   substring, not an exact match). `requestGuard.ts`'s exact-match `isAllowedHost()`
   (see [[pr_179_dashboard-lan-access]]) rejects it correctly.
3. **Output-route path traversal** — `runId` is format-validated
   (`/^[0-9a-f]{16}$/`), used only as a lookup key into `runs.jsonl`, and never
   itself joined into a filesystem path (matches the existing documented pattern —
   see [[dashboard]]'s Run Output viewer section).
4. **The exec lock** — `writeFileSync` with flag `wx` (exclusive create). The
   exclusive-create operation *is* the check; no TOCTOU window exists between
   checking and creating.
5. **osascript escalation** — arguments are passed via `on run argv`, never
   interpolated into AppleScript source text, so no injection surface exists.
6. **Run-builder hash re-validation + symlink rejection** — the second-layer check
   referenced in F2's disposition; confirmed to actually reject a symlinked path,
   not just a malformed hash string.
7. **SSE framing** — `JSON.stringify` on every event payload prevents an
   attacker-controlled string from forging a fake event line in the stream.
8. **Read-only profile cannot reach the queue** — `--allowedTools Read Grep Glob`
   on the `read-only` profile has no tool capable of writing a queue intent file in
   the first place.

## PR #263 — security review doc removed from the repo

`docs/DASHBOARD-SECURITY-REVIEW.md` (added by PR #260 itself, commit `b32686f`) was
removed from tracking and the path gitignored, on the reasoning that the document
catalogues live attack surface and shouldn't sit in a readable tracked file.
**Honest gap, stated rather than hidden:** the file remains readable from history at
commit `b32686f` — removing it from history would need a force-push, which was not
authorised as part of this change. The doc's removal reduces *current-tree*
visibility only, not historical visibility.

## MERGED ≠ DEPLOYED — the most important operational finding

The routine-sweeper launchd jobs (`com.coderails.routine-sweeper.calendar.plist`,
`com.coderails.routine-sweeper.watch.plist`) run the sweeper directly from the
**working checkout** — `ProgramArguments` points at
`/Users/harrison/Github/coderails/skills/dashboard/runner/bin/{seed-and-sweep,sweeper}.sh`
— not from an installed copy of the plugin. `(verified 2026-07-22)` That checkout was
57 commits behind `origin/main` at review time, so the F1 guard (queue-path
`inputAllowed` check) was verifiably **absent** from the code launchd actually runs,
despite being merged to `main`.

**Proven by live-fire, not by static analysis.** A queue intent carrying
unauthorized input against the `sync-docs` `bypass`-profile button reached a real
`claude` invocation running under `--dangerously-skip-permissions` — the argv was
captured directly in `runs.jsonl`. No damage resulted: the invoked skill recognised
the prompt as anomalous and made no tool calls. But that outcome is the **model's**
judgement call, not a property of the gate — the gate that was supposed to make this
impossible was not running. No unit test or mutation-proof of the F1 fix could have
caught this; only firing the actual attack at the actual running system did, because
the defect was never in the merged code — it was in what code was actually deployed.

This is the same class of gap [[enforcement-model]] documents for the tier-gate
daemon's own trust chain (root-owned config path, root-owned network tool) and for
`routines.md`'s boot-persistence finding ([[routines]]: bootstrapping a launchd job
from a repo path rather than an installed `~/Library/LaunchAgents/` copy silently
breaks on reboot) — a security or correctness property proven true of the source
tree is not automatically true of the running process, and the two can drift apart
silently with no error, no crash, no log entry marking the divergence.

## See also

- [[dashboard]] — the skill page this cluster's findings and fixes attach to
  (Button/run model, Run Output viewer, LAN-access sections)
- [[pr_179_dashboard-lan-access]] — the `DASHBOARD_HOST`/`isAllowedHost()` exact-match
  guard F6's sibling attack (DNS rebinding) was re-verified against
- [[enforcement-model]] — the ceiling framing this review's threat-model reframing and
  the MERGED ≠ DEPLOYED finding both instantiate: a hook or a fix proven correct in
  source is not proven correct in the process actually running
- [[routines]] — the boot-persistence finding (repo-path vs. installed-copy launchd
  bootstrap) that is the same drift class as this cluster's deployment gap
