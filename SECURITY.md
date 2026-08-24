# Security policy

## Supported versions

Only the latest tagged release is supported with security fixes.

## Security boundaries

Classic Firefox RDP permits privileged browser-chrome evaluation and permanent XPI installation. Keep every endpoint on loopback (`127.0.0.1`); do not expose a debugger listener to a network.

Reports must omit profile paths and names, credentials, tokens, private-browsing data, and other private session details.

## Reporting a vulnerability

Submit vulnerabilities privately through [GitHub Security Advisories](https://github.com/117649/debugging-firefox/security/advisories/new).
