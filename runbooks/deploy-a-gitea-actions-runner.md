---
tags:
  - runbook
  - "#gitea"
  - "#docker"
---

# Deploy a Gitea Actions runner in a Debian LXC

Stand up `act_runner` in Docker mode inside its own unprivileged Debian LXC, registered to one Gitea repo. The runner gets its own container because mounting the Docker socket into it is effectively root on that machine. Values in angle brackets are placeholders. See [Gitea Actions runner](../services/gitea-actions-runner.md) for the finished service.

## Step 1: Pre-checks on the Proxmox host
Confirm a free container ID and address, and a storage that supports container disks.
```
pct list
pvesm status
pveam available --section system | grep debian-13 | grep amd64
pveam download local <debian-13-template>
```
Filter the template list for the right architecture explicitly. A plain grep for the distro name can match both the amd64 and arm64 builds and silently take the wrong one.

Use LVM-thin for the rootfs rather than a ZFS-backed one: Docker's overlay2 storage driver behaves better on a plain filesystem.

## Step 2: Create the container
```
pct create <ctid> local:vztmpl/<debian-13-template>.tar.zst \
  --hostname <name> --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 2 --memory 4096 --swap 512 \
  --rootfs <storage>:20 \
  --net0 name=eth0,bridge=vmbr0,ip=<ip>/24,gw=<gateway> \
  --nameserver <dns> --onboot 1 --tags ci \
  --ssh-public-keys /root/<key>.pub
pct start <ctid>
```
Both `nesting` and `keyctl` are needed for Docker to run inside an unprivileged container. If a pasted multi-line command keeps failing on the line continuations, the Proxmox web UI's container wizard builds the same configuration; just remember it doesn't expose `keyctl`, so tick it under Options, Features before the first start.

## Step 3: Install Docker
Docker's own apt repository, inside the container:
```
apt update && apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
tee /etc/apt/sources.list.d/docker.sources <<REPO
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
REPO
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
docker run --rm hello-world
```
> **Gotcha:** if `apt update` returns a 404 for the new repo, check the URI line for a typo like `linux.debian` instead of `linux/debian`. Easy to introduce and easy to miss on a read-through.

## Step 4: Get a registration token
In Gitea, on the repo: Settings, Actions, Runners, create a new runner. A repo-level runner is the tightest scope available; an org- or instance-level one is for later if more than one repo needs CI.

## Step 5: Configure and start the runner
```
mkdir -p /opt/act-runner && cd /opt/act-runner
printf 'RUNNER_TOKEN=<token>\n' > .env
chmod 600 .env
```
`compose.yaml`, with the label matching whatever `runs-on` value the workflow uses:
```yaml
services:
  runner:
    image: gitea/act_runner:latest
    restart: unless-stopped
    environment:
      GITEA_INSTANCE_URL: <gitea-url>
      GITEA_RUNNER_REGISTRATION_TOKEN: ${RUNNER_TOKEN}
      GITEA_RUNNER_NAME: <runner-name>
      GITEA_RUNNER_LABELS: "ubuntu-latest:docker://node:<version>-bookworm"
    volumes:
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
```
```
docker compose up -d
docker compose logs -f
```
Look for a "registered successfully" line, then the runner idling. The `./data` bind mount keeps the registration across restarts, so it doesn't re-register every time the container comes back up.

## Step 6: Harden the container
```
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```
In `sshd_config`: `PermitRootLogin prohibit-password`, then `PasswordAuthentication no` once key login is confirmed from a second session. Keep the first session open while testing the second, so a typo can't lock you out.

## Step 7: Add a workflow
In the project's repo, a workflow file under its Actions directory. A minimal example:
```yaml
name: check
on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    container:
      image: node:<version>-bookworm
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run check
      - run: npm run build
```
If a job sits queued forever instead of running, the `runs-on` label in the workflow doesn't exactly match a registered runner's label. Check the instance's runner list.

## Step 8: Lock down networking
Turn on the Proxmox firewall for this guest (the datacenter-level firewall has to be on too). Allow outbound DNS and HTTPS only, inbound SSH from one trusted host only, and block the rest of the LAN. Test with a request that should succeed and one that should be blocked, since rule order matters and a rule that looks right can still shadow an earlier one.

## Step 9: Install monitoring and snapshot
Add the monitoring agent used elsewhere in the lab, pinned to the same version as everything else. Take a snapshot once the runner is confirmed working, and add the container to the regular backup job; the only state worth keeping is the runner's registration directory, so a rebuild from this runbook is otherwise quick.

## Verification
- `docker run --rm hello-world` succeeds inside the container.
- `docker compose ps` shows the runner up, and the repo's Runners page lists it as idle or online.
- A push or pull request starts a run and it finishes with the job's checks passing.
- The `.env` file is mode 600.
- Key-only SSH is confirmed and password auth is off.
- With the firewall on: DNS and HTTPS still work from inside the container, and a blocked LAN port does not answer.

## Related Notes
- [Gitea Actions runner](../services/gitea-actions-runner.md)
- [Gitea](../services/gitea.md)

## Tags
#runbook
