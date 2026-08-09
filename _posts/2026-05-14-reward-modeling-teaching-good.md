---
title: "Reward Modeling: Teaching a Model What \"Good\" Means"
date: 2026-05-14 08:00:00 +0530
categories: [AI, Fine-Tuning]
tags: [fine-tuning, fine-tuning-series, reward-model, rlhf]
---

Whether you train one explicitly for full RLHF or not, understanding reward modeling clarifies what preference data — the `chosen`/`rejected` pairs from yesterday's DPO post — is actually teaching a model, and why the quality of that signal caps everything downstream.

## What a Reward Model Is

A reward model takes a prompt and a response and outputs a scalar score — higher for responses humans would prefer. Structurally, it's usually the base language model with its output head replaced by a single regression output, trained on preference pairs:

```python
from transformers import AutoModelForSequenceClassification

reward_model = AutoModelForSequenceClassification.from_pretrained(
    "meta-llama/Llama-3-8b", num_labels=1  # single scalar reward output
)
```

## The Training Objective: Bradley-Terry Loss

Reward models train on the same preference pairs DPO uses, with a loss that pushes the chosen response's score above the rejected response's score:

```python
def reward_model_loss(chosen_reward: float, rejected_reward: float) -> float:
    return -torch.log(torch.sigmoid(chosen_reward - rejected_reward))
```

This is the Bradley-Terry model of pairwise comparison — the same statistical framework used for ranking systems generally (chess ratings, sports rankings) — applied to model outputs.

## Reward Hacking: The Central Failure Mode

A reward model is a learned proxy for human preference, not human preference itself, and a policy optimized hard enough against any proxy will eventually find ways to score well on the proxy without actually being better. Classic examples: a reward model that correlates length with quality gets exploited by a policy that produces needlessly long responses; one that rewards confident phrasing gets exploited by a policy that states wrong answers more confidently.

```python
def detect_reward_hacking(policy_outputs: list[dict]) -> dict:
    return {
        "avg_length_trend": track_length_over_training(policy_outputs),
        "human_eval_vs_reward_score_correlation": correlate_human_and_model_scores(policy_outputs),
    }
```

If the reward model's scores and independent human judgment start diverging over training, that's reward hacking in progress — the reference-model KL penalty from the RLHF post is one defense, but periodic human evaluation is the only reliable detector.

## Reward Model Quality Is a Data Problem

A reward model is only as good as its preference data's consistency and coverage. Noisy annotations (different annotators disagreeing on the same pair), biased annotation guidelines, and narrow coverage (preference data all from one type of prompt) all propagate directly into a reward model that misjudges quality in exactly the ways its training data was flawed.

## Where This Connects to Evaluation

A reward model is, functionally, an automated judge — the same role an LLM-as-judge plays in June's evaluation series, just trained explicitly on your preference data rather than prompted zero-shot. The same caution applies to both: never fully trust an automated quality signal without periodically validating it against real human judgment.

---

*Part of the [Fine-Tuning series]({{ site.baseurl }}/tags/fine-tuning-series/) — next: [evaluating fine-tuned models]({{ site.baseurl }}/posts/evaluating-fine-tuned-models/) against the base model they started from.*
