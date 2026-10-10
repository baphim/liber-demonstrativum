---
tags:
  - runbook
  - "#kvm"
---

# Build a Windows 11 VM on KVM

Windows 11 VM on KVM/libvirt with bridged LAN networking, so it can reach the rest of the lab. Written for an Arch-based workstation with btrfs and snapper, Docker, and firewalld. I chose KVM over VirtualBox and VMware because it lives in the kernel and does not break on Arch kernel updates. It is also the same stack Proxmox uses, so what I learn here carries over. Gotchas are marked at the step where they bite.

## Step 1: Check the hardware
```
lscpu | grep -i virtualization
ls -l /dev/kvm
nmcli device status
```
Need AMD-V or VT-x, a `/dev/kvm` device, and a wired NIC. Bridging does not work over Wi-Fi in the normal way.

> **Gotcha:** Size the VM for the host. On a 16GB workstation, 6GB for the VM leaves room for the desktop.

## Step 2: Create a non-CoW subvolume for the images
```
sudo mkdir -p /mnt/btrfs-top
sudo mount -o subvolid=5 /dev/<btrfs-device> /mnt/btrfs-top
sudo btrfs subvolume create /mnt/btrfs-top/@libvirt-images
sudo umount /mnt/btrfs-top
sudo mkdir -p /var/lib/libvirt/images
```
Add to `/etc/fstab`, copying the UUID from the existing root line:
```
UUID=<same-uuid-as-root>  /var/lib/libvirt/images  btrfs  rw,noatime,subvol=/@libvirt-images  0 0
```
```
sudo systemctl daemon-reload
sudo mount /var/lib/libvirt/images
sudo chattr +C /var/lib/libvirt/images
```

> **Gotcha:** Set `chattr +C` while the directory is still empty. It only applies to files created afterward. A VM disk on a copy-on-write subvolume fragments badly, and snapper would snapshot it over and over. A separate subvolume also keeps it out of snapper's snapshots of `/`.

`findmnt` still lists `compress=zstd:3` for this mount. That is the filesystem-wide option, and `+C` overrides it.

## Step 3: Install and enable libvirt
```
sudo pacman -S --needed qemu-desktop libvirt virt-manager dnsmasq edk2-ovmf swtpm
sudo usermod -aG libvirt <user>
sudo systemctl enable --now libvirtd.service
```
Log out and back in so the group applies, then:
```
virt-host-validate qemu
```
Two warnings are harmless here: cgroup `devices` (cgroup v2, and the check runs unprivileged) and SEV/TDX (server CPUs only).

## Step 4: Stop Docker from blocking bridged traffic
Docker sets the kernel FORWARD policy to DROP, so traffic through the bridge disappears with no error. On Docker 28 or newer, put this in `/etc/docker/daemon.json`:
```
{
    "ip-forward-no-drop": true
}
```
```
sudo systemctl restart docker
```

> **Gotcha:** The option only takes effect when the daemon starts, so the restart is required. On an older Docker, add an ACCEPT rule for the bridge in the `DOCKER-USER` chain instead.

## Step 5: Allow forwarding in firewalld
```
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --query-forward
```
If forwarding says `no`:
```
sudo firewall-cmd --permanent --zone=public --add-forward
sudo firewall-cmd --reload
```
Use the zone your NIC is actually in. The bridge goes in the same zone.

## Step 6: Build the bridge
Find out whether the host uses DHCP or a static address, and note the NIC's MAC:
```
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns connection show "<wired-profile>"
ip link show <nic>
```
```
sudo nmcli connection add type bridge ifname br0 con-name br0 \
  bridge.stp no \
  bridge.mac-address <nic-mac> \
  connection.zone public \
  connection.autoconnect-ports true \
  ipv4.method auto ipv6.method auto
sudo nmcli connection add type ethernet ifname <nic> con-name br0-port-<nic> \
  controller br0 port-type bridge
sudo nmcli connection modify "<wired-profile>" connection.autoconnect no
sudo nmcli connection down "<wired-profile>"; sudo nmcli connection up br0
```

> **Gotcha:** `autoconnect-ports` takes `true`, not `yes`. nmcli rejects the whole command and creates nothing, so just rerun it.

Cloning the NIC's MAC keeps any DHCP reservation working. STP is off because there is a single bridge, and it would only add a delay when the link comes up. Run the switch-over as one line, since the network drops for a few seconds.

