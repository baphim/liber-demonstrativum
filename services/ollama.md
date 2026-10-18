---
status: Running
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Ollama

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Runs open-weight models locally, so logs and alerts can be analyzed without leaving the machine |
| **Host** | Workstation, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
Local API on port 11434, bound to localhost only. The API has no authentication, so it must stay that way. `ss -tlnp | grep 11434` should show `127.0.0.1`.

## Deployment
Installed from the Arch repos (`ollama-cuda`) and enabled with systemd. The GPU is an RTX 3060 with 12 GB of VRAM. I first assumed a 3090, which was wrong and moved the target from 32B models down to the 14B class. Steps: [Set up Ollama](../runbooks/set-up-ollama.md). The model list and measurements are in [Local models](local-models.md).

## Configuration
A systemd drop-in sets an 8192-token context, flash attention and an 8-bit KV cache. With those, a 14B model at 4-bit stays at 100% on the GPU at roughly 34 tokens per second.

`ollama ps` only lists loaded models. It comes back empty after a restart or about five minutes idle, which is normal.

## Dependencies
- `ollama-cuda` and the NVIDIA driver.
- [Wazuh](wazuh.md), as the source of alerts for the triage script.

## Known Issues
- A 14B model at 8k context leaves about 2.5 GB of VRAM for the desktop. Do not push the context higher.
- Models with a thinking phase print a long reasoning section first. `--think=false` turns it off.
- Small models over-flag. On a test log, one rated a routine root cron job as High, and another with a forced JSON schema flagged a single typo'd password as a possible brute force. Detection now happens in code and the model only explains what the code found. See [Detection in code, explanation in the model](../concepts/detection-in-code-explanation-in-the-model.md).
- Log text is untrusted input. The triage script tells the model not to follow instructions that appear inside it. See [Models, agents and weights](../concepts/models-agents-and-weights.md).
- On a real alert run, the model recommended isolating a machine over a false positive. Its explanations must be checked against the raw alert, see [the incident](../incidents/wazuh-alert-review.md).

## Related Notes
- [Local models](local-models.md)
- [Set up Ollama](../runbooks/set-up-ollama.md)
- [Run Wazuh alert triage with a local model](../runbooks/run-wazuh-alert-triage.md)
- [Wazuh](wazuh.md)

## Tags
#service
