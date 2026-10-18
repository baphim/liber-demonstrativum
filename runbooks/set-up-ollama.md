---
tags:
  - runbook
  - "#ollama"
  - "#llm"
---

# Set up Ollama

Install Ollama on an Arch-based machine with an NVIDIA GPU, keep it bound to localhost, and tune the context so a model stays fully on the GPU. See [Ollama](../services/ollama.md) for the finished setup.

## Step 1: Check the GPU
VRAM decides which models fit: about 5 GB for an 8B model, 9 GB for a 14B, and 20 GB for a 32B, all at 4-bit.
```
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv
```
Read the model name off the output. Do not go by what you remember buying.

## Step 2: Check disk space
Each model is 5 to 10 GB, and they live under `/var/lib/ollama`.
```
df -h /var
```

## Step 3: Install and start
```
sudo pacman -S ollama-cuda
sudo systemctl enable --now ollama
```

## Step 4: Confirm it listens on localhost only
The API has no authentication.
```
ss -tlnp | grep 11434
```

## Step 5: Pull a model and test
```
ollama pull phi4
ollama run phi4 --verbose "Explain what a SYN flood is in three sentences"
```

## Step 6: Raise the context and shrink the cache
```
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf <<'CONF'
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=8192"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
CONF
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## Step 7: Check the result
Load the model again and look from a second terminal. `ollama ps` is empty until a model is loaded, and models unload after about five minutes idle.
```
ollama run phi4 --verbose "Explain what a SYN flood is in three sentences"
ollama ps
```

## Verification
- `ss` shows `127.0.0.1:11434`, not `0.0.0.0` or `*`.
- `ollama ps` shows `100% GPU` and a context of 8192. Anything under 100% GPU means the model spilled into system RAM and will be slow.
- `eval rate` in the verbose stats is in the tens of tokens per second. On a 3060 with `phi4` it was about 36.
- If the context still shows 4096, check the override with `systemctl show ollama | grep -i environment`.
- Models with a thinking phase can skip it with `--think=false`.

## Related Notes
- [Ollama](../services/ollama.md)
- [Local models](../services/local-models.md)
- [Run Wazuh alert triage with a local model](run-wazuh-alert-triage.md)

## Tags
#runbook
