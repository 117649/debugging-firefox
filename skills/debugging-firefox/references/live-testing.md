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

Before connection, target identity comprises the resolved executable; reported product, version, or build identifier when available; detected platform; PID and start time; caller-selected loopback port; profile identity; lineage; and correlated browser-window evidence when the selected launch branch or browser-UI work requires it. After capability preflight, add a stable browser-instance sentinel before mutation.

An explicitly supplied executable overrides discovery. Validate it as the exact file, preserve its path with the host's native file-path API, and use a true argument-vector API for arbitrary arguments. A directory, missing file, substituted binary, channel mismatch, unsafe handoff to a different existing Firefox instance, or locked profile blocks launch. Do not silently add isolation flags, create/select a profile, or choose another channel. Discovery is only a fallback when no executable was supplied.

Record whether Firefox and the listener predated the task and whether this task launched either. Do not launch merely to inspect a supplied path. Connect only to `127.0.0.1` at the caller-selected port.

Before starting a listener, prove the selected profile and packaged defaults make `devtools.debugger.force-local` effectively true; do not silently change the user's preference. Before any task operation, prove the OS listener is bound only to loopback and owned by the retained target. A wildcard (`0.0.0.0` or `::`) or non-loopback bind blocks use. The narrowly scoped first-connection branch below may resolve otherwise inconclusive Windows enumeration for a just-started task-owned listener; it never waives owner/bind proof before mutation. Firefox 154 enforces loopback in [`SocketListener.open`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L373-L417) and [`SocketListener.host`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L455-L463).

## Listener launch state gate

Classify in this precedence order: existing listener, selected-target process presence and identity, then real browser-window state only for an existing or task-started target. Trusted no-process and no-listener evidence selects cold start without a pre-launch window-enumeration attempt. Do not invoke the debugger flag until the selected target fits one row:

| State | Required action |
|---|---|
| Compatible listener already exists | Launch nothing. Prove loopback bind, owner, exact executable/profile, retained instance, and full RDP preflight. Work requiring browser UI also requires a real window. Unknown or mismatched ownership stops. |
| No selected-target process and no listener | Cold start: launch the selected executable normally with no debugger flag or invented profile/isolation arguments. Causally correlate the retained tree to this launch by baseline delta, exact executable/profile, start time, retained process/window identity, and lineage where available; a delta alone is not ownership. Wait boundedly for a real browser window, then make the separate debugger-server invocation. A combined cold debugger-server launch is forbidden. |
| Matching real browser window, no listener | Record the pre-existing retained instance and window. Do not normal-launch Firefox. Invoke the exact executable once with `--start-debugger-server PORT`, then prove the retained listener owner rather than the helper PID. |
| Window evidence unavailable/inconclusive | Follow the recovery contract below; without trusted correlation, stop before the listener. |
| Successful trusted enumeration proves selected-target processes but no real browser window | This is neither cold start nor existing-window attachment. Do not invoke the debugger server. If this task just made the normal launch, finish its bounded window wait and clean its proven tree on timeout only under the cleanup-matrix identity and ownership gates; otherwise leave and report it. If trusted PID start time or explicit user context proves a user launch is still starting, make one bounded read-only window wait/re-enumeration and leave it untouched on timeout. Otherwise stop and leave all processes untouched. |
| Executable, profile, listener, or retained-instance mismatch | Stop. Do not substitute a binary/profile, add `--no-remote`, or create another listener. |

An enumerator has exactly three outcomes: a correlated browser window, confirmed absence after a successful bounded enumeration, or unavailable/inconclusive evidence. Invocation errors, service/bootstrap failures, empty results from a probe known to be isolated from the interactive desktop, and screenshot/capture failures are not completed enumerations. Preserve the exact error and do not convert the third outcome into the second.

When a window tool fails before enumeration for an existing process or after a task-owned normal launch, allocate exactly one fresh recovery
service/session. Prefer one fresh read-only subagent when it provides a separate tool service/session; otherwise use one fresh local service/session.
Do not try both sequentially. A separate subagent service prevents a stale parent-task helper from ending the Firefox task.

