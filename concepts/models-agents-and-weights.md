---
tags:
  - concept
  - "#llm"
---

# Models, agents and weights

The vocabulary I needed before running a model locally, in my own words.

## Terms
- **Model:** the trained network. Text in, text out, no memory between calls, and it cannot act on anything by itself.
- **Agent:** a model in a loop with tools, so it can do things and not only say them. Claude Code is an agent.
- **Weights:** the learned numbers inside the model. An open-weight model has its files published, so anyone can download and run it locally.
- **Abliteration:** editing an open-weight model to remove its tendency to refuse, without retraining. It is only possible because the weights are open.
- **Quantization:** storing the weights in fewer bits. A 4-bit version needs about a quarter of the memory of the 16-bit original, with some quality loss.
- **Context:** the tokens the model holds at once. It costs VRAM through the KV cache, and input past the window is silently dropped.
- **Prompt injection:** text in a log, page or file that tries to give the model instructions. It matters whenever a model reads untrusted text.

## In my setup
VRAM sets the ceiling, so quantization and context are the two dials that decide whether a model stays on the GPU. See [Local models](../services/local-models.md) for sizes and [Ollama](../services/ollama.md) for the configuration.

Prompt injection is why the triage script tells the model that log text is untrusted data, and why the model only explains findings the code produced. See [Detection in code, explanation in the model](detection-in-code-explanation-in-the-model.md).

## Related Notes
- [Ollama](../services/ollama.md)
- [Detection in code, explanation in the model](detection-in-code-explanation-in-the-model.md)

## Tags
#concept
