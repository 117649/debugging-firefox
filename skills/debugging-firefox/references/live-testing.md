# Live Firefox test contract

Read this before installing or reloading an XPI, changing live browser state, or taking the restart branch. Scale assertions to the requested behavior; do not turn a focused regression into a compatibility suite.

## Evidence ladder

Keep each claim at its highest completed level:

1. **Static:** source inspection and syntax checks.
2. **Package:** the exact XPI contains the intended files and metadata.
3. **Protocol:** framing, actor discovery, request correlation, and capability probes succeed.
4. **Install/readiness:** the intended add-on version is active in the target process and feature-specific readiness succeeds.
5. **Behavior:** the actual user-facing or native runtime path produces the expected result.
6. **Restoration:** task-owned state is removed and the shared session matches its baseline.

## Resolve the target before connecting

Target identity comprises the resolved executable; reported product, version, or build identifier when available; detected platform; PID and start time; caller-selected loopback port; profile identity; and a stable browser-instance sentinel.

An explicitly supplied executable overrides discovery. Validate it as the exact file and preserve it through argument arrays. A directory, missing file, substituted binary, channel mismatch, unsafe handoff to a different existing Firefox instance, or locked profile blocks launch. Do not silently add isolation flags, create/select a profile, or choose another channel. Discovery is only a fallback when no executable was supplied.

Record whether Firefox and the listener predated the task and whether this task launched either. Do not launch merely to inspect a supplied path. Connect only to `127.0.0.1` at the caller-selected port.

Before starting a listener, prove the selected profile and packaged defaults make `devtools.debugger.force-local` effectively true; do not silently change the user's preference. After start and before RDP connection, verify the OS listener is bound only to loopback. A wildcard (`0.0.0.0` or `::`) or non-loopback bind blocks use. Firefox 154 enforces this in [`SocketListener.open`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L393-L441) and [`SocketListener.host`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L482-L490).

On Windows, a `Start-Process` result or exited PID may be only the Firefox command-line forwarding helper, not the listener owner. Pass the exact executable as `-FilePath` and its tokens through `-ArgumentList`. Wait boundedly for the selected loopback port, resolve its owning Firefox process and retained instance, then run the full RDP preflight. Direct invocation is not a substitute for these checks.

## Ownership and baseline

- Establish one explicit mutation owner for the Firefox instance. Other tasks remain read-only and use separate sockets, or close their clients and explicitly hand off mutation ownership. If coordination is unavailable or target identity is uncertain, stop before mutation.
- Use one live task-owned socket at a time. Do not share it across tasks or reconnect per evaluation. After invalidation or restart, dispose the old client before the single authorized sequential replacement.
- Normally omit `localPort` or use `0` for a sequential replacement; reusing a fixed client port can fail with `EADDRINUSE` while the prior socket is in `TIME_WAIT`.
- Capture opaque window/tab order and selection, feature state, listener/process ownership, add-on state, and only the URLs/titles needed as evidence. Choose restoration invariants before mutation.
- Keep task-operation sentinels and live identities, including original method references, in a uniquely named property on a stable browser window. Keep only serializable restoration invariants in the client.
- For browser-managed groups or containers, retain group identity and neighboring anchors. Treat a global native index as diagnostic unless the requested behavior owns it.

## Capability gate

Begin with a small read-only probe using the tested client. Record the root greeting, traits, and actor forms. The supported parent-process path requires:

1. `listProcesses` on the root actor;
2. a parent-process descriptor;
3. `getTarget` on that descriptor;
4. a console actor on the target; and
5. `evaluateJSAsync` on the console actor.

If any part is missing, return `unsupported` with the exact missing capability and detected target identity. Do not guess alternate actors, packets, or privileged globals. Probe every version-sensitive API needed by the planned matrix before mutation.

