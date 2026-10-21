---
tags:
  - runbook
  - "#caddy"
  - "#tailscale"
---

# Take a service off the public internet

Based on moving Vaultwarden from public access to tailnet-only. A password manager does not need to be exposed to the whole internet when every device that uses it is already on the tailnet. See [Vaultwarden](../services/vaultwarden.md) for the result.

## Starting state
A small VPS terminated public TLS with its own Caddy and forwarded to the internal [Caddy](../services/caddy.md), which already held a real Let's Encrypt cert via DNS-01 and proxied to the backend. A public DNS record pointed at the VPS.

## Order matters
Build and test the private path first. Remove the public path last. If the private path turns out to be broken and the public one is already gone, you are locked out of the service. I was working over SSH from a phone, so there was no easy recovery.

1. Inspect the public proxy block.
2. Inspect the internal proxy block and confirm the cert.
3. Fix local DNS.
4. Choose how remote devices reach the service.
5. Test DNS from a remote device, then test the real connection.
6. Cut over on the VPS.
7. Verify it is closed from outside.
8. Delete the public DNS record.

## Step 1: Inspect the VPS block
```
grep -n -B2 -A8 <service> /etc/caddy/Caddyfile
```
Know exactly what to remove before touching anything. Note the line range, and check that it is the only config for that service.

## Step 2: Inspect the internal Caddy and confirm the cert
```
grep -n -B2 -A8 <service> /etc/caddy/Caddyfile
caddy list-modules | grep -i dns
curl -sv --resolve <fqdn>:443:127.0.0.1 https://<fqdn> 2>&1 | grep -E 'issuer|subject|HTTP/'
```
Expect a real issuer, `HTTP/2 200`, and no `-k` needed. A real cert here means clients do not need to trust a private CA. That matters on mobile, where the Bitwarden Android app is picky about user-installed CAs.

## Step 3: Fix local DNS
List the records first, then replace the whole array in one command so the other records stay untouched.
```
pihole-FTL --config dns.hosts
pihole-FTL --config dns.hosts '["<ip> <name>", ...]'
dig +short <fqdn> @127.0.0.1
```
Review the existing records while you are in there. Mine had a typo in the service's own hostname, so the record never matched, and a stale address for the workstation.

After the first change, `dig` returned two answers. I checked every place a second record could live (`dns.hosts`, `dns.cnameRecords`, `misc.dnsmasq_lines`, `/etc/dnsmasq.d`, `/etc/pihole`, `/etc/hosts`) and found nothing. A full `dig` then showed a single authoritative answer. The likely cause is a stale cached answer. I did not prove it.

## Step 4: Choose how remote devices reach it
A record pointing at a LAN IP resolves fine on cellular and then fails to connect, because nothing advertises a route to the LAN. Two options:

1. Point the record at the container's tailnet IP. Simple. Every device needs Tailscale running, even at home.
2. Advertise the LAN subnet from a node and approve the route in the admin console. The LAN IP then works everywhere. More steps to maintain.

I chose option 1. Everything that uses the service is already on the tailnet, and it is one record instead of a routing change. Revisit option 2 if anyone else ever needs access without Tailscale.

## Step 5: Test before cutting over
MagicDNS was off tailnet-wide at the time (it is on now), so I did not assume the phone was asking the Pi-hole. Watch the query log while loading the page on a phone with Wi-Fi off and Tailscale on:
```
tail -f /var/log/pihole/pihole.log | grep -i <service>
```
The log should show the phone's tailnet address asking, and the answer coming from the local records. Then load the site and sync the client app. Doing this while the public route is still live means a failure costs nothing.

If it fails on Android, check "Use Tailscale DNS settings" in the Tailscale app, and set Private DNS to Off or Automatic in system settings. A fixed Private DNS provider overrides Tailscale's resolver.

## Step 6: Cut over on the VPS
Preview first, so the deletion range is only the comment and the block. Then back up, delete, validate, reload.
```
sudo sed -n '1,11p' /etc/caddy/Caddyfile
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak
sudo sed -i '1,11d' /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
systemctl is-active caddy
```
The backup gives a one-command rollback. Validate matters because that Caddy serves other public sites too, so a bad config would take them down as well.

## Step 7: Verify it is closed
From the VPS itself:
```
curl -sS -o /dev/null -w '%{http_code}\n' --resolve <fqdn>:443:127.0.0.1 https://<fqdn>
```
Expect a TLS failure (`tlsv1 alert internal error`). Caddy no longer has a site for the name, so it refuses the handshake. From outside, a phone on cellular with Tailscale off should not load the page, and should load it again with Tailscale on.

Also confirm the internal path is still healthy: HTTPS returns 200, and plain HTTP returns a 308 redirect to HTTPS.

Right after the cutover, the phone browser loaded the bare hostname but not with `https://`. It resolved itself. A stale cache holding the old public IP is the likely cause. If it returns, check Chrome's "Use secure DNS" setting, which bypasses the Pi-hole, and toggle Tailscale or airplane mode to flush the cache.

## Step 8: Delete the public DNS record
With the VPS block gone, the record only advertises the VPS address for a name it no longer serves. Deleting it does not affect certificate renewal, since the internal Caddy uses DNS-01 through the registrar's API, which only needs TXT record access.

## Verification
- `dig +short <fqdn> @1.1.1.1` returns nothing once caches expire.
- The service works from a device on the tailnet, on and off the LAN.
- It does not load from a device that is off the tailnet.
- The VPS Caddyfile has no block for the service, and the other public sites still work.

## Cleanup
- Delete the Caddyfile backup on the VPS when you are comfortable.
- Add a DHCP reservation for any host that has a local DNS record. The workstation's address had drifted, which is how its record went stale.

## Trade-off to remember
A device without Tailscale cannot reach the service anywhere, including at home. The Bitwarden clients keep an offline vault cache, so passwords can still be read in a pinch.

## Related Notes
- [Vaultwarden](../services/vaultwarden.md)
- [Caddy](../services/caddy.md)
- [Pi-hole](../services/pi-hole.md)
- [DNS-01 vs HTTP-01](../concepts/dns-01-vs-http-01.md)
