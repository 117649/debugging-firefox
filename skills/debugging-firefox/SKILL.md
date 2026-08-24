---
name: debugging-firefox
description: Use when inspecting Firefox browser chrome or add-on runtime behavior, working with classic DevTools RDP or --start-debugger-server, installing or reloading an XPI without restart, or proving a live Firefox regression in an existing profile.
license: MIT
compatibility: Requires Node.js 22+ and desktop Firefox exposing classic DevTools RDP browser-chrome actors. Executable path and loopback port are caller-selectable.
---

# Debugging Firefox

## Core principle

Treat the live Firefox session as user data. Use one live task-owned loopback RDP socket and restore every task-owned change.

## Target and ownership

An explicitly supplied executable overrides discovery. Resolve that exact file, preserve spaces and non-ASCII characters with argument arrays, identify its reported version when safe, and never substitute another installation. If no path is supplied, discover without hardcoded locations across desktop Release, ESR, Beta, Developer Edition, Nightly, custom, portable/extracted, and side-by-side installations.

On macOS require the Firefox binary inside the application bundle; on Windows and Linux accept the caller-selected executable or launcher. A supplied path does not authorize launch; launch only when a debugger listener is needed and ownership permits.

Use `127.0.0.1` and the caller-selected port. Before launch, detect handoff to a different running instance and profile locks. Stop rather than adding `--no-remote`, creating/selecting a profile, or changing channels.

Allow one mutation owner per instance and one live task-owned socket at a time. Other tasks may use read-only sockets; mutation requires handoff. At contention, report the exact retained executable, open no competing mutation socket, and state restart requires ownership release plus separate approval.

## Workflow

1. Capture the baseline: target identity, browser state, ownership, and privacy-minimized restoration invariants.
2. Connect once with [scripts/firefox-rdp.mjs](scripts/firefox-rdp.mjs). Record the root greeting and require `listProcesses`, a parent-process descriptor, `getTarget`, a console actor, and `evaluateJSAsync`; name missing capabilities exactly and stop rather than guess actor calls.
3. Read [references/live-testing.md](references/live-testing.md) for mutation or behavior proof. Install/reload claims require install/readiness and restoration; behavior claims require an exercised user-facing/native path. Otherwise mark behavior unverified and continue install-only work.
4. Use bounded readiness predicates. A timeout invalidates the socket; dispose it before one authorized sequential replacement, query task-owned authoritative state once, then decide whether one retry is safe.
5. In `finally`, restore only task-owned changes; compare captured restoration invariants with the baseline; close the client; report retained Firefox listeners/processes.

Restart requires separate authorization, a baseline, and prior release from other tasks. Afterward every actor, sentinel, socket, and snapshot is stale; rediscover the replacement instance.

## Quick reference

| Need | Gate |
|---|---|
| Target | Exact file, or fallback discovery |
| Mutation | Explicit owner or handoff |
| Readiness | Bounded predicate; authoritative check after timeout |
| Proof | Actual path plus restoration |

## Common mistakes

Do not keep concurrent task-owned sockets, replay a timed-out mutation, restart without separate approval, or use `gBrowser.adoptTab()` as native-drag proof.

## Boundary

Ordinary webpages use normal browser automation. Never terminate pre-existing Firefox or remove a requested add-on.

Run offline tests before live use:

```text
node scripts/firefox-rdp.test.mjs
```
