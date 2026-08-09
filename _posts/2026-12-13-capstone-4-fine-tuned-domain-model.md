---
title: "Capstone Project 4: Build a Fine-Tuned Domain Model"
date: 2026-12-13 08:00:00 +0530
categories: [AI, Career]
tags: [career, career-series, capstone, fine-tuning, python]
---

Fourth capstone: a complete fine-tuning project end to end, following May's series and its closing project-walkthrough post specifically.

## Project Brief

Fine-tune a small open-weight model (LoRA/QLoRA) for a narrow, well-defined task — a specific classification, extraction, or style-adaptation task where you can build genuine before/after evaluation evidence that fine-tuning outperforms prompting alone.

## Requirements

```python
capstone_4_requirements = {
    "justified_approach": "explicit reasoning for why fine-tuning beats prompting/RAG for this task (May's decision framework)",
    "dataset": "50+ curated, cleaned examples with a documented sourcing process (May's dataset posts)",
    "training": "LoRA fine-tune with tracked experiments (May's Weights & Biases post)",
    "evaluation": "task-specific accuracy AND general-capability regression check (May's evaluation post)",
    "serving": "a working inference endpoint using the fine-tuned adapter",
}
```

## Suggested Task Ideas

```python
task_ideas = {
    "style_adaptation": "fine-tune for a consistent tone/format on a specific content type",
    "narrow_classification": "a domain-specific categorization task with clear ground truth",
    "structured_extraction": "consistent JSON extraction for a specific document type (connects to Capstone 7)",
}
```

Choosing a task where the fine-tuning-vs-prompting comparison is genuinely close (not an obvious win either way) makes for a more interesting, defensible write-up than a task where fine-tuning was clearly always going to win.

## Milestones

```python
milestones = {
    "week_1": "baseline: measure prompting-only performance on your task (this is your comparison point)",
    "week_2": "build and clean the training dataset, following May's quality checklist",
    "week_3": "run the fine-tuning job, track experiments",
    "week_4": "evaluate against baseline, serve the adapter, write up the comparison",
}
```

## The Comparison That Makes This Project Credible

```python
def report_capstone_4_results(baseline: dict, fine_tuned: dict) -> dict:
    return {
        "baseline_accuracy": baseline["accuracy"],
        "fine_tuned_accuracy": fine_tuned["accuracy"],
        "general_capability_regression": fine_tuned["general_benchmark_delta"],
        "cost_per_1000_requests_comparison": {"baseline": baseline["cost"], "fine_tuned": fine_tuned["cost"]},
    }
```

This head-to-head comparison — not just "I fine-tuned a model" — is what makes the project genuinely demonstrate May's economics and evaluation discipline, rather than just technical execution of a training run.

## Evaluation Rubric

```python
def self_evaluate_capstone_4(project: dict) -> dict:
    return {
        "has_documented_baseline": "baseline_accuracy" in project,
        "checked_general_capability_regression": "general_benchmark_delta" in project,
        "dataset_quality_documented": project.get("dataset_cleaning_process_documented", False),
        "experiment_tracked": project.get("uses_experiment_tracking", False),
        "serves_the_adapter": project.get("has_working_inference_endpoint", False),
    }
```

## Common Pitfalls

```python
pitfalls = {
    "no_baseline_comparison": "without it, you can't actually demonstrate fine-tuning was the right call",
    "tiny_dataset": "under 30-40 examples rarely produces a meaningful, defensible result",
    "ignoring_catastrophic_forgetting": "not checking general capability regression (May's specific warning)",
    "confusing_style_and_knowledge": "May's specific caution — verify you're not just baking in facts that belong in RAG",
}
```

## Stretch Goals

```python
stretch_goals = {
    "try_dpo_instead_of_pure_sft": "if you can source preference pairs (May's DPO post)",
    "quantize_and_measure_the_tradeoff": "May/August's quantization posts, applied to your own model",
    "serve_multiple_adapters": "if you fine-tune for more than one related task (May's multi-adapter serving post)",
}
```

## Why This Project Matters for Interviews

Being able to walk through a genuine before/after fine-tuning comparison, with real numbers, directly demonstrates the depth May's series covered — a strong differentiator from candidates who can describe LoRA conceptually but haven't run the full evaluation loop themselves.

---

*Part of the [Career series]({{ site.baseurl }}/tags/career-series/) — next: [Capstone 5: an observability dashboard]({{ site.baseurl }}/posts/capstone-5-observability-dashboard/).*
