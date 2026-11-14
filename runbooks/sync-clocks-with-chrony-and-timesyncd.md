---
tags:
  - runbook
  - "#ntp"
---

# Sync clocks across the lab with chrony and timesyncd

Make one host the lab's own time server, syncing from public NTP pools, and point every other machine at it first with the public pools kept as a fallback. A consistent clock matters for log correlation, cluster software, and anything that checks certificate validity windows.

## Step 1: Pick the time server and confirm it's healthy
On the chosen host:
```
chronyc tracking
chronyc sources -v
```
It should already be synced to a public pool with a low stratum number and a small offset before it serves anyone else.

## Step 2: Serve the LAN
Check whether the config uses a drop-in directory before editing the main file:
```
grep -vE '^\s*(#|$)' /etc/chrony/chrony.conf
```
If it includes a `confdir`, add a drop-in instead of touching the main file:
```
echo 'allow <lan-subnet>' > /etc/chrony/conf.d/serve-lan.conf
systemctl restart chrony
```
Confirm it's listening and still synced:
```
ss -ulnp | grep :123
chronyc tracking
```
If a host firewall is active, it needs a rule for UDP 123 inbound; if it's disabled, there's nothing to add.

## Step 3: Point other Linux hosts at it
On a client already running chrony, with `prefer` so it favors the local server but keeps public pools as a fallback:
```
echo 'server <local-server-ip> iburst prefer' > /etc/chrony/conf.d/local-server.conf
systemctl restart chrony
```
> **Gotcha:** some distributions' chrony packages refuse a bare service restart after a config change, if a startup script creates a symlink that a restart's own setup step then tries to recreate and fails on. If a restart leaves the service down, check its status for a failing `ExecStartPre` line, remove the stale symlink it's complaining about, and start (not restart) the service. For routine config changes afterward, reload sources instead of restarting.

After a short wait, `chronyc sources -v` should show the local server as the selected source, with public pools listed as available backups.

On a host using `systemd-timesyncd` instead of chrony:
```
mkdir -p /etc/systemd/timesyncd.conf.d
tee /etc/systemd/timesyncd.conf.d/local.conf <<CONF
[Time]
NTP=<local-server-ip>
FallbackNTP=<distro-ntp-pool>
CONF
systemctl restart systemd-timesyncd
timedatectl timesync-status
```
> **Gotcha:** the stratum shown in `timesync-status` is the server's stratum, not the client's own. A client one hop below a stratum-2 server will itself show stratum 2 in that status output, not 3.

## Step 4: Containers that share the host's kernel
A container that shares its host's kernel clock (an LXC, for instance) has nothing to configure and no client to install. Confirm it's actually on the same clock as the host with a direct time check rather than assuming:
```
date '+%F %T %Z'
```

## Step 5: A host that won't accept the host's own clock source scheme
A VM with its own guest OS and its own NTP client is set up the same way as any other standalone Linux host in Step 3, pointed at the local server with the OS's native public pool as a fallback. Keep a security-sensitive host's logging timezone as it already is (UTC is the usual convention for a log-correlating service) rather than changing it to match the rest of the fleet; the timestamp a central logging or alerting pipeline uses typically isn't the local system clock's timezone anyway, so changing it there buys nothing and adds a second thing to remember.

## Step 6: A Windows host
In an elevated PowerShell:
```
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```
makes the hardware clock UTC, matching a Linux dual-boot on the same machine. Then:
```
net start w32time
w32tm /config /manualpeerlist:"<local-server-ip>" /syncfromflags:manual /update
w32tm /resync
w32tm /query /status
```
> **Gotcha:** if the Windows Time service isn't already running, `w32tm /config` silently has nothing to apply. Start the service first and check `/query /status` afterward to confirm it actually took.

Set the system timezone correctly for wherever the machine actually is:
```
Set-TimeZone -Id "<timezone-id>"
```
A wall-clock time that looks several hours off right after a successful sync is almost always a timezone setting, not a sync failure. Check the timezone before chasing an NTP problem that isn't there.

## Step 7: Containers whose timezone differs from the host
If some containers ended up on a different timezone than the rest (common after building them at different times from different templates), check for any cron jobs or timers first, since changing a container's timezone shifts when they fire:
```
crontab -l
systemctl list-timers
```
Then set it:
```
timedatectl set-timezone <zone> 2>/dev/null || { ln -sf /usr/share/zoneinfo/<zone> /etc/localtime; echo <zone> > /etc/timezone; }
```
> **Gotcha:** a running daemon inside the container typically reads the timezone once at startup, so anything already running keeps logging in the old timezone until it restarts. Reboot the container (or restart the daemon) afterward if the new timezone needs to take effect right away.

## Verification
- The time server itself stays synced to public pools and only serves the configured subnet.
- Every client shows the local server as its selected (or preferred) source, with public pools reachable as a fallback.
- `date` on a sample of containers shows the same wall-clock time as the host.
- A security-sensitive logging host stays on the timezone its downstream tooling expects, deliberately, not by oversight.
- A dual-boot machine's two operating systems agree on the hardware clock.

## Related Notes
- [Proxmox](../services/proxmox.md)

## Tags
#runbook
