# Tailscale tags and grants

## What It Is
A tag is a label on a device (like `tag:pve`) that stands in for "this is a server, not a person's device". A grant is a rule in the tailnet policy file that says which sources may reach which destinations on which ports. Together they let me say "the Proxmox nodes can reach one port on one machine and nothing else", where the old default policy said everything could reach everything.

## How It Works
- A tagged device is owned by the tag, not by a user. It does not lose access if my user account changes, and its key can be set not to expire, which matters for a server I forget about.
- The policy file defines who may apply a tag, and grants use the tag as a source or destination.
- Tailscale's default policy allows every device to talk to every other device on every port. The first thing I changed was replacing that with explicit grants. A new rule file is easy to get wrong, so the policy has a `tests` section, which checks that expected connections pass and expected ones fail before the file can be saved.
- Two flags on the device side matter for a server:
  - `--shields-up` blocks incoming connections from the rest of the tailnet to this node. It can still make outgoing connections.
  - `--accept-dns=false` stops Tailscale rewriting `/etc/resolv.conf` on the device (see [Tailnet DNS override](tailnet-dns-override.md)).
- The node announces its tag with `--advertise-tags=tag:pve` when it joins.
- The policy language has an older `acls` syntax and a newer `grants` syntax. My first draft used `acls` and did not match the file I had, so I switched to `grants`.

## In My Setup
`tag:pve` covers both Proxmox nodes. Members of my tailnet reach each other and exit nodes as before. `tag:pve` may reach only the VPS, on the one port the [QDevice](../services/qdevice.md) uses. It also had SSH for the QDevice setup, and I removed that once the setup was done. Both nodes run `--shields-up --accept-dns=false`, with key expiry off.

The reason is that the Proxmox hosts are the most important machines in the lab. They are only on the tailnet at all because the VPS is the QDevice. I want a compromised phone or laptop on the tailnet to have no route to them. After saving the policy, I turned wifi off on my phone and checked that two tailnet-only services still loaded, so I had not broken my own access.

## Related Notes
- [Tailscale](../services/tailscale.md)
- [QDevice](../services/qdevice.md)
- [Tailnet DNS override](tailnet-dns-override.md)
