---
tags:
  - runbook
  - "#zeek"
  - "#podman"
---

# Capture traffic with Zeek in a podman container

Captures live traffic on a VM's interface with Zeek in a podman container, then reads the logs. I use the container because Kali's own `zeek` package does not install on the current release. Podman has no daemon, so it adds nothing to the VM's listening ports. See [Zeek](../services/zeek.md).

## Step 1: Install podman and check the baseline
Every unit file should show `disabled` in the state column, and nothing new should be listening.
```
sudo apt install podman
systemctl list-unit-files | grep -i podman
sudo ss -tulpn
```

## Step 2: Pull the image and note the version
`latest` moves, so record the version. Root has its own image storage, so the first `sudo podman run` pulls the image again.
```
podman pull docker.io/zeek/zeek:latest
podman run --rm docker.io/zeek/zeek:latest zeek --version
```

## Step 3: Make a scratch folder
Zeek writes its logs into the folder it starts in.
```
mkdir -p ~/zeek-lab && cd ~/zeek-lab
```

## Step 4: Start the capture
Live sniffing needs root, host networking and two capabilities. `-C` ignores checksums, because packets the VM sends can have unfilled checksums. The `-w` path needs a leading slash.
```
sudo podman run --rm --net=host --cap-add=NET_RAW --cap-add=NET_ADMIN -v "$PWD":/logs -w /logs docker.io/zeek/zeek:latest zeek -C -i eth0
```

## Step 5: Generate traffic and stop
From a second terminal, make some traffic. Wait about 30 seconds, then press `Ctrl+C` in the first terminal. Zeek writes its logs when it exits.
```
curl -s http://example.com -o /dev/null
getent hosts example.org
ping -c 2 1.1.1.1
```

## Step 6: Take ownership of the logs
Root created them.
```
sudo chown -R "$USER": ~/zeek-lab
ls ~/zeek-lab
```

## Step 7: Read the logs
Use `zeek-cut` inside the container. The field names have to be exact, or it prints a blank column instead of an error.
```
cat conn.log | podman run --rm -i docker.io/zeek/zeek:latest zeek-cut id.orig_h id.resp_h id.resp_p proto service
cat dns.log | podman run --rm -i docker.io/zeek/zeek:latest zeek-cut query qtype_name answers
cat http.log | podman run --rm -i docker.io/zeek/zeek:latest zeek-cut method host uri user_agent
cat reporter.log
```

## Verification
- `ls` shows `conn.log`, `dns.log`, `http.log` and a few housekeeping logs.
- The logs match the traffic generated: the HTTP request, the DNS lookups and the ICMP.
- `http.log` shows the same `curl` request that trips the local rule in [Add a local Suricata rule](add-a-local-suricata-rule.md), so one request is visible through both tools.
- `reporter.log` shows packets received on the interface and 0 dropped.
- Timestamps are in UTC, so expect them to be hours ahead of local time.

## Related Notes
- [Zeek](../services/zeek.md)
- [Suricata](../services/suricata.md)

## Tags
#runbook
