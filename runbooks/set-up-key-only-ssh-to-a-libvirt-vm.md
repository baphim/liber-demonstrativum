---
tags:
  - runbook
  - "#libvirt"
  - "#ssh"
---

# Set up key-only SSH to a libvirt VM

SSH from the host to a libvirt VM using a dedicated key, with password login off, a fixed address from a DHCP reservation, and the SSH server stopped except when needed. I use it for pulling files off a VM with `scp`, since SPICE drag-and-drop only works from host to guest. I did not want a shared folder, because it would give the VM write access to part of the host.

## Step 1: Make SSH on demand
On the VM, disable and stop SSH. If `ssh.socket` shows enabled, disable and stop it too. The last command should print nothing.
```
sudo systemctl disable ssh
sudo systemctl stop ssh
systemctl is-enabled ssh ssh.socket
sudo ss -tln | grep ':22'
```

## Step 2: Find the MAC address
On the host, get the VM's MAC address and confirm the network's range before picking a reserved address.
```
sudo virsh domiflist <vm-name>
sudo virsh net-dumpxml default
```

## Step 3: Add a DHCP reservation
The `name` field is only the DHCP hostname. It does not rename anything in the guest, and the host will not resolve it unless its DNS points at the libvirt network.
```
sudo virsh net-update default add ip-dhcp-host "<host mac='<mac>' name='<hostname>' ip='<reserved-ip>'/>" --live --config
```

## Step 4: Pick up the new lease
Reboot the VM, then check it from the host.
```
sudo virsh net-dhcp-leases default
```

## Step 5: Generate a dedicated key
On the host, with a passphrase.
```
ssh-keygen -t ed25519 -f ~/.ssh/<vm-name> -C "<host> to <vm-name>"
```

## Step 6: Copy the key over
Start SSH on the VM from its console with `sudo systemctl start ssh`. Then, while password login is still on, copy the key and test it. Use the VM's username, not the host's.
```
ssh-copy-id -i ~/.ssh/<vm-name>.pub <vm-user>@<reserved-ip>
ssh -i ~/.ssh/<vm-name> <vm-user>@<reserved-ip>
```

## Step 7: Add an SSH alias
On the host, in `~/.ssh/config`, then test `ssh <vm-name>`. It asks to accept the host key the first time.
```
Host <vm-name>
    HostName <reserved-ip>
    User <vm-user>
    IdentityFile ~/.ssh/<vm-name>
    IdentitiesOnly yes
```

## Step 8: Turn off password login
On the VM, use a drop-in. Keep the working session open while doing this. `KbdInteractiveAuthentication` is included because PAM can still accept a password through keyboard-interactive. The `10-` prefix matters, since sshd uses the first value it reads.
```
sudo tee /etc/ssh/sshd_config.d/10-key-only.conf <<'CONF'
PasswordAuthentication no
KbdInteractiveAuthentication no
CONF
sudo sshd -t
sudo sshd -T | grep -iE 'passwordauthentication|kbdinteractiveauthentication'
sudo systemctl restart ssh
```
Both settings should show `no`. If one shows `yes`, another file is setting it earlier, and `grep -ri passwordauthentication /etc/ssh/` will find it.

## Step 9: Stop SSH again
When finished, run `sudo systemctl stop ssh` on the VM.

## Verification
- `ssh <vm-name>` asks for the key passphrase, not the account password.
- `ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <vm-user>@<reserved-ip>` fails with `Permission denied (publickey)`.
- With SSH stopped, `sudo ss -tln | grep ':22'` on the VM prints nothing.
- The host's lockdown rules do not block this, because the host starts the connection. See [Lock a libvirt VM off the LAN and tailnet](lock-a-libvirt-vm-off-the-lan.md).

## Related Notes
- [vm-lockdown](../services/vm-lockdown.md)
- [Take a baseline of a fresh VM](take-a-vm-baseline.md)

## Tags
#runbook
