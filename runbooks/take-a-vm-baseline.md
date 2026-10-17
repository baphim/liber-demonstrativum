---
tags:
  - runbook
  - "#baseline"
---

# Take a baseline of a fresh VM

Records the clean state of a fresh VM, so I can tell later what my own installs changed. It captures installed packages, enabled and running services, listening ports and idle resource use. Take it with SSH stopped, so it shows the resting state, and run it before installing any tools. I used it on the Kali Purple VM, where it showed that "Purple" is mostly a theme and the `kali-tools-*` bundles. There was no Elasticsearch, Kibana, Suricata, Zeek or Arkime, so those were my own installs.

## Step 1: Stop SSH and make the folder
```
sudo systemctl stop ssh
mkdir -p ~/baseline
```

## Step 2: Save the installed packages
The Kali metapackages show what the image actually installed. The tools check uses the full Beats names, since an earlier pattern with just `beat` also matched `libeatmydata1`. Read the output, do not just count it.
```
apt list --installed 'kali-*' 2>/dev/null | tee ~/baseline/kali-metapackages.txt
dpkg -l | grep '^ii' | grep -iE 'elasticsearch|kibana|logstash|filebeat|metricbeat|suricata|zeek|arkime' | tee ~/baseline/defensive-tools.txt
apt-mark showmanual | tee ~/baseline/manual-packages.txt | wc -l
```

## Step 3: Save what starts at boot and what is running
```
systemctl list-unit-files --state=enabled --type=service | tee ~/baseline/enabled-services.txt
systemctl list-units --type=service --state=running | tee ~/baseline/running-services.txt
```

## Step 4: Save every listening socket
In the `Local Address` column, `127.0.0.1` and `[::1]` are loopback only, while `0.0.0.0`, `*` and `[::]` listen on all interfaces and are reachable from the NAT network.
```
sudo ss -tulpn | tee ~/baseline/listening.txt
```

## Step 5: Save idle resource use
```
{ nproc; free -h; df -h /; uptime; } | tee ~/baseline/resources.txt
ps -eo pid,comm,%mem,%cpu --sort=-%mem | head -n 10
```

## Step 6: Compare later
Diff the saved files against the current state.
```
apt-mark showmanual | diff ~/baseline/manual-packages.txt -
systemctl list-unit-files --state=enabled --type=service | diff ~/baseline/enabled-services.txt -
sudo ss -tulpn | diff ~/baseline/listening.txt -
```

## Step 7: Copy the folder to the host
Start SSH from the console, pull the folder with `scp`, and stop SSH again. See [Set up key-only SSH to a libvirt VM](set-up-key-only-ssh-to-a-libvirt-vm.md).
```
scp -r <vm-name>:baseline .
```

## Verification
- `~/baseline/` has seven files, and none are empty except `listening.txt` on a quiet VM, which should hold only the header line.
- A `listening.txt` with only the header means nothing is listening on any port.
- Disabling things after the baseline does not change the baseline files. They stay as the record of the original state.

## Related Notes
- [Suricata](../services/suricata.md)
- [Zeek](../services/zeek.md)

## Tags
#runbook
