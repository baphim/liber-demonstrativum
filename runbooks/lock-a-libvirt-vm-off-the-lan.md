---
tags:
  - runbook
  - "#libvirt"
  - "#firewall"
---

# Lock a libvirt VM off the LAN and tailnet

Keeps a VM on the libvirt `default` NAT network off the LAN, the tailnet and the host's services, while leaving internet access. libvirt NAT only blocks inbound connections from the LAN. The VM can still start connections out to anything the host can reach, including tailnet devices through the host's Tailscale. Use this for any VM that runs offensive tools or untrusted samples. Background: [NAT is not isolation](../concepts/nat-is-not-isolation.md). Result: [vm-lockdown](../services/vm-lockdown.md).

## Step 1: Test what the VM can reach
Before changing anything, try a LAN device, a tailnet device and the host's ports on the NAT gateway, from inside the VM.
```
ip route | grep default
ping -c 2 <lan-device-ip>
curl -sI -m 5 http://<lan-device-ip>:<port> | head -n 1
ping -c 2 <tailnet-device-ip>
for p in <port-list>; do timeout 3 bash -c "</dev/tcp/<gateway-ip>/$p" 2>/dev/null && echo "$p open" || echo "$p closed or filtered"; done
```

## Step 2: See what the host exposes
On the host, list what is listening on all interfaces and which firewall is active.
```
sudo ss -tulpn | grep -vE '127\.0\.0\.|\[::1\]'
systemctl is-active ufw firewalld nftables
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --info-zone=libvirt
```

## Step 3: Trim the libvirt firewalld zone
Remove services the VM does not need, runtime and permanent. Skip `--reload`, so libvirt's binding of the bridge to the zone is not disturbed.
```
sudo firewall-cmd --zone=libvirt --remove-service=ssh
sudo firewall-cmd --permanent --zone=libvirt --remove-service=ssh
sudo firewall-cmd --zone=libvirt --remove-service=tftp
sudo firewall-cmd --permanent --zone=libvirt --remove-service=tftp
sudo firewall-cmd --info-zone=libvirt | grep services
```

## Step 4: Write the lockdown rule in its own nftables table
A reject verdict is final no matter what other chains accept, so this works even though libvirt and firewalld add their own rules. The first two lines make the file safe to load repeatedly.
```
sudo mkdir -p /etc/nftables.d
sudo tee /etc/nftables.d/vm-lockdown.nft <<'RULES'
table inet vm_lockdown
delete table inet vm_lockdown
table inet vm_lockdown {
    chain forward {
        type filter hook forward priority -10; policy accept;
        iifname "virbr0" ip daddr { 10.0.0.0/8, 100.64.0.0/10, 169.254.0.0/16, 172.16.0.0/12, 192.168.0.0/16 } counter reject
    }
}
RULES
sudo nft -f /etc/nftables.d/vm-lockdown.nft
```
The ranges are the private blocks, the tailnet range (`100.64.0.0/10`), link-local, and the `172.16.0.0/12` block that also covers Docker's bridges.

## Step 5: Repeat the tests
Run the Step 1 tests again before making the rule persistent.

## Step 6: Make it survive reboots
Use a oneshot unit. Do not enable the distro's `nftables.service` for this, since its default config starts with `flush ruleset`.
```
sudo tee /etc/systemd/system/vm-lockdown.service <<'UNIT'
[Unit]
Description=Keep VMs on the libvirt default network off the LAN and tailnet
After=network.target libvirtd.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/nft -f /etc/nftables.d/vm-lockdown.nft
ExecStop=/usr/bin/nft delete table inet vm_lockdown

[Install]
WantedBy=multi-user.target
UNIT
sudo systemctl daemon-reload
sudo systemctl enable --now vm-lockdown.service
```

## Verification
- From the VM, the host ports are closed, the LAN and tailnet pings fail, and an internet ping and `apt update` still work.
- On the host, `sudo nft list table inet vm_lockdown` shows the reject counter going up for the blocked attempts.
- Reboot the host, start the VM, and repeat the LAN ping. It should still fail.
- The VM can still resolve internal names through the host's DNS, but it cannot connect to what they point to. The rule applies to every VM on the bridge.

## Related Notes
- [vm-lockdown](../services/vm-lockdown.md)
- [Kali Purple VM exposed to the LAN and tailnet](../incidents/kali-purple-vm-lan-tailnet-exposure.md)
- [Set up key-only SSH to a libvirt VM](set-up-key-only-ssh-to-a-libvirt-vm.md)

## Tags
#runbook