Rollback: `sudo nmcli connection down br0; sudo nmcli connection up "<wired-profile>"`.

## Step 7: Get the ISOs
- Windows 11 Enterprise evaluation, from the Microsoft Evaluation Center (90 days).
- The regular Windows 11 ISO from Microsoft, for when the evaluation expires.
- `virtio-win.iso` from the Fedora project.

> **Gotcha:** The evaluation download requires a registration form (company name, size, job role). Windows Server's evaluation uses the same form, so it does not avoid it. The regular ISO needs no registration.

Copy the ISOs you need now into the images directory:
```
sudo cp <windows-eval>.iso /var/lib/libvirt/images/
sudo cp virtio-win-*.iso /var/lib/libvirt/images/
```

> **Gotcha:** QEMU runs as a separate user and cannot read files in your home directory. Keep ISOs somewhere it can read, or loosen access deliberately.

## Step 8: Start virt-manager
```
/usr/bin/python3 /usr/bin/virt-manager &
```

> **Gotcha:** With the conda `base` environment active, `virt-manager` fails with `ModuleNotFoundError: No module named 'gi'`. Its launcher picks the first `python3` in PATH, and Anaconda's does not have `gi`. Launch with the system Python, run `conda deactivate` first, or start it from the desktop menu.

## Step 9: Create the VM
New VM, local install media, the Windows ISO.
- RAM 6144 MB, 4 vCPUs, 100 GiB disk (sparse qcow2, so the size is a ceiling. Growing later is easy, shrinking is not)
- Pick a descriptive name. The disk file takes the VM's name, and renaming later does not rename the file.
- Network selection: bridge device `br0`
- Tick **Customize configuration before install**

In the customize screen:
- Chipset Q35, firmware UEFI with `secboot` in the name
- Copy host CPU configuration
- Disk bus VirtIO, discard mode `unmap`
- NIC model virtio
- Add a TPM: emulated, CRB, version 2.0
- Add a CDROM with the virtio-win ISO
- Windows ISO first in the boot order

> **Gotcha:** "Press any key to boot from CD or DVD" only waits a few seconds. Press a key right away. Do not press one after the first reboot, so it boots from the new disk.

## Step 10: Install Windows
At "Where do you want to install Windows?" the disk list is empty. Choose Load driver, Browse, and use the virtio-win CD:
- `viostor\w11\amd64` (Red Hat VirtIO SCSI controller)
- `NetKVM\w11\amd64` (Red Hat VirtIO Ethernet Adapter)

If nothing is listed, untick "Hide drivers that aren't compatible". Install to the unallocated space and do not make partitions by hand.

> **Gotcha:** The first-run screen demands a work or school account on Enterprise. Click **Sign-in options**, then **Domain join instead**. It only creates a local account. If that option is missing, press Shift+F10 and run `start ms-cxh:localonly`.

## Step 11: Finish inside Windows
Run `virtio-win-guest-tools.exe` from the virtio-win CD and reboot. Then rename the computer from an **elevated** PowerShell:
```
Rename-Computer -NewName <name> -Restart
```

> **Gotcha:** A non-elevated shell gives "Access is denied". The prompt being in `C:\WINDOWS\system32` does not prove it is elevated, because a normal PowerShell can start there too. Check that the title bar says "Administrator".

## Step 12: Detach the install media and snapshot
Shut down from inside Windows, not with force off. Remove the Windows ISO CDROM in the VM's hardware details. Then View, Snapshots, and add one named `clean-install`.

These are internal qcow2 snapshots, so deleting the disk file deletes them too. They are a rollback point, not a backup. Keep scripts and notes outside the VM.

## Verification
- `virt-host-validate qemu` passes, apart from the two harmless warnings.
- `lsattr -d /var/lib/libvirt/images` shows the `C` flag.
- `bridge link` shows the NIC with `master br0`.
- Inside the VM:
  ```
  Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -like '<lan-prefix>.*' }
  Test-Connection <lan-host> -Count 2
  ```
  Expect a LAN address and replies from another host on the network.

## Why it exists
I wanted a Windows box for Windows-only apps and PowerShell practice that can also reach Kali, Metasploitable and Wazuh on the LAN. Bridged networking is what makes that possible. A Windows 11 dual boot stays on the workstation for software that refuses to run in a VM.

## Related Notes
- [Infrastructure](../infrastructure.md)
