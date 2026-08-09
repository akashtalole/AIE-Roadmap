---
title: "Data Cleaning and Deduplication for Training Sets"
date: 2026-05-07 08:00:00 +0530
categories: [AI, Fine-Tuning]
tags: [fine-tuning, fine-tuning-series, data-quality, python]
---

A dataset combining curated real examples and generated synthetic ones from the last two posts almost always has duplicates, near-duplicates, and quality outliers hiding in it. Cleaning this before training isn't optional busywork — duplicate-heavy training data measurably skews a fine-tuned model toward overfitting on whatever's overrepresented.

## Exact and Near-Duplicate Detection

Exact duplicates are trivial to catch with a hash; near-duplicates (same scenario, trivially reworded) need embedding similarity:

```python
import hashlib

def exact_dedup(examples: list[dict]) -> list[dict]:
    seen = set()
    result = []
    for ex in examples:
        h = hashlib.sha256(json.dumps(ex, sort_keys=True).encode()).hexdigest()
        if h not in seen:
            seen.add(h)
            result.append(ex)
    return result

def near_dedup(examples: list[dict], threshold: float = 0.95) -> list[dict]:
    embeddings = [embed(ex["messages"][-1]["content"]) for ex in examples]
    keep = []
    for i, emb in enumerate(embeddings):
        if not any(cosine_similarity(emb, embeddings[j]) > threshold for j in keep):
            keep.append(i)
    return [examples[i] for i in keep]
```

Near-dedup is O(n²) as written — fine for a few thousand examples, but switch to an approximate nearest-neighbor index (FAISS, from the vector database series) for larger datasets.

## Filtering Low-Quality Examples

```python
def filter_quality(examples: list[dict]) -> list[dict]:
    return [
        ex for ex in examples
        if MIN_LENGTH < len(ex["messages"][-1]["content"]) < MAX_LENGTH
        and not is_truncated(ex)
        and not has_repetition_loop(ex["messages"][-1]["content"])
        and language_detected(ex["messages"][-1]["content"]) == expected_language
    ]
```

`has_repetition_loop` matters specifically for synthetic data — generation occasionally degenerates into repeated phrases, and training on that teaches the model the same bad habit.

## Balancing Category Distribution

Even after deduplication, a dataset can be skewed — 80% of examples covering the easy, common case and 20% spread across everything else. Check the distribution across your task's known categories, and either downsample the overrepresented category or generate more of the underrepresented ones rather than training on whatever ratio you happened to collect:

```python
def check_category_balance(examples: list[dict]) -> dict:
    counts = Counter(ex["category"] for ex in examples)
    total = len(examples)
    return {cat: count / total for cat, count in counts.items()}
```

## PII and Sensitive Data Scrubbing

Any real production transcripts in the dataset need a PII pass before training — names, emails, account numbers, anything that shouldn't be memorized into model weights. This connects directly to September's AI security series on PII detection; run that same tooling over training data, not just live traffic.

## The Final Sanity Check

Before kicking off a training run, manually read a random sample of 30-50 examples from the final cleaned dataset end to end. Automated checks catch structural problems; they miss a systematic issue where every example subtly reflects the same bias or gap that only becomes obvious reading them as a human would.

---

*Part of the [Fine-Tuning series]({{ site.baseurl }}/tags/fine-tuning-series/) — next: with clean data in hand, [fine-tuning OpenAI models via the API]({{ site.baseurl }}/posts/fine-tuning-openai-models-api/), the fastest path to a first fine-tuned model.*
