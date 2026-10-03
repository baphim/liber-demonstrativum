# Infrastructure

Everything runs on a single Proxmox host, hypervisor on bare metal, LXCs and VMs for each service.

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
| wazuh-server | VM | Amazon Linux 2023 | SIEM / security monitoring |
| kali | VM | Kali Linux | Pentesting, on-demand |
| metasploitable | VM | Metasploitable | Deliberately vulnerable pentest target, isolated segment |

Kali and Metasploitable sit on a separate virtual bridge from everything else, routed so Metasploitable is reachable only from Kali, never from the rest of the LAN.

See individual service pages for what runs on each host and how they're configured.
