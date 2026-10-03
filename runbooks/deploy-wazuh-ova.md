---
tags:
  - runbook
  - "#wazuh"
---

# Deploy a Wazuh manager from OVA

Based on the actual build of the Wazuh server in this lab. See [Wazuh](../services/wazuh.md) for the resulting service note.

## Overview
Wazuh ships a prebuilt OVA rather than requiring a manual OS install, significantly less error-prone than building a VM from scratch. The OVA is Amazon Linux 2023-based, which has real implications (FIPS mode, `dnf` not `apt`, `systemd-networkd` not the usual Debian networking) covered below.

## Step 1: Get and verify the OVA
```
curl "https://packages.wazuh.com/4.x/vm/wazuh-<version>.ova" --output wazuh-<version>-1.ova
curl "https://packages.wazuh.com/4.x/checksums/wazuh/<version>/wazuh-<version>.ova.sha512" --output wazuh-<version>.ova.sha512
sha512sum -c wazuh-<version>.ova.sha512
```
Gotcha: the `.sha512` file expects a specific filename/path (note the `-1`). Rename and move the file to match before verifying, or the checksum check fails with a misleading "file could not be read" rather than a clear filename mismatch error.

## Step 2: Extract and check specs
```
mv wazuh-<version>.ova /tmp/wazuh-<version>-1.ova
cd /tmp
tar -xvf wazuh-<version>-1.ova
cat wazuh-<version>.ovf | grep -i "cpu\|memory"
```
Wazuh's documented recommendation is 4 vCPU / 8192MB RAM. If the host can't afford 4 cores, 2 works but is untested under real alert load, decide deliberately rather than by default.

## Step 3: Create the VM and import the disk
```
qm create <vmid> --name wazuh-server --cores <N> --sockets 1 --cpu x86-64-v2-AES --memory 8192 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single
qm importdisk <vmid> wazuh-<version>-disk-1.vmdk <storage-pool>
```
Choosing the storage pool: prefer one with snapshot support and enough free space margin, the vulnerability-detector module can consume significant disk over time (see the disk-bloat runbook).

In the Proxmox web UI after import: set the imported "Unused Disk 0" Bus/Device to SCSI, set boot order to disable `net0` and enable the new `scsi0` disk, enable "Start at boot".

## Step 4: First boot and credential rotation
Boot the VM. Default credentials are documented publicly by Wazuh, change them immediately, then run the bundled password tool to rotate every internal user:
```
sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh --change all
```
Run the same tool against the API users too. Restart the dashboard afterward:
```
sudo systemctl restart wazuh-dashboard
sudo systemctl status wazuh-dashboard
```

## Step 5: SSH key auth (FIPS gotcha)
The OVA's AL2023 base ships in **FIPS mode**, which disallows Ed25519 keys. If `ssh-copy-id` with an existing Ed25519 key results in endless password prompts despite the key being present, this is why. Generate an RSA key instead:
```
ssh-keygen -t rsa -b 4096 -C "<purpose>" -f ~/.ssh/id_rsa_wazuh
ssh-copy-id -i ~/.ssh/id_rsa_wazuh.pub wazuh-user@<vm-ip>
```

## Step 6: Static IP (systemd-networkd, not Debian-style)
AL2023 uses `systemd-networkd`, not `ifupdown`/NetworkManager. Edit the network file directly:
```
sudo vi /etc/systemd/network/20-eth0.network
```
Remove DHCP, set a static address, gateway, and DNS, then:
```
sudo systemctl restart systemd-networkd
```

## Step 7: Update, and consider the release upgrade prompt
```
sudo dnf update -y
```
If it reports a newer AL2023 release available and there's no uptime constraint, upgrading is reasonable for a security-facing service:
```
sudo dnf upgrade --releasever=<version> -y
```

## Step 8: Tailscale (optional, for reaching it without public exposure)
```
sudo curl -o /etc/yum.repos.d/tailscale.repo https://pkgs.tailscale.com/stable/amazon-linux/2023/tailscale.repo
sudo dnf install tailscale -y
sudo systemctl enable --now tailscaled
sudo tailscale up
```

## Step 9: Firewall (firewalld, rich rules, default DROP)
```
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --zone=public --remove-service=ssh   # remove the default open-to-anywhere ssh rule first
```
Add one rich rule per source/port pair needed, rather than a blanket allow:
```
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="<CIDR>" port port="<PORT>" protocol="tcp" accept'
```
Standard Wazuh manager ports: 22 (SSH), 443 (dashboard), 1514 (agent event data), 1515 (agent enrollment), 55000 (API). Add each for every source that needs it.
```
sudo firewall-cmd --permanent --zone=public --set-target=DROP
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

## Step 10: DNS
Create a local DNS record pointing a friendly hostname at the manager's LAN IP, rather than exposing it publicly through a reverse proxy.

## Step 11: Deploy agents to existing hosts
For a Debian/Ubuntu LXC or VM:
```
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo WAZUH_MANAGER='<manager-ip>' apt-get install wazuh-agent -y
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```
Use the manager's LAN IP for hosts on the same LAN, or its tailnet IP for hosts reached only over Tailscale.

For Arch-based systems (no official build, AUR only):
```
paru -S wazuh-agent
```
Then edit `/var/ossec/etc/ossec.conf` directly, set the manager address, and register the agent's key:
```
sudo /var/ossec/bin/agent-auth -m <manager-ip> -A <agent-name>
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent
```

## Notes / gotchas
- The 4 vCPU vs 2 vCPU tradeoff is a real, unverified-under-load decision, not a default to accept blindly.
- The disk pool choice matters, see [Fix Wazuh vulnerability-detector disk bloat](fix-wazuh-disk-bloat.md) for what happens when the vulnerability-detector module fills the disk.
- Always use `wazuh-control`, not `systemctl`, to stop/start wazuh-manager day to day, see the disk-bloat runbook for why this matters.
- Email alerting requires a separate OS-level mail relay setup, Wazuh doesn't handle SMTP auth itself. Not covered here.

## Related Notes
- [Wazuh](../services/wazuh.md)
- [Fix Wazuh vulnerability-detector disk bloat](fix-wazuh-disk-bloat.md)
