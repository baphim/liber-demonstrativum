# Infrastructure

Everything in the homelab runs on a single Proxmox host, hypervisor on bare metal, LXCs and VMs for each service. The exception is two VMs on my workstation, listed below the main table.

| Name | Type | OS | Role |
|---|---|---|---|
| Proxmox host | Bare metal | Proxmox VE | Hypervisor |
| pihole | LXC | Ubuntu 22.04 | DNS / ad-blocking, local DNS records |
| caddy | LXC | Ubuntu 22.04 | Reverse proxy, TLS termination |
| vaultwarden | LXC | Ubuntu 22.04 | Password manager |
| gitea | LXC | Debian 12 | Git hosting |
| nextcloud | LXC | Debian 12 | File storage |
| prometheus | LXC | Debian 13 | Metrics collection |
| grafana | LXC | Debian 13 | Dashboards |
| rss | LXC | Debian 13 | [FreshRSS](services/freshrss.md), security news reader, tailnet access only |
| wazuh-server | VM | Amazon Linux 2023 | SIEM / security monitoring |
| kali | VM | Kali Linux | Pentesting, on-demand |
| metasploitable | VM | Metasploitable | Deliberately vulnerable pentest target, isolated segment |

Kali and Metasploitable sit on a separate virtual bridge from everything else, routed so Metasploitable is reachable only from Kali, never from the rest of the LAN.

See individual service pages for what runs on each host and how they're configured.

## Workstation
My daily workstation runs an Arch-based distro and hosts two VMs through KVM/libvirt.

| Name | Type | OS | Role |
|---|---|---|---|
| Windows 11 VM | KVM VM | Windows 11 Enterprise Evaluation | Windows-only apps and PowerShell practice, bridged to the LAN so it can reach Kali, Metasploitable and Wazuh |
| Kali Purple VM | KVM VM | Kali Linux Purple | Defensive-tools sandbox for [Suricata](services/suricata.md) and [Zeek](services/zeek.md), internet access only |

Windows 11 VM: 4 vCPU, 6GB RAM, 100GB sparse disk, UEFI with Secure Boot and an emulated TPM 2.0. The disk lives on its own btrfs subvolume with copy-on-write off, outside snapper. Planned: OpenSSH with key auth for PowerShell over SSH, a Wazuh agent, and a DHCP reservation. The evaluation lasts 90 days, so it gets rebuilt from the retail ISO afterward, with scripts kept outside the VM. Build steps: [Build a Windows 11 VM on KVM](runbooks/build-windows-11-vm-on-kvm.md).

### Kali Purple VM
A second Kali, separate from the one on Proxmox. The Proxmox Kali already covers the attacker role, so this one is a local sandbox for the defensive tools. 4 vCPU, 8GB RAM, 80GB disk, UEFI, on libvirt's default NAT network with a DHCP reservation. It is deliberately not enrolled in Wazuh and not on the tailnet. [vm-lockdown](services/vm-lockdown.md) keeps it off the LAN and tailnet while leaving internet access, after I found that NAT alone did not (see [the incident](incidents/kali-purple-vm-lan-tailnet-exposure.md)). SSH is off by default and started on demand for `scp`, key login only. Related: [Set up key-only SSH to a libvirt VM](runbooks/set-up-key-only-ssh-to-a-libvirt-vm.md), [Take a baseline of a fresh VM](runbooks/take-a-vm-baseline.md).
