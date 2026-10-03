---
tags:
  - concept
  - "#tls"
  - "#networking"
---

# Reverse proxy TLS termination

Caddy holds the actual CA-issued cert because that is what the browser is communicating with. When I navigate to `vaultwarden.baphim.xyz` via a browser, it makes a TLS connection with my Caddy instance, which acts as a proxy to Vaultwarden. The Vaultwarden backend is never in direct HTTPS connection with a browser.

This pattern is called **TLS termination**: the encrypted connection "terminates" at Caddy, the browser's TLS handshake ends there. Caddy then opens a second, separate connection to the actual backend over plain HTTP, not HTTPS, and forwards the decrypted request. The backend doesn't need its own certificate because it's never the thing a browser connects to directly, it only ever receives already-decrypted traffic from Caddy over the trusted LAN.

## Why this matters here
This is exactly how every proxied service in this lab works, [Caddy](../services/caddy.md) is the single TLS endpoint for every public subdomain, while each backend runs plain HTTP and is only ever reached over the LAN, never directly from a browser.

## Related Notes
- [Caddy](../services/caddy.md)
- [Vaultwarden](../services/vaultwarden.md)

## Tags
#concept
