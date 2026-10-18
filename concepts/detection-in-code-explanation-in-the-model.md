---
tags:
  - concept
  - "#llm"
  - "#detection"
---

# Detection in code, explanation in the model

A language model is good at explaining something that has already been found, and unreliable at deciding whether something was found at all. So the split I settled on is deterministic code for detection and the model for explanation.

## How it works
The triage script reads the logs and applies fixed rules: count failures per user and address, group alerts by rule and host, and flag what crosses a threshold. Only the flagged items go to the model, with an instruction to explain them and nothing else. A clean log never reaches the model at all.

## What I saw
- With a plain prompt on a fake log, both models I tried caught the real problem. One rated a routine root cron job as High and contradicted itself.
- Forcing a JSON schema made the output valid every time, but the model flagged a single typo'd password on a clean log as a possible brute force, and listed four failures as four entries.
- Moving the counting into code fixed both. The clean log never reached the model, and the test log produced the two real findings and no extras.
- The model's remediation advice was the weakest part. Use it for what an alert is, not for what to do.

## The input matters as much as the model
On real alerts, two different models made the same mistakes, because the alert sample was cut before the part that changed. Better input (the diff, the distinct paths, a one-line note on the usual cause) is a bigger fix than a better model. See [Wazuh alert review](../incidents/wazuh-alert-review.md).

## Related Notes
- [Run Wazuh alert triage with a local model](../runbooks/run-wazuh-alert-triage.md)
- [Local models](../services/local-models.md)
- [Models, agents and weights](models-agents-and-weights.md)

## Tags
#concept
