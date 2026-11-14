---
status: Running
host: "a dedicated LXC"
reviewed: 2026-11-14
tags: [service]
---

# Gitea Actions runner

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | CI for one of my own projects: lint, type-check and build run on every push and pull request |
| **Host** | A dedicated unprivileged LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
No inbound access. It only makes outbound connections: it registers with and polls [Gitea](gitea.md), and it reaches Docker Hub, the npm registry, and GitHub for one action it uses. SSH is key only, for me.

## Deployment
`gitea/act_runner` in Docker mode, run with Docker Compose inside an unprivileged Debian LXC. The runner has its own container rather than living on the Gitea host or the workstation: mounting the Docker socket is effectively root on whatever machine it runs on, and CI runs whatever is in the repo, so I wanted it somewhere with no other secrets on it. Steps: [Deploy a Gitea Actions runner in a Debian LXC](../runbooks/deploy-a-gitea-actions-runner.md).

## Configuration
- Registered at the repo level, to one repo only. The tightest scope Gitea offers.
- A label on the runner maps a `runs-on: ubuntu-latest` job to a specific container image, so I control exactly what the job runs in without touching the workflow file's own `runs-on` value.
- The registration token lives in its own file, mode 600, never in the compose file itself.
- The registration itself is kept in a bind-mounted data directory, so the runner does not have to re-register every time the container restarts.
- The workflow lives in the project's own repo, checkout, install, lint, build.

## Dependencies
- [Gitea](gitea.md), for job dispatch.
- Docker Hub, the npm registry, and GitHub's action marketplace, all outbound only.

## Known Issues
- The Docker socket is mounted into the runner container, so any CI job can act as root on this LXC. That is the reason this LXC holds nothing else.
- The Proxmox firewall is not enabled for this guest yet, so it can currently reach the rest of the LAN outbound. Acceptable while the only code it runs is mine; it would need locking down before it ever ran code from people I don't trust.
- LXC isolation is weaker than a VM, since the container shares the hypervisor's kernel. If this runner ever needs to run untrusted code, it moves to a small VM instead.
- The runner image and the job's base image aren't pinned to a digest yet, and the one GitHub action it uses is pinned to a tag rather than a commit hash.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Deploy a Gitea Actions runner in a Debian LXC](../runbooks/deploy-a-gitea-actions-runner.md)
- [Gitea](gitea.md)

## Tags
#service
