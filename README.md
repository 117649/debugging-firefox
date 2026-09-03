# Debugging Firefox

Debug browser-chrome and add-on runtime behavior through Firefox's classic DevTools Remote Debugging Protocol (RDP), while preserving the user's existing session. Classic RDP can evaluate privileged browser-chrome code and install an XPI permanently: use only a task-owned loopback connection and treat the live browser as user data.

## Scope and non-goals

This skill is for desktop Firefox browser-chrome or add-on runtime work, classic RDP capability probing, restartless XPI install/reload, and evidence-backed live regression checks. It is not ordinary webpage automation; use normal browser automation for that. It excludes mobile Firefox and does not claim compatibility with it.

## Compatibility

| Surface | Designed for | Exact versions actually tested |
| --- | --- | --- |
| Node client | Node.js 22+ | The deterministic Node suite is the offline acceptance gate. This repository does not record a release Node/OS compatibility matrix. |
| Firefox target | Official desktop channels—Release, ESR, Beta, Developer Edition, and Nightly—on Windows, macOS, and Linux when the required classic RDP capabilities exist; Firefox-derived desktop browsers are best-effort only. This is not a universal-compatibility claim. | Bounded live evidence: Firefox Developer Edition on Windows, with exact CLI label `Mozilla Firefox 155.0b3`, reached Level 3 Protocol only. One effect-free identity query failed; after that socket closed, one sequential replacement succeeded. No XPI install, readiness, behavior, or restoration was tested. |
| Agent host | Hosts that support the [Agent Skills specification](https://agentskills.io/specification) | Bounded live evidence: Codex Desktop 26.825.6671 on Windows recovered a stale parent-task Computer Use enumerator through one fresh read-only subagent and listed the existing Firefox windows. No Firefox restart or browser mutation was performed. Other hosts remain unverified. |

OpenAI supports standalone skills, while plugins are the separate preferred packaging route for marketplace distribution. This repository intentionally remains standards-first and does not imply marketplace publication.

## Install

With a GitHub CLI that provides the preview skill commands, install the named skill:

```text
gh skill install 117649/debugging-firefox debugging-firefox
```

Or manually copy `skills/debugging-firefox` into the supported skill directory for your agent host. Do not assume a host-specific directory from this repository; use that host's documentation.

## Exact executable-path input

Give the skill the exact executable file, including spaces, when you want to override discovery. For example:

```text
C:\Program Files\Mozilla Firefox\firefox.exe
```

On macOS, supply the binary inside the application bundle, such as `/Applications/Firefox.app/Contents/MacOS/firefox`, not the `.app` bundle itself. An explicit path overrides discovery, but it does not itself authorize a launch; the skill launches only when a debugger listener is needed and ownership permits it.

Permission to launch normally, prove a real window, or capture diagnostics does not authorize Firefox's `new-window` option in either accepted dash spelling, headless/screenshot mode, hidden/no-new-window process settings, or a different profile or executable.

## Offline test and validation

From the repository root, run the deterministic offline suite, whitespace check, and current-spec validation:

```text
node skills/debugging-firefox/scripts/firefox-rdp.test.mjs
git diff --check
gh skill publish --dry-run
```

The test starts only its local mock RDP server. It does not download, launch, connect to, or modify Firefox. The GitHub CLI dry run validates without publishing.

## Optional live-test evidence ladder

Read `skills/debugging-firefox/references/live-testing.md` before any live mutation. Record the highest completed level only: static, package, protocol, install/readiness, behavior, and restoration. A build, package check, or successful install is not proof of the user-facing runtime path.

Live work requires a resolved executable and selected loopback port, one mutation owner, one task-owned socket, a captured baseline, and a restoration
plan. Resolve an existing listener first. Otherwise prove the exact target will accept the server flag without opening a window, then prove Firefox's
profile/channel-specific command-forwarder belongs to the retained process. On Windows this is the hidden `Mozilla_*_RemoteWindow`, not a DevTools
listener or visible browser window. If Firefox is absent, launch it normally without the flag, wait for that forwarder, then request the listener
separately. Never cold-launch with the debugger flag.

A failed visible-window tool does not block attachment after exact handler and forwarder proofs. After the first real RDP connection, Firefox must report a
`navigator:browser` window with `gBrowser` and completed delayed startup; every task-affected window must be ready before mutation. Request the listener
at most once per retained instance. A just-started task-owned listener permits at most one new-client retry only when the first client never reached
TCP acceptance and all target, forwarder, ownership, OS, and log predicates still pass. After TCP acceptance, keep that socket through approval and
capability preflight. Only the reference's post-dispatch, restart, and cleanup branches may replace it. If any prerequisite fails, stop before the
operation and report the unverified boundary.

## Primary documentation

- [Mozilla: Remote Debugging Protocol](https://firefox-source-docs.mozilla.org/devtools/backend/protocol.html)
- [Mozilla: DevTools backward compatibility](https://firefox-source-docs.mozilla.org/devtools/backend/backward-compatibility.html)
- [Mozilla: command-line parameters](https://firefox-source-docs.mozilla.org/browser/CommandLineParameters.html)
- [Agent Skills specification](https://agentskills.io/specification)
- [GitHub CLI: `gh skill install`](https://cli.github.com/manual/gh_skill_install) and [`gh skill publish`](https://cli.github.com/manual/gh_skill_publish)
- [OpenAI: Skills in ChatGPT and Codex](https://help.openai.com/en/articles/20001066-skills-in-chatgpt) and [Plugins in Codex](https://help.openai.com/en/articles/20001256-plugins-in-codex/)
- [Anthropic: Claude Code skills](https://code.claude.com/docs/en/skills)
- [GitHub: Agent skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)
