---
title: "Merging Fine-Tuned Models with Model Soups and TIES"
date: 2026-05-22 08:00:00 +0530
categories: [AI, Fine-Tuning]
tags: [fine-tuning, fine-tuning-series, model-merging, python]
---

Instead of choosing between separate task-specific adapters from the multi-task post, weight-space merging combines multiple fine-tuned models — or checkpoints from the same run — directly, producing a single model that blends their behaviors without any additional training.

## Model Soups: Simple Weight Averaging

The most basic form: average the weights of several fine-tuned checkpoints, trained on the same task with different hyperparameters or random seeds.

```python
def average_checkpoints(checkpoint_paths: list[str]) -> dict:
    state_dicts = [torch.load(p) for p in checkpoint_paths]
    averaged = {}
    for key in state_dicts[0]:
        averaged[key] = torch.stack([sd[key] for sd in state_dicts]).mean(dim=0)
    return averaged
```

This works better than intuition suggests when the checkpoints are close in weight-space — several runs of the same fine-tuning setup with minor variations — and often produces a model that outperforms any single checkpoint on generalization, effectively acting as a cheap ensemble baked into one set of weights.

## TIES-Merging: Combining Different-Task Fine-Tunes

Naive averaging breaks down when merging models fine-tuned on genuinely *different* tasks — their weight updates can conflict, canceling each other out. TIES-Merging addresses this by explicitly resolving conflicts between task updates before combining them:

```python
def ties_merge(base_weights: dict, task_deltas: list[dict], density: float = 0.2) -> dict:
    merged = {}
    for key in base_weights:
        deltas = [td[key] for td in task_deltas]
        trimmed = [trim_to_top_k_magnitude(d, density) for d in deltas]  # keep only strongest signals
        sign_resolved = resolve_sign_conflicts(trimmed)  # elect a consistent sign per parameter
        merged[key] = base_weights[key] + sum(sign_resolved) / len(sign_resolved)
    return merged
```

The two key steps — trimming to the most significant weight changes and resolving sign conflicts where different tasks pull the same parameter in opposite directions — are what make TIES meaningfully better than naive averaging for combining genuinely distinct capabilities into one model.

## LoRA-Specific Merging

For LoRA adapters specifically, merging is simpler because you're combining small adapter matrices, not full model weights:

```python
from peft import PeftModel

model = PeftModel.from_pretrained(base_model, "adapters/task_a")
model.load_adapter("adapters/task_b", adapter_name="task_b")
model.add_weighted_adapter(["default", "task_b"], weights=[0.5, 0.5], adapter_name="merged", combination_type="ties")
model.set_adapter("merged")
```

## When Merging Beats Multi-Task Training

Merging is attractive when you already have separately fine-tuned models and want to combine them without a full retraining run, or when different teams own different task-specific fine-tunes and merging lets you compose their work without coordinating a joint training run. It's generally not as reliable as genuinely multi-task training from the start when you're starting from scratch and can plan the combination up front.

## Always Evaluate the Merged Result Per-Task

A merged model's quality on each original task needs to be checked independently against the pre-merge specialist model — merging can degrade performance on one or more tasks even when it helps overall, and that tradeoff needs to be an explicit, measured decision, not an assumption.

---

*Part of the [Fine-Tuning series]({{ site.baseurl }}/tags/fine-tuning-series/) — next: [continual fine-tuning]({{ site.baseurl }}/posts/continual-fine-tuning-without-retraining/) without starting the whole pipeline over for every update.*
