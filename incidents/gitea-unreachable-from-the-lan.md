---
date: 2026-10-21
status: Resolved
tags:
  - incident
  - "#dns"
  - "#gitea"
---

# A DNS record pointed a public name at the wrong backend

## What Happened
My Gitea instance's public name stopped resolving to anything usable from inside the LAN, while every other service kept working and the public route through the VPS tunnel was unaffected.

## Investigation
- A lookup from the LAN returned the Gitea host's own LAN address instead of the VPS's public address. A local DNS record on my LAN resolver was overriding it.
- A connection attempt to that address on the HTTPS port failed immediately with connection refused, meaning the host itself was up and reachable, just not listening there.
- A port check against the Gitea host showed its own web port open, SSH open, and nothing listening on the standard HTTP or HTTPS ports.
- Confirmed Gitea itself was healthy by reaching it directly on its own port: it answered normally.
- A process check on the Gitea host confirmed there was no reverse proxy running there at all. It was never meant to terminate TLS for its own public name, that happens one hop over, at the edge.

## Root Cause
The day before, I'd created a local DNS override for this name by mistake, meant for something else, and pointed it straight at the Gitea host's own address. Before that record existed, the name resolved normally and worked through the public edge and its tunnel. With the override in place, LAN clients were sent straight to the backend host instead, which only serves plain HTTP on its own internal port and has nothing listening on the ports a browser or git client expects for a public HTTPS name.

## Fix
- Deleted the stray local DNS record.
- Reloaded DNS and flushed local caches, then confirmed the name resolved to the public address again.
- LAN traffic goes back out to the public edge and back through the tunnel, same as any external client.

## Lessons for next time
- A local DNS override should never point a public hostname straight at a backend host, unless that host is actually set up to terminate TLS for that name itself. If it isn't, the override breaks exactly the ports a client expects to find.
- Testing a backend directly on its own internal port is fine for confirming the service itself is healthy, but it's not a valid login test if the application sets a cookie that requires a secure connection; a plain HTTP session can look like it's working and then silently fail to authenticate.
- When a service is reachable through more than one path (direct on the LAN, and through a public tunnel), a one-off local override for convenience can quietly become the only path that resolves, and it's easy to forget it exists until something trips over it.

## Related Notes
- [Gitea](../services/gitea.md)
- [Pi-hole](../services/pi-hole.md)

## Tags
#incident