In that fresh session, first follow the current tool's required initialization or discovery call and inspect the API it actually exposes. Do not reuse
an import, global binding, method name, or example remembered from another session or an older tool version. First-use documentation and errors such
as an undefined binding, missing module, or missing method before a supported native-window list/state operation starts are capability validation,
not enumeration. If a stale call fails that way, use the returned current documentation to make at most one corrected supported list/state call in
the same fresh session. Do not reset it, allocate another service/session, or dispatch a second supported list/state operation.

The service/session allocation is the outer one-shot budget; dispatch of a supported native-window list/state operation is the inner one-shot budget.
A browser-tab inventory, a tool state with no native-app/window surface, or output that cannot be correlated to the retained Firefox process is
unavailable/inconclusive, not confirmed absence. If the fresh service cannot initialize, no supported native-window operation exists, or the single
operation fails or remains uncorrelatable, report `window evidence unavailable`. A bootstrap failure leaves zero enumeration attempts but never
authorizes a second fresh service/session. A full agent-host relaunch is a later user-controlled fallback after active tasks finish. A screenshot
policy error after successful listing does not invalidate the listing.

Independently correlate candidates to the retained process and selected executable/profile. On Windows require a visible, non-cloaked top-level candidate; process presence, launcher PID/exit, a nonzero `MainWindowHandle`, a blank-title/hidden window, helper success, and listener appearance are not proof. An `EnumWindows`-based native probe can reject invisible, cloaked, owned, zero-area, remoting, or wrong-executable HWNDs, but a survivor is only a browser-window candidate: Firefox's main window declares `windowtype="navigator:browser"` in [`browser.xhtml`](https://hg.mozilla.org/releases/mozilla-beta/file/490e9bad7f985070dce4abc80b1fbe6b11567d00/browser/base/content/browser.xhtml), while Browser Toolbox declares `windowtype="devtools:toolbox"` in [`window.html`](https://hg.mozilla.org/releases/mozilla-beta/file/490e9bad7f985070dce4abc80b1fbe6b11567d00/devtools/client/framework/browser-toolbox/window.html). [`nsWindow::ChooseWindowClass`](https://hg.mozilla.org/releases/mozilla-beta/file/490e9bad7f985070dce4abc80b1fbe6b11567d00/widget/windows/nsWindow.cpp#l1371) does not see that DOM attribute, and [`nsWindow::GetMainWindowClass`](https://hg.mozilla.org/releases/mozilla-beta/file/490e9bad7f985070dce4abc80b1fbe6b11567d00/widget/windows/nsWindow.cpp#l7942) permits a class override. Final positive proof therefore needs trusted automation that distinguishes browser chrome, or—when a compatible listener already exists—a browser-side check for `windowtype === "navigator:browser"` and `gBrowser`.

On Windows pass the exact executable through `Start-Process -FilePath`. For this listener invocation, `-ArgumentList @('--start-debugger-server', [string]$port)` is safe because both argument tokens contain no spaces; PowerShell otherwise joins array elements into one string, so arbitrary arguments require `System.Diagnostics.ProcessStartInfo.ArgumentList` or verified explicit quoting. The result or exited PID may be only the command-line forwarding helper. After the separate debugger invocation, take bounded error-visible listener evidence, resolve its owning Firefox process and retained instance when possible, then follow the recipe below. An empty error-suppressed port query is not evidence.

## Listener evidence, logs, and connection attempts

Count the listener request separately from client attempts. Invoke the selected executable with `--start-debugger-server PORT` at most once per retained Firefox instance; an authorized restart permits exactly one new request on the replacement instance. Never replay a request because a port enumerator is empty or a client is refused.

On Windows, run `Get-NetTCPConnection` without suppressing errors. If it errors or has no exact row, cross-check the full port in `netstat.exe -ano -p tcp`; retain the error and rows instead of collapsing them to empty. A wildcard/non-loopback bind, conflicting rows, or a foreign owner stops use. An exact loopback row owned by the retained Firefox proceeds normally.

For a just-started task-owned listener only, an inconclusive or empty socket table does not cancel the mandatory first read-only RDP attempt when the pre-start port was free, effective `force-local` and the exact target source/package prove `LoopbackOnly`, the window/profile/instance/ownership evidence remains unchanged, and neither OS check reports an unsafe or foreign bind. Connect to `127.0.0.1` once, then recheck bind and owner before any task operation. This branch never applies to a pre-existing or unknown listener.

