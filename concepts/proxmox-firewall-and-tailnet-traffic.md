# Proxmox firewall and tailnet traffic

## What It Is
The Proxmox firewall filters traffic crossing a guest's virtual network interface. When Tailscale runs inside a guest, tailnet traffic arrives wrapped in an encrypted WireGuard tunnel before it reaches that interface, so the Proxmox firewall can't filter it by tailnet address. Any rule that depends on a tailnet-range source has to live inside the guest instead.

## How It Works
- Two tailnet peers talk over an encrypted tunnel. On the wire that's UDP, usually to Tailscale's own port for a direct connection, or outbound HTTPS to a relay when no direct path works.
- What the hypervisor's firewall sees at the guest's interface is that encrypted UDP, with the peer's real address as the source. It never sees the RDP, SSH, or HTTP traffic riding inside it, and it never sees a tailnet-range source address.
- The guest's own Tailscale client decrypts the packet and delivers it on its tailnet interface. That happens inside the guest, after the packet has already passed the hypervisor's firewall.
- So a hypervisor-level rule like "allow this port only from the tailnet range" never matches tailnet traffic at all. With a default-drop policy, a rule like that would block legitimate tailnet traffic while doing nothing to stop LAN traffic, since LAN traffic never had to clear it in the first place.
- A rule inside the guest can tell the two paths apart, because there the tailnet traffic arrives on its own interface and LAN traffic arrives on the LAN interface. Accept everything on the tailnet interface, accept established connections, drop the rest.
- Tailscale installs its own firewall rules on the guest to keep itself working. A ruleset replacement that doesn't account for those (a bare "flush and rebuild" at the top of a new config, say) can delete them along with whatever else was there, which breaks the tailnet connection itself, not just the thing being locked down.
- An accept rule for established and related connections has to come early, or replies to the guest's own outbound traffic (a DNS query, for instance) get dropped by the same policy meant to block inbound LAN access.

## In My Setup
One of my LXCs runs a remote-desktop service that listens on every address by default. I wanted it reachable over the tailnet only, not the LAN.

My first plan was a Proxmox firewall rule allowing that port only from the tailnet's address range. It wouldn't have worked, for the reason above. I also wrote a first local firewall config that started by flushing the whole ruleset, which never actually loaded, and would have removed Tailscale's own rules along with everything else.

What I ended up with is a ruleset of its own inside the guest, with a default-drop input policy that explicitly accepts loopback, established connections, everything on the tailnet interface, ICMP, and Tailscale's own UDP port. After loading it, the port was closed on the LAN address and open on the tailnet address, and it stayed that way across a reboot.

How I check it: from a LAN machine, test the same port against both the LAN address and the tailnet address and confirm one is closed and the other open.

## Related Notes
- [Tailscale](../services/tailscale.md)
- [Tailscale tags and grants](tailscale-tags-and-grants.md)
- [spyderbox](../services/spyderbox.md)

## Tags
#concept #tailscale #firewall #networking
