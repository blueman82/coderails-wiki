---
title: "PR #179 — dashboard LAN access (opt-in)"
type: source
created: 2026-07-15
last_updated: 2026-07-15
sources: []
tags: [source, dashboard, lan-access, dashboard-host, request-guard, security-posture, launchd]
---

# PR #179 — dashboard LAN access (opt-in)

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #179 |
| Title | dashboard lan access |
| Merged | 2026-07-15T17:21:28Z |
| Merge SHA | `b8fbe190511b00bc04ec4154d226e23e31d93661` |
| JIRA ticket | — |

## Summary

Adds opt-in LAN exposure to the observability dashboard via a single env var, `DASHBOARD_HOST`. Previously the dashboard bound `127.0.0.1` unconditionally and `requestGuard.ts` accepted only loopback `Host`/`Origin` — that behaviour is unchanged when `DASHBOARD_HOST` is unset/empty. When set to a concrete host IP, the bind and the guard change together: both serving paths bind that address instead of loopback, and the guard additionally exact-matches that one host (never a relaxation to "any non-loopback host") for both `Host` and `Origin`, on top of loopback which remains always-allowed. DNS-rebinding and cross-origin requests are still rejected exactly as before — the guard is never bypassed, only its allowlist gains one more literal entry.

## Files changed

- `skills/dashboard/scripts/start-dashboard.sh` — reads `DASHBOARD_HOST` (default `127.0.0.1`), validates it, binds it
- `skills/dashboard/runner/bin/dashboard-server.sh` — same read/validate/bind, for the launchd-managed process
- `skills/dashboard/app/src/lib/requestGuard.ts` — new `isAllowedHost()` helper (loopback always; `process.env.DASHBOARD_HOST` exact-match additionally); `isLocalOrigin()`'s two call sites (`Host` header, `Origin` header) switched from `isLocalhost()` to `isAllowedHost()`
- `skills/dashboard/app/test/requestGuard.test.ts` — new; full allow/reject matrix for both default and LAN modes
- `launchd/com.coderails.dashboard.plist` — new `EnvironmentVariables` dict with an empty `DASHBOARD_HOST` entry (loopback default preserved until filled in)
- `skills/dashboard/SKILL.md` — new "LAN access (opt-in)" section

## Validation behaviour (both scripts, identical shape)

`DASHBOARD_HOST` unset/empty: falls through to `127.0.0.1`, no validation branch runs. When set, a `case` statement:
- Accepts `localhost`, `127.0.0.1`, `::1` (loopback spellings), a concrete IPv4 literal (`^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$`), or a bare IPv6 literal (contains `:`).
- Rejects `0.0.0.0`, `::`, `*` (wildcard binds) and any `ip:port` form, with a non-zero exit and a stderr message naming the reason before `npm run start`/`exec` is ever reached.

Rationale stated in the script comments: the guard exact-matches **one** literal host string, so a wildcard bind would silently 403 every real LAN request that reached it — validating at startup converts that into a fail-loud config error instead. Under the launchd agent (`KeepAlive` + `ThrottleInterval 60`), a rejected value means the agent respawns and fails every 60s until fixed, which is treated as correct behaviour for a misconfiguration, not a bug to route around.

## requestGuard.ts change, precisely

`isAllowedHost(hostname)`: `true` if `isLocalhost(hostname)`; else `false` if `DASHBOARD_HOST` is unset/empty; else compares `stripIpv6Brackets(hostname) === stripIpv6Brackets(process.env.DASHBOARD_HOST)`. Read fresh from `process.env` on every call (matches the app's existing direct-env-read style elsewhere, e.g. `build/spawn.ts`) rather than cached at module load, specifically so tests can flip it per-case. `isLocalOrigin()` calls this once for the `Host` header's hostname and once for the `Origin` header's hostname (when an `Origin` header is present at all) — both gates must pass, unchanged structurally from the pre-PR loopback-only version.

Test coverage (`requestGuard.test.ts`) exercises: loopback accepted in both modes; an arbitrary host rejected in both modes; a LAN host rejected when `DASHBOARD_HOST` is unset (proves the default mode doesn't accidentally widen); the configured LAN host accepted on both `Host` and `Origin` when set; an arbitrary `Host` still rejected even with `DASHBOARD_HOST` set (DNS-rebinding defence intact); a `Host` that merely contains the configured host as a substring (`192.168.50.140.evil.com`) rejected — an exact-match check, not a substring/prefix check; a present-but-mismatched `Origin` rejected even with a matching `Host` (cross-origin still blocked); a second, different LAN host rejected; a bracketed IPv6 literal (`[fe80::1]`) accepted when configured as such.

## Security posture — documented as deliberate

The dashboard's `POST /api/run` (COMMAND DECK) and the workflow-audit Approve/Deny queue execute declared commands with no authentication of any kind — this predates PR #179 (see [[dashboard]]'s "Button / run model" section) and is unchanged by it. The `Host`/`Origin` guard this PR extends defends against a hostile web page reaching the dashboard from a browser tab already open on the machine, or a DNS-rebinding attack — it does not, and was never intended to, authenticate individual devices on a network. Once `DASHBOARD_HOST` is set, any device on that LAN that can reach the port can trigger declared runs.

This is recorded as a **known, deliberate posture**, not an oversight: the decision to ship LAN access without adding device-level authentication was made autonomously under a crack-on envelope, scoped explicitly to "trusted home network only" use (stated directly in `SKILL.md`'s new section and in the `dashboard-server.sh`/`start-dashboard.sh` comments). No auth mechanism (token, mTLS, allowlist-by-MAC, etc.) was evaluated or deferred-with-a-ticket in this PR — the scope was the bind+guard mechanism only.

## Wiki pages updated

- [[dashboard]] — "Architecture" section's stale "Binds `127.0.0.1` only" claim corrected to "by default"; new "LAN access (opt-in)" subsection under Starting/stopping; CSRF-guard sentence in "Button / run model" reworded from "non-localhost" to the allowed-set framing; frontmatter `sources`/`tags`/`last_updated` updated
- `index.md` — new source-table row
- `log.md` — new ingest entry

## Caveats / gotchas

- The plist ships with `DASHBOARD_HOST` present but **empty** — installing/reinstalling the agent without filling it in changes nothing; the operator must edit the plist and reinstall for LAN access to actually take effect on the persistent agent.
- A DHCP-assigned LAN IP that changes will make the dashboard **fail to bind on next start** (validation now requires a concrete IP, and if the old IP is stale it just won't match the interface — this shows as a startup failure, not a silent partial-listen). `SKILL.md` recommends a DHCP reservation/static lease for this reason.
- `design/dashboard-runner.md` documents a *different* runner (`skills/dashboard/runner`'s intent-queue sweeper) that shares a directory name with `runner/bin/dashboard-server.sh` but is unrelated to this PR's bind/guard change — checked during this ingest and confirmed to contain no stale bind claims requiring correction.