Stdout/stderr capture is optional diagnostic evidence. Use logs already available; for a task-owned cold launch, redirect to task-owned files only when that preserves the exact ordinary visible launch and drains without blocking. Logging does not authorize the `new-window` option in either accepted dash spelling, `headless`, `screenshot`, `-WindowStyle Hidden`, `-NoNewWindow`, `CreateNoWindow`, another profile/executable, or a restart. Do not relaunch a pre-existing instance merely to obtain logs. Inspect both the normal-launch and separate listener-invocation streams; a forwarded listener handler may write through the retained Firefox process rather than the helper. If capture could alter startup or no real browser window is independently proven, omit it and use the other evidence gates.

[`DevToolsStartup.handleDevToolsServerFlag`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/startup/DevToolsStartup.sys.mjs#L1034-L1105) calls `listener.open()` without awaiting its result and immediately dumps `Started devtools server on PORT`; [`SocketListener.open`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L373-L417) reports a rejected open as `Could not start debugging listener on 'PORT': ...`. The first line proves the handler reached the open request, not that binding ultimately succeeded. A matching second line is explicit open failure: do not make a known-doomed client attempt; stop or take the pre-authorized restart checkpoint. Missing lines are inconclusive.

Unless an explicit listener-open failure is already proven, the first real RDP attempt is mandatory and is not a retry. Inspect `FirefoxRdpClient.tcpAccepted` after failure. When it is `false`, dispose that client and make at most one sequential retry only if the listener was just requested by this task, the same bounded readiness deadline remains, target/window/ownership evidence is unchanged, OS/log refresh shows no explicit open failure or unsafe/foreign bind, and no task operation ran. Use a new client with an ephemeral local port; do not replay the listener request. A second pre-accept failure marks that listener unhealthy. For a pre-existing compatible listener, the first pre-accept failure ends preflight as unhealthy with no retry. When `tcpAccepted` is `true`, keep that socket through approval and capability preflight with no retry or reconnect; afterward only the explicitly scoped post-dispatch, restart, and cleanup replacements below are allowed.

Firefox 154.0.1 source explains the split. [`XREMain::XRE_mainStartup`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/toolkit/xre/nsAppRunner.cpp#L4951-L5011) selects the profile, calls `nsRemoteService::StartClient`, and exits the helper when forwarding succeeds; [`nsRemoteService::StartClient`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/toolkit/components/remote/nsRemoteService.cpp#L172-L226) forwards the whole command line through `nsWinRemoteClient`. [`nsWinRemoteServer::Startup`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/toolkit/components/remote/nsWinRemoteServer.cpp#L59-L85) creates a hidden remoting window, so forwarding success does not prove a real browser window. [`DevToolsStartup.handle`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/devtools/startup/DevToolsStartup.sys.mjs#L339-L390) and [`handleDevToolsServerFlag`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/devtools/startup/DevToolsStartup.sys.mjs#L1034-L1104) accept the flag on both initial and forwarded command lines, while [`nsDefaultCommandLineHandler.handle`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/browser/components/BrowserContentHandler.sys.mjs#L1611-L1632) may open the normal window on initial launch. This contract deliberately stages cold launch so window proof precedes the listener request. When remote debugging is enabled and the forwarded flag is handled, [`WinRemoteMessageReceiver::ParseV3`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/toolkit/components/remote/WinRemoteMessage.cpp#L58-L100) supplies `STATE_REMOTE_AUTO`, `handleDevToolsServerFlag` sets `preventDefault`, and the default handler's [no-URL branch](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_1_RELEASE/browser/components/BrowserContentHandler.sys.mjs#L1616-L1640) does not open a browser window. Thus a handled forwarded server flag can create a listener without creating the missing browser window, and port readiness never proves UI readiness.

## Ownership and baseline

- Establish one explicit mutation owner for the Firefox instance. Other tasks remain read-only and use separate sockets, or close their clients and explicitly hand off mutation ownership. If coordination is unavailable or target identity is uncertain, stop before mutation.
- Use one live task-owned socket at a time. The pre-accept retry above creates a new client only after the unaccepted client is disposed. Do not share sockets across tasks or reconnect per evaluation; every other sequential replacement must be the explicitly scoped post-dispatch, restart, or cleanup branch below.
- Normally omit `localPort` or use `0` for a sequential replacement; reusing a fixed client port can fail with `EADDRINUSE` while the prior socket is in `TIME_WAIT`.
- Capture opaque window/tab order and selection, feature state, listener/process ownership, add-on state, and only the URLs/titles needed as evidence. Choose restoration invariants before mutation.
- After capability preflight and before mutation, keep task-operation sentinels and live identities, including original method references, in a uniquely named property on a stable browser window. Keep only serializable restoration invariants in the client.
- For browser-managed groups or containers, retain group identity and neighboring anchors. Treat a global native index as diagnostic unless the requested behavior owns it.

## Capability gate

Begin with a small read-only probe using the tested client. `FirefoxRdpClient.connect()` performs the final effect-free `evaluateJSAsync("void 0")` probe before it returns. Record the root greeting, traits, and actor forms. The supported parent-process path requires:

1. `listProcesses` on the root actor;
2. a parent-process descriptor;
3. `getTarget` on that descriptor;
4. a console actor on the target; and
5. `evaluateJSAsync` on the console actor.

If any part is missing, return `unsupported` with the exact missing capability and detected target identity. Do not guess alternate actors, packets, or privileged globals. Probe every version-sensitive API needed by the planned matrix before mutation.

This gate is the listener preflight and runs before any task operation. An accepted socket that receives no root greeting may be waiting on Firefox's connection-approval prompt: [`Prompt.Server.authenticate`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/auth.js#L148-L204) invokes [`Server.defaultAllowConnection`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/prompt.js#L125-L164) before [`ServerSocketConnection._handle`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L547-L556) allows the connection. Before connecting, tell the user and choose a visible bounded deadline long enough for approval; keep that one socket while the user accepts or declines. Do not automate or bypass the prompt, open replacement sockets during preflight, or restart. If the deadline expires while prompt state is unknown, dispose the socket and report approval unresolved rather than listener failure. Once TCP was accepted, only after approval is resolved or the prompt is ruled out does reset, a malformed or missing greeting, or timeout/transport failure in a mandatory capability mark the listener unhealthy. Dispose that client. With prior restart authorization, a complete baseline, and exclusive ownership, take the restart checkpoint immediately instead of opening a replacement socket to the same listener. Without those prerequisites, stop before the task call. Run one full gate cycle on the replacement instance, including the standard eligible pre-accept retry; another unhealthy result stops without a task operation or second restart. Treat an explicit unsupported-capability response as incompatibility, not listener failure.

For privileged evaluation that needs XPCOM services, probe and use `globalThis.Services` in the selected target. Mainline Firefox 117 removed the legacy `resource://gre/modules/Services.jsm`; it was not renamed to `Services.sys.mjs`. If the global is absent, use the legacy JSM only after the target package or source proves it exists and the target exposes a compatible JSM importer; otherwise return `unsupported`. ESR and derived builds follow their actual capabilities, not the mainline version number. This boundary comes from [`xpc::InitGlobalObject`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_117_0_RELEASE/js/xpconnect/src/nsXPConnect.cpp#L465-L489), [`mozJSModuleLoader::DefineJSServices`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_117_0_RELEASE/js/xpconnect/loader/mozJSModuleLoader.cpp#L1750-L1772), and Firefox [bug 1780695](https://bugzilla.mozilla.org/show_bug.cgi?id=1780695).

## Same-process XPI install and readiness

Validate the exact XPI path, contents, expected add-on ID, and version first. Through the same parent-process connection, import `AddonManager`, construct an `nsIFile`, pass the native path serialized with `JSON.stringify(xpiPath)` to `initWithPath()`, and call `AddonManager.getInstallForFile(file)`. Capture candidate state/error before and after `install.install()`, then query the expected add-on ID, version, active/disabled flags, and restart requirement. Observed numeric states are run evidence, not cross-version constants.

Start asynchronous work behind the browser-window sentinel. Poll a small serialized status object with bounded `evaluateJson` or `pollJson` calls; raw RDP values may be protocol grips. Readiness is feature-owned: assert the predicate for every affected pre-existing window or frame. A navigation `tabNavigated` event with `state: "stop"` proves navigation completion, not application or network idle.

A timeout invalidates the socket but does not cancel dispatched Firefox-side work and never authorizes replay. Dispose the invalidated client, then create one authorized sequential replacement only to query the task-owned sentinel or authoritative add-on state once. Resume without replay if the operation succeeded. Task-operation replay is safe only when the operation is absent, conclusively failed without effects, or fully restored; otherwise stop and report the last state.

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

Closing the RDP client does not stop a listener created by `--start-debugger-server`: the startup handler deliberately sets `DevToolsServer.keepAlive = true`. Do not hide Firefox's remote-control cue with CSS or remove its DOM attribute; that conceals the security indicator while the port remains open. The cue is not port proof because [`gRemoteControl.getRemoteControlComponent`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/browser/base/content/browser.js#L3870-L3890) also covers Marionette and Remote Agent.

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

Firefox 154 source basis: [`DevToolsStartup.handleDevToolsServerFlag`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/startup/DevToolsStartup.sys.mjs#L1034-L1105), [`useDistinctSystemPrincipalLoader`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/loader/DistinctSystemPrincipalLoader.sys.mjs#L18-L43), [`DevToolsServer.closeAllSocketListeners`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/server/devtools-server.js#L235-L292), and [`SocketListener.close`](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_154_0_RELEASE/devtools/shared/security/socket.js#L441-L454).

## Restart checkpoint

Restart only for a pre-authorized explicit listener-open failure, a pre-authorized unhealthy listener preflight, or after restartless operation is shown insufficient and the user separately approves it. Before listener start, the restart plan must identify an independently usable ordinary browser/UI quit path that does not depend on RDP; absence of that path blocks restart. Capture the executable/profile, original instance sentinel, PID/process tree, listener, serializable session invariants, relevant add-on state, and task-owned resources. Require every other known task to release mutation ownership and close its client; uncertain release blocks restart.

Request an ordinary quit through that pre-authorized browser/UI path; never open another RDP client solely to quit. Socket closure is expected: dispose any old client and wait to the diagnostic deadline without killing a shared or pre-existing process. If the retained process tree has not exited by that deadline, stop and leave it running. Only after confirmed exit, relaunch the same executable and profile without adding isolation flags. Detect the replacement PID, expose the caller-approved loopback listener once on that replacement, run one full preflight cycle with its standard eligible pre-accept retry, and capture a new sentinel and actor set. Old actors, sentinels, sockets, and snapshots are invalid.

Wait for feature readiness and authoritative session restoration, repeat the intended real-path behavior, then compare the captured invariants. Recreate a task-owned captured page once only if absent after restoration completes; do not reorder unrelated tabs. Record what was restored and every remaining difference.

## Cleanup matrix

| Initial ownership | Required end state |
|---|---|
| Firefox and listener pre-existed | Close only this task's client; leave both running. |
| Task started listener on pre-existing Firefox | Use the source-gated restartless shutdown when its ownership and compatibility checks pass; otherwise close the client, leave Firefox running, and report the retained listener. |
| Task cold-started dedicated Firefox | Close the client, then revalidate retained-instance identity and exclusive current ownership and require known other tasks to release it. Only then terminate the causally proven task-owned tree and verify both the tree and its listener stopped. User adoption, another task's use, or incomplete proof leaves Firefox running and reported. |

In `finally`, restore task-owned globals, wrapped methods, preferences, tabs/groups, window selection/focus, and feature layout. Do not uninstall an add-on the user asked to install or update, and never clean up another task's resources.

## Result contract

Report the resolved executable, reported product and version/build, and detected platform; selected launch branch and real-window evidence; PID/start time; opaque/redacted profile identity and instance sentinel; port and ownership; listener-request count; Windows listener evidence and Firefox-log evidence; client-attempt count and `tcpAccepted` result for each; capability path; XPI identity; install state/error; per-window readiness; actual-path assertions; restoration comparison; client closure; and any listener/process left running.

Name the highest evidence level reached. Label **static inference**, **live observation**, and **unverified compatibility** separately. A build or package check does not prove live Firefox behavior; a completed install does not prove readiness or the real path; blocked or skipped checks remain unverified.
