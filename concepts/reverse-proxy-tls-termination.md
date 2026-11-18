---
tags:
  - concept
  - "#tls"
  - "#networking"
---

# Reverse proxy TLS termination

Caddy holds the actual CA-issued cert, because that's what the browser is talking to. When I go to `vaultwarden.baphim.xyz` in a browser, it makes a TLS connection to Caddy, which proxies to Vaultwarden. The Vaultwarden backend never has a direct HTTPS connection to a browser.

This pattern is called TLS termination: the encrypted connection ends at Caddy, the browser's TLS handshake stops there. Caddy then opens a second, separate connection to the real backend over plain HTTP, not HTTPS, and forwards the decrypted request. The backend doesn't need its own certificate, since it's never what a browser connects to directly. It only ever receives traffic that Caddy has already decrypted, over the trusted LAN.

## Why this matters here
This is how every proxied service in the lab works. [Caddy](../services/caddy.md) is the single TLS endpoint for every public subdomain, and each backend runs plain HTTP, reachable only over the LAN and never directly from a browser.

## Related Notes
- [Caddy](../services/caddy.md)
- [Vaultwarden](../services/vaultwarden.md)

## Tags
#concept
