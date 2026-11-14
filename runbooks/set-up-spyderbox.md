---
tags:
  - runbook
  - "#tailscale"
  - "#firewall"
---

# Set up spyderbox (a remote IDE desktop in an LXC)

Stand up an unprivileged Debian LXC with a lightweight desktop, an IDE, and an RDP server, reachable from a tablet over the tailnet only. Values in angle brackets are placeholders. See [spyderbox](../services/spyderbox.md) for the finished service.

## Step 1: Create the container
```
pct create <ctid> local:vztmpl/<debian-template>.tar.zst \
  --hostname <name> \
  --cores 2 --memory 4096 --swap 512 \
  --rootfs <storage>:25 \
  --unprivileged 1 \
  --features nesting=1 \
  --net0 name=eth0,bridge=vmbr0,ip=<ip>/24,gw=<gateway> \
  --nameserver <dns> \
  --onboot 1
```
> **Gotcha:** fill in the gateway at creation time. A missing gateway doesn't fail the container create, it just surfaces later as `Network is unreachable` the first time something needs outbound access, which looks unrelated to the real cause.

Add TUN passthrough for Tailscale to the container's config, then start it:
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

## Step 2: Base packages, a user, and Tailscale
```
apt update && apt install -y curl sudo git wget ca-certificates python3 python3-venv python3-pip
useradd -m -s /bin/bash <username>
ls /dev/net/tun
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --hostname=<name>
```
If `ls /dev/net/tun` fails, recheck the TUN lines from Step 1 and restart the container before going further.

## Step 3: Desktop, IDE, and RDP server
```
apt install -y --no-install-recommends xfce4 xfce4-terminal dbus-x11 fonts-dejavu
apt install -y spyder xrdp
adduser xrdp ssl-cert
```

## Step 4: RDP login and session
```
passwd <username>
echo "xfce4-session" > /home/<username>/.xsession
chown <username>:<username> /home/<username>/.xsession
systemctl enable --now xrdp
```
The RDP login is the account's Linux password, not a separate credential.

## Step 5: Connect and fix the display
Connect from an RDP client to the tailnet address on the RDP port, as that user. Set the client to fit to screen at 100% scale.

If the desktop is too small to read on a high-density screen, the client's own resolution and scaling options may not help. From a terminal inside the session, raise the desktop's own DPI setting and log out and back in. At a high DPI the application menu can become too large to use, worth a keyboard shortcut and a panel launcher for the IDE as a workaround.

## Step 6: Project repo and shared environment
```
mkdir -p ~/projects && cd ~/projects
git clone <repo-url>
cd <repo>
git config user.name "<name>"
git config user.email "<email>"
git config --global credential.helper store
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r requirements.txt <ide-kernel-package>
```
Make sure `.venv/` is in the repo's own ignore file. In the IDE's interpreter settings, point it at the new venv's Python and restart its console.

## Step 7: Keep the RDP server off the LAN
The RDP server listens on every address by default. A Proxmox firewall rule restricting the port to the tailnet's address range won't work: tailnet traffic arrives inside an encrypted tunnel, so the hypervisor never sees a tailnet-range source. Filter inside the container instead. Never start a new ruleset by flushing everything, since that wipes the rules Tailscale installs for itself; load it live and test before making it permanent.
```
apt install -y nftables
systemctl disable nftables
```
```
table inet <name>
delete table inet <name>
table inet <name> {
  chain input {
    type filter hook input priority 0; policy drop;
    iif lo accept
    ct state established,related accept
    iifname "tailscale0" accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept
    udp dport 41641 accept
  }
}
```
```
nft -c -f <file> && nft -f <file>
```
Test the RDP port from a LAN machine against both the LAN address and the tailnet address; the LAN one should be closed and the tailnet one open. Once confirmed, make it the container's permanent ruleset and enable the service.

## Step 8: Finish up
- Install the monitoring agent used elsewhere in the lab.
- `apt update && apt upgrade`, then reboot the container and confirm the RDP server and Tailscale both come back.
- Take a snapshot once everything works.
- Add the container to the regular backup job.

## Verification
- `ip route` shows a default route, and outbound connectivity works.
- The RDP server is active and reachable over the tailnet.
- The IDE opens and its interpreter resolves to the project venv.
- A push to the repo works with the stored credential.
- The RDP port is closed on the LAN address and open on the tailnet address, including after a reboot.
- If the IDE reports it's already running with no window, clear its lock file rather than killing a process match on its name, since that name can collide with an unrelated system process on a host whose hostname contains it.

## Related Notes
- [spyderbox](../services/spyderbox.md)
- [Proxmox firewall and tailnet traffic](../concepts/proxmox-firewall-and-tailnet-traffic.md)
- [Decommission an LXC](decommission-an-lxc.md)

## Tags
#runbook
