---
title: "Fine-Tuning Claude Models with Anthropic's Fine-Tuning API"
date: 2026-05-09 08:00:00 +0530
categories: [AI, Fine-Tuning]
tags: [fine-tuning, fine-tuning-series, anthropic, claude, python]
---

Anthropic's fine-tuning offering follows the same managed pattern as yesterday's OpenAI walkthrough, with a few Claude-specific considerations worth calling out explicitly rather than treating the two as interchangeable.

## Dataset Format

```python
import json

def to_anthropic_format(examples: list[dict], path: str):
    with open(path, "w") as f:
        for ex in examples:
            f.write(json.dumps({
                "messages": ex["messages"],
            }) + "\n")
```

The message-list format is consistent with the Messages API you've already been using throughout this roadmap — a fine-tuning dataset for Claude is structurally the same shape as the request bodies from the earlier FastAPI integration posts, just collected into a file instead of sent live.

## Starting and Monitoring a Job

```python
import anthropic
client = anthropic.Anthropic()

job = client.fine_tuning.jobs.create(
    training_file_id=uploaded_file.id,
    base_model="claude-haiku-4-5",
    hyperparameters={"n_epochs": 3, "learning_rate_multiplier": 1.0},
)

status = client.fine_tuning.jobs.retrieve(job.id)
print(status.status, status.result_model_id)
```

## Why Fine-Tuning a Smaller Model Is Usually the Right Move Here

Anthropic's fine-tuning is generally most cost-effective on smaller, faster models in the lineup — the pattern from the fine-tuning-vs-prompting post applies directly: fine-tuning shines at getting a smaller, cheaper model to match a narrow slice of what a larger model can do by default. Fine-tuning the largest model in a lineup for a task a smaller fine-tuned model could handle is usually the wrong economic tradeoff.

## System Prompt Interaction

A subtlety specific to managed chat-model fine-tuning: your fine-tuned behavior and your runtime system prompt both influence output, and they can conflict if not designed together. Best practice is to keep the system prompt used at inference time consistent with (or absent from) what the model saw during training — a fine-tuned model trained without a system prompt and then served with a contradictory one at runtime can produce inconsistent behavior.

```python
response = client.messages.create(
    model=status.result_model_id,
    messages=[{"role": "user", "content": "My deployment is stuck in pending state."}],
    # keep system prompt consistent with what training examples assumed
)
```

## Evaluating Before and After

Never trust a fine-tuning job's training-loss curve alone as evidence of success. Run the held-out validation set from the dataset-construction posts through both the base model and the fine-tuned model, scored with the same rubric, and compare directly — tomorrow's post on evaluating fine-tuned models covers this comparison in depth.

## Managed vs Self-Hosted: Revisit the Decision Here

If your evaluation shows the managed fine-tuned model doesn't clear your quality bar, or you need more control over training hyperparameters and data mixing than a managed API exposes, that's the signal to move to self-hosted fine-tuning with open-weight models — exactly what tomorrow's Hugging Face TRL post covers.

---

*Part of the [Fine-Tuning series]({{ site.baseurl }}/tags/fine-tuning-series/) — next: [self-hosted fine-tuning with Hugging Face TRL]({{ site.baseurl }}/posts/fine-tuning-open-weight-models-trl/) for full control over the process.*
