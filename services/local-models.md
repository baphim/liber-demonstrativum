---
status: Running
host: "workstation"
tags: [reference]
---

# Local models

Every model pulled into [Ollama](ollama.md) gets an entry here, with its size, what it measured on this card, and what I use it for. The GPU is an RTX 3060 with 12 GB of VRAM.

## How to read the numbers
- **File size** is the download size of the default tag, which is 4-bit weights unless the tag says otherwise.
- **Fit** assumes about 11 GB usable, since the desktop takes some. File size plus roughly 1 GB for context has to stay under that. Anything larger spills into system RAM and slows to a crawl.
- **Tokens per second** comes from a small benchmark script that sends one prompt and reads the eval rate. The 14B models all land near 34 because the card is limited by memory bandwidth, which scales with file size. Speed does not separate them.

## Pulled
| Model | Parameters | File size | Tokens/s | Use for |
|-------|-----------|-----------|----------|---------|
| `gemma4:12b` | 11.9B | 8.0 GB | 67.2 | First choice for explaining alerts. Also takes image and audio input |
| `qwen3:14b` | 14B | 9.3 GB | 33.8 | Second choice for alerts. Short answers, good coding and reasoning |
| `phi4` | 14B | 9.1 GB | 35.1 | Logic and long careful explanations. Its advice puts a password change ahead of containment |
| `deepseek-r1:14b` | 14B (distilled) | 9.0 GB | 33.8 | Hard problems where I want to read the chain of thought. Slow, because it thinks first |
| `qwen2.5-coder:14b` | 14B | 9.0 GB | 34.1 | Writing and fixing scripts and detection rules. Generic on security advice |
| `llama3.1:8b` | 8B | 4.9 GB | 63.3 | Fast and light. Weak advice, so not for triage |
| `nomic-embed-text` | about 137M | 274 MB | n/a | Embeddings only. Returns 768 dimensions per input, confirmed through the embed API |

## Measurement gotchas
- `gemma4:12b` showed 1.1 GB loaded in both `ollama ps` and my script, which is wrong. `nvidia-smi` before and after loading showed about 8.8 GB, so for this model trust `nvidia-smi`.
- Its 67 tokens per second is more than an 8 GB dense model should manage on this card. The model parameters mention a draft-token setting, which may mean speculative decoding. That is a guess I have not confirmed.
- The benchmark script printed 0.0 GB for `phi4` because `ollama ps` lists it as `phi4:latest`. Use the full name from `ollama list`.
- Thinking models spend output tokens before answering. On the same three-sentence prompt, the non-thinking models used about 100, while `qwen3:14b` used 364, `deepseek-r1:14b` 471 and `gemma4:12b` 610.

## Triage test on a fake log
A fake ssh log with a brute-force run followed by a root login from the same address. The code finds the two findings and the model only explains them. All six chat models stayed on those two findings and none invented a third. They differ on advice for the root login:

| Rank | Model | Advice |
|------|-------|--------|
| 1 | `gemma4:12b` | Isolate the host and start incident response. Strongest, but calls the login a confirmed breach, which is more certain than the log shows |
| 2 | `qwen3:14b` | Check for file changes and process anomalies, block the address. Short and sound |
| 3 | `phi4` | Change the root password first. Thorough, but rotating a password does not remove someone already in |
| 4 | `deepseek-r1:14b` | Check the origin, review activity, disable root login. Reasonable |
| 5 | `qwen2.5-coder:14b` | Generic: review logs, change the password, tighten SSH |
| 6 | `llama3.1:8b` | No containment step |

One prompt and one run, so the top and bottom are more reliable than the middle.

## Triage test on real alerts
The same script on four real alert groups, with `gemma4:12b` and `qwen3:14b`. I had already worked out the right answers by hand.

| Alert | `gemma4:12b` | `qwen3:14b` | Right answer |
|-------|-------------|-------------|--------------|
| SSH brute force on the VPS | Check for a successful login from the address. Good | Look up the address reputation. Misses the check that matters | Noise unless a login succeeded |
| Rootcheck trojan alert on the workstation | Isolate and do forensics. Wrong | Verify checksums, then isolate. The checksum step is the right test | False positive |
| Listening ports changed on the VPS | Blames the reverse proxy. Wrong | Same idea, hedged | `tailscaled` temporary UDP ports |

Neither model is better overall, and the differences are small next to what they got wrong together. Both invented a cause for the ports alert because the alert sample was cut off before the part that changed, and both treated the rootcheck alert as a likely infection because the sample showed one path. The input was the weak point, not the model. The fix is in the script: include the diff for the ports alert, list distinct paths for the rootcheck alert, and attach a one-line note per rule about what it usually means and the first check.

## Models that do not fit
Anything near 20 GB or more (30B+ dense models and the 27B class) spills into system RAM on a 12 GB card.

## Testing a new model
1. Run the benchmark script with the exact name from `ollama list`.
2. If the loaded size looks wrong, compare `nvidia-smi` memory before and after loading.
3. Run the triage script on the fake log. Check that the explanations stay on the findings and that the advice for the root login includes containment.
4. Write down what the model is good at and what it got wrong.

## Related Notes
- [Ollama](ollama.md)
- [Set up Ollama](../runbooks/set-up-ollama.md)
- [Wazuh alert review](../incidents/wazuh-alert-review.md)

## Tags
#reference