This gate is the listener preflight and runs before any task operation. A connected socket that receives no root greeting may be waiting on Firefox's connection-approval prompt: [`Prompt.Server.authenticate`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/auth.js#L148-L204) invokes [`Server.defaultAllowConnection`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/prompt.js#L125-L164) before [`ServerSocketConnection._handle`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L574-L624) allows the connection. Before connecting, tell the user and choose a visible bounded deadline long enough for approval; keep that one socket while the user accepts or declines. Do not automate or bypass the prompt, open replacement sockets, or restart. If the deadline expires while prompt state is unknown, dispose the socket and report approval unresolved rather than listener failure. Only after approval is resolved or the prompt is ruled out does connection refusal/reset, a malformed or missing greeting, or timeout/transport failure in a mandatory capability mark the listener unhealthy. Dispose that client. With prior restart authorization, a complete baseline, and exclusive ownership, take the restart checkpoint immediately instead of opening a replacement socket to the same listener. Without those prerequisites, stop before the task call. Rerun the full gate once on the replacement instance; another unhealthy result stops without a task operation or second restart. Treat an explicit unsupported-capability response as incompatibility, not listener failure.

For privileged evaluation that needs XPCOM services, probe and use `globalThis.Services` in the selected target. Mainline Firefox 117 removed the legacy `resource://gre/modules/Services.jsm`; it was not renamed to `Services.sys.mjs`. If the global is absent, use the legacy JSM only after the target package or source proves it exists and the target exposes a compatible JSM importer; otherwise return `unsupported`. ESR and derived builds follow their actual capabilities, not the mainline version number. This boundary comes from [`xpc::InitGlobalObject`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_117_0_RELEASE/js/xpconnect/src/nsXPConnect.cpp#L505-L523), [`mozJSModuleLoader::DefineJSServices`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_117_0_RELEASE/js/xpconnect/loader/mozJSModuleLoader.cpp#L1890-L1913), and Firefox [bug 1780695](https://bugzilla.mozilla.org/show_bug.cgi?id=1780695).

## Same-process XPI install and readiness

Validate the exact XPI path, contents, expected add-on ID, and version first. Through the same parent-process connection, import `AddonManager`, construct an `nsIFile`, pass the native path serialized with `JSON.stringify(xpiPath)` to `initWithPath()`, and call `AddonManager.getInstallForFile(file)`. Capture candidate state/error before and after `install.install()`, then query the expected add-on ID, version, active/disabled flags, and restart requirement. Observed numeric states are run evidence, not cross-version constants.

Start asynchronous work behind the browser-window sentinel. Poll a small serialized status object with bounded `evaluateJson` or `pollJson` calls; raw RDP values may be protocol grips. Readiness is feature-owned: assert the predicate for every affected pre-existing window or frame. A navigation `tabNavigated` event with `state: "stop"` proves navigation completion, not application or network idle.

A timeout invalidates the socket but does not cancel dispatched Firefox-side work and never authorizes replay. Dispose the invalidated client, then create one authorized sequential replacement only to query the task-owned sentinel or authoritative add-on state once. Resume without replay if the operation succeeded. One retry is safe only when the operation is absent, conclusively failed without effects, or fully restored; otherwise stop and report the last state.

If an active same-version install still serves old chrome/resource bytes, do not bump the version or repeat installation as a cache workaround. Take the restart branch only with separate authorization; otherwise label the cache-sensitive behavior unverified.

## Real-path regression design

Test the earliest native path that reproduces the failure. Until the first divergent call or error is observed, the cause is **unlocalized**. A passing downstream shortcut narrows the boundary but is not the acceptance test.

For a native cross-window tab-drag regression:

1. Create one uniquely identifiable disposable tab.
2. Trace the native drag/drop handler with temporary in-memory hooks.
3. Perform drag-out, then return the replacement tab created by adoption.
4. Assert destination group identity, selection, contiguous native `_tPos` order, and no thrown errors or duplicate tabs.
5. Remove only the disposable tab/window and tracing.

Direct `gBrowser.adoptTab()` bypasses the native drop handler and cannot prove this regression.

## Restartless command-line listener shutdown

Closing the RDP client does not stop a listener created by `--start-debugger-server`: the startup handler deliberately sets `DevToolsServer.keepAlive = true`. Do not hide Firefox's remote-control cue with CSS or remove its DOM attribute; that conceals the security indicator while the port remains open. The cue is not port proof because [`gRemoteControl.getRemoteControlComponent`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/browser/base/content/browser.js#L4125-L4145) also covers Marionette and Remote Agent.

Use the internal path below only when the task started that command-line listener on a pre-existing Firefox, the target's exact source or packaged modules still match the named symbols, every other task has released the instance, no foreign RDP connection remains established, and the distinct server reports exactly one listener. Foreign-client release is a coordination fact, not a mechanically provable condition: Firefox exposes no public per-client ownership API, so uncertainty blocks shutdown. `closeAllSocketListeners()` is broad and closes every listener and accepted connection on that server. This path was live-proven on a Firefox-derived 154.0.1 build; it is not a public or cross-version API.

Evaluate this through the retained parent-process connection. If a healthy client was already closed intentionally, open exactly one cleanup connection only after the ownership checks, using the approval-prompt rule above and an ephemeral `localPort`; this is not a retry of an unhealthy listener. Releasing the temporary requester before closure preserves the startup requester's loader reference. The executing RDP socket closing before an evaluation reply is expected success evidence, not a reason to reconnect:

```javascript
(() => {
  const api = ChromeUtils.importESModule(
    "resource://devtools/shared/loader/DistinctSystemPrincipalLoader.sys.mjs",
    { global: "shared" }
  );
  const requester = {};
  const loader = api.useDistinctSystemPrincipalLoader(requester);
  let server;
  try {
    ({ DevToolsServer: server } = loader.require(
      "resource://devtools/server/devtools-server.js"
    ));
    if (server.listeningSockets !== 1 ||
        typeof server.closeAllSocketListeners !== "function") {
      return JSON.stringify({ safe: false, listeners: server.listeningSockets });
    }
  } finally {
    api.releaseDistinctSystemPrincipalLoader(requester);
  }
  server.closeAllSocketListeners();
})()
```

Never call `DevToolsServer.destroy()` or synthesize `quit-application`. Verify shutdown out of band with an explicit negative loopback-port probe plus proof that the retained Firefox PID remains alive; an empty error-suppressed socket query is not evidence. If source, symbol, ownership, or listener-count checks differ, leave the listener running until ordinary Firefox exit or a separately approved restart.

Firefox 154 source basis: [`DevToolsStartup.handleDevToolsServerFlag`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/startup/DevToolsStartup.sys.mjs#L1117-L1194), [`useDistinctSystemPrincipalLoader`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/loader/DistinctSystemPrincipalLoader.sys.mjs#L18-L43), [`DevToolsServer.closeAllSocketListeners`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/server/devtools-server.js#L235-L292), and [`SocketListener.close`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L463-L480).

## Restart checkpoint

Restart only for a pre-authorized unhealthy listener preflight or after restartless operation is shown insufficient and the user separately approves it. Capture the executable/profile, original instance sentinel, PID/process tree, listener, serializable session invariants, relevant add-on state, and task-owned resources. Require every other known task to release mutation ownership and close its client; uncertain release blocks restart.

Request an ordinary quit through the sole authorized control path. Socket closure is expected: dispose the old client and wait to the diagnostic deadline without killing a shared or pre-existing process. Relaunch the same executable and profile without adding isolation flags. Detect the replacement PID, expose the caller-approved loopback listener, create one new client, rediscover capabilities, and capture a new sentinel and actor set. Old actors, sentinels, sockets, and snapshots are invalid.

Wait for feature readiness and authoritative session restoration, repeat the intended real-path behavior, then compare the captured invariants. Recreate a task-owned captured page once only if absent after restoration completes; do not reorder unrelated tabs. Record what was restored and every remaining difference.

## Cleanup matrix

| Initial ownership | Required end state |
|---|---|
| Firefox and listener pre-existed | Close only this task's client; leave both running. |
| Task started listener on pre-existing Firefox | Use the source-gated restartless shutdown when its ownership and compatibility checks pass; otherwise close the client, leave Firefox running, and report the retained listener. |
| Task launched dedicated Firefox | Close the client, terminate only the recorded owned process tree, and verify its listener stopped. |

In `finally`, restore task-owned globals, wrapped methods, preferences, tabs/groups, window selection/focus, and feature layout. Do not uninstall an add-on the user asked to install or update, and never clean up another task's resources.

## Result contract

Report the resolved executable, reported product and version/build, and detected platform; PID/start time; opaque/redacted profile identity and instance sentinel; port and ownership; capability path; XPI identity; install state/error; per-window readiness; actual-path assertions; restoration comparison; client closure; and any listener/process left running.

Name the highest evidence level reached. Label **static inference**, **live observation**, and **unverified compatibility** separately. A build or package check does not prove live Firefox behavior; a completed install does not prove readiness or the real path; blocked or skipped checks remain unverified.
