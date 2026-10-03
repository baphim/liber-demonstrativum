---
date: 2026-09-19
status: Resolved
tags:
  - incident
  - "#caddy"
---

# Caddy LXC out of disk space

## What Happened
Logging into the Caddy LXC surfaced an `apt_pkg.Error: E:Write error - write (28: No space left on device)` traceback from Ubuntu's release-upgrade checker. Unrelated to the task at hand, just surfaced while working on something else.

## Investigation
- `df -h /` confirmed the LXC's disk was at 100% capacity.
- `du -h -d 1 /` narrowed it to three directories, roughly evenly split between system packages, variable data, and a user home directory.
- The variable data split roughly evenly between apt cache and logs, with the systemd journal making up most of the log share.
- The home directory turned out to be almost entirely a Go module cache, versus a tiny fraction in actual compiled binaries.

## Root Cause
This Caddy instance isn't stock Caddy, it was built with `xcaddy` to bundle a DNS-01 plugin for the domain registrar's API, needed for getting real certificates on LAN-only services. Building with `xcaddy` requires a full Go toolchain, and its module cache alone was nearly a fifth of this LXC's entire disk. That, plus a never-trimmed apt cache and systemd journal, ate the rest. The disk was sized for a plain reverse proxy, not for also compiling one.

## Fix
```
go clean -modcache
apt-get clean
journalctl --vacuum-size=100M
```
Recovered roughly 1.4GB. None of this touched the running Caddy config or the built `xcaddy` binary, all three are cache/log data that regenerates on demand.

## Related Notes
- [Caddy](../services/caddy.md)

## Tags
#incident
