---
tags:
  - concept
  - "#tls"
  - "#dns"
---

# DNS-01 vs HTTP-01 (ACME challenges)

To obtain a certificate from a Certificate Authority (CA) that proves you own a domain, a challenge is issued to prove ownership.

HTTP-01: the CA gives you a token, you put it at a URL on your web server, and the CA connects to that URL over port 80 to check it. That connection happens over the public internet.

DNS-01: the CA gives you a token, and instead of hosting it publicly, you create a DNS TXT record at your registrar containing it. The CA checks it with a DNS lookup instead of connecting to your server. This needs API access to your DNS provider, and it means the CA never has to reach your server directly.

## Why this matters here
This is why [Caddy](../services/caddy.md) can issue real, publicly-trusted certificates (through the registrar's DNS-01 API) for services that are LAN or tailnet-only, the way Grafana would be if it used this instead of the self-signed internal CA. The cert type and whether a service is actually reachable from the internet are two separate facts. A real cert doesn't mean a service is public, and a self-signed one doesn't mean it isn't. That distinction mattered when I was confirming Vaultwarden's public exposure: the DNS-01 method alone couldn't answer it, I had to test separately.

## Related Notes
- [Caddy](../services/caddy.md)
- [Vaultwarden](../services/vaultwarden.md)

## Tags
#concept
