---
tags:
  - concept
  - "#tls"
  - "#dns"
---

# DNS-01 vs HTTP-01 (ACME challenges)

To obtain a certificate from a Certificate Authority (CA) that proves you own a domain, a challenge is issued to prove ownership.

**HTTP-01** challenges occur when the CA gives you a token and you add it to a URL on your web server. The CA then connects to that URL over port 80 to confirm its validity. This connection happens over the public internet.

**DNS-01** challenges happen when the CA issues a token and, instead of publicly hosting it, you create a DNS TXT record with your registrar containing that token. The CA is then able to verify the token via a DNS lookup rather than by actually connecting to your server. This requires API access to your DNS provider, which allows the certificate to be authenticated without the CA ever needing to reach your server directly.

## Why this matters here
This is why [Caddy](../services/caddy.md) can issue real, publicly-trusted certificates (via the registrar's DNS-01 API) for services that are LAN or tailnet-only, like Grafana would be if it used this method instead of the self-signed internal CA. The cert type and whether a service is actually reachable from the internet are two independent facts, a real cert doesn't mean a service is public, and a self-signed one doesn't mean it isn't. This came up directly when confirming Vaultwarden's public exposure, the DNS-01 method alone couldn't answer that question, it had to be tested separately.

## Related Notes
- [Caddy](../services/caddy.md)
- [Vaultwarden](../services/vaultwarden.md)

## Tags
#concept
