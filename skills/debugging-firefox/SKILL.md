---
name: debugging-firefox
description: Use when inspecting Firefox browser chrome or add-on runtime behavior, working with classic DevTools RDP or --start-debugger-server, installing or reloading an XPI without restart, or proving a live Firefox regression in an existing profile.
license: MIT
compatibility: Requires Node.js 22+ and desktop Firefox exposing classic DevTools RDP browser-chrome actors. Executable path and loopback port are caller-selectable.
---

# Debugging Firefox

## Core principle

Treat the live Firefox session as user data. Use one task-owned loopback RDP socket and restore every safely reversible task-owned change; follow the cleanup matrix and report retained state when safe restoration is unavailable.

## Target and ownership

A supplied executable overrides discovery. Resolve that file, preserve spaces and non-ASCII executable paths with native file-path APIs and arbitrary argument boundaries with a true argument-vector API, identify its version when safe, and never substitute it. Otherwise discover without hardcoded locations across desktop Release, ESR, Beta, Developer Edition, Nightly, custom, portable/extracted, and side-by-side installations.

On macOS require the binary inside the application bundle; elsewhere accept the selected executable or launcher. A path does not authorize launch; launch only when a debugger listener is needed and ownership permits.

Before invoking `--start-debugger-server`, classify in this precedence: compatible listener, selected-target process presence and exact instance identity, then real browser-window evidence for an existing or task-started target. If no selected-target process and no listener exist, take the cold-start branch immediately: launch Firefox normally without the debugger flag before any window enumeration, prove the new real browser window, then invoke the debugger flag separately. If that real window already exists, skip the normal launch and invoke the flag separately. Processes confirmed by trusted enumeration to lack a real window are neither cold-start proof nor an attachable existing window: wait only for a normal launch this task just started or a trusted currently-starting user launch; otherwise stop. Never use a combined cold debugger-server launch. Normal launch, window proof, and diagnostic logging do not authorize the `new-window` option in either accepted dash spelling, `headless`, `screenshot`, hidden/no-new-window process settings, or any other argument that changes visible startup.

Window enumeration begins only after a selected-target process exists, either pre-existing or from the task-owned normal launch. With trusted zero-process and zero-listener evidence, the next action is that normal cold launch; do not call or recover a window enumerator first. Window evidence is tri-state: **confirmed browser window**, **confirmed absent**, or **unavailable/inconclusive**. Only a completed trusted enumeration correlated to the selected target can establish the first two. A tool bootstrap error or empty output from a sandbox-limited probe is unavailable—not absence. A screenshot/capture failure alone establishes neither state and does not invalidate a completed trusted listing. Keep pre-existing Firefox unchanged; retain a task-owned cold launch only through its bounded window wait. Make exactly one fresh list-only recovery attempt: prefer one fresh read-only subagent when it provides a fresh tool service/session; otherwise use one fresh local service/session, never both sequentially. Do not loop an in-process reset after a pre-execution/bootstrap failure. If fresh enumeration is unavailable or also fails, stop as `window evidence unavailable`, not process-only/no-window. Clean a causally proven task-owned cold-launch tree only after revalidating its identity, exclusive current ownership, and release by other tasks; otherwise leave and report it. Verify a cleaned tree's selected port is closed and leave pre-existing processes untouched. Restarting the agent host is an outside-task fallback after active work is safe; it is never a Firefox restart. The process-only stop branch applies only to confirmed absence.

Before starting, prove effective `devtools.debugger.force-local`; afterward verify the selected port binds only to `127.0.0.1`. Wildcard or non-loopback binding blocks use. Before launch, detect handoff to another instance and profile locks. Stop rather than adding `--no-remote`, creating/selecting a profile, or changing channels.

Allow one mutation owner per instance and one live task-owned socket at a time. Other tasks may use read-only sockets; mutation requires handoff. At contention, report the exact retained executable, open no competing mutation socket, and state restart requires ownership release plus separate approval.

## Workflow

1. Capture the baseline: target identity, browser state, ownership, and privacy-minimized restoration invariants.
2. Read [references/live-testing.md](references/live-testing.md) before listener start/connection, mutation, behavior proof, or restart. Install/reload claims require install/readiness and restoration; behavior claims require an exercised user-facing/native path. Otherwise mark behavior unverified and continue only separately requested install/reload work.
3. Request the listener at most once per retained Firefox instance; an authorized restart permits one request on its replacement. Unless Firefox logging already proves an explicit listener-open failure, follow the reference's OS/log evidence recipe and make the mandatory first real connection with [scripts/firefox-rdp.mjs](scripts/firefox-rdp.mjs), whose `connect()` ends with an effect-free `evaluateJSAsync` probe. If `tcpAccepted` remains false and every just-started task-owned readiness predicate passes, dispose that client and make at most one sequential connection retry. Once `tcpAccepted` is true, keep that socket through initial approval and capability preflight; do not reconnect during that preflight. Afterward, only the reference's explicitly scoped post-dispatch, restart, and cleanup replacements are allowed.
4. After dispatch, a timeout invalidates the socket; one authorized sequential replacement may only query task-owned authoritative state once before deciding whether task-operation replay is safe.
5. In `finally`, restore only task-owned changes; compare captured restoration invariants with the baseline; close the client; report retained Firefox listeners/processes.

An explicit listener-open failure, or an unhealthy preflight after the permitted pre-accept attempt(s), permits one pre-authorized restart with baseline, released ownership, and an independently available ordinary browser/UI quit path that does not depend on the failed RDP listener. Discard protocol state; rediscover and run one full replacement preflight cycle, including the standard eligible pre-accept retry. A second unhealthy result or explicit unsupported response stops.

## Quick reference

| Need | Gate |
|---|---|
| Target | Exact file, or fallback discovery |
| Launch | Existing listener, confirmed real window, cold start, confirmed absence, or recover unavailable evidence |
| Listener | One request per retained instance; first connection mandatory unless explicit open failure is proven; at most one eligible pre-accept retry |
| Mutation | Explicit owner or handoff |
| Readiness | Bounded predicate; authoritative check after timeout |
| Proof | Actual path plus restoration |

## Common mistakes

Never treat process presence, launcher PID/exit, `MainWindowHandle`, a listening port, a failed enumerator, empty sandbox-limited output, or screenshot denial as proof that a browser window is present or absent; loop a reset after a pre-execution helper failure; substitute port polling for the first real RDP attempt; change visible launch merely to capture logs; invent `Services.sys.mjs`; replay timed-out mutations; restart without approval; or use `gBrowser.adoptTab()` as drag proof.

## Boundary

Ordinary webpages use normal browser automation. Never terminate pre-existing Firefox or remove a requested add-on.

Run offline tests before live use:

```text
node scripts/firefox-rdp.test.mjs
```
