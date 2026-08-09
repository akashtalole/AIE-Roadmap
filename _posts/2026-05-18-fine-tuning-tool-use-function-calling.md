---
title: "Fine-Tuning for Tool Use and Function Calling"
date: 2026-05-18 08:00:00 +0530
categories: [AI, Fine-Tuning]
tags: [fine-tuning, fine-tuning-series, function-calling, tool-use, python]
---

Most models handle common tool-calling patterns well out of the box via prompting alone. Fine-tuning for tool use earns its cost in a narrower case: a fixed, high-volume set of tools where you need consistently correct argument formatting at a reliability level prompting alone doesn't reach.

## When This Is Worth Doing

- **High call volume, narrow toolset** — an agent that calls the same 5-10 tools thousands of times a day, where a small percentage improvement in argument accuracy has real operational impact
- **Unusual or strict argument formats** — tools requiring precise formats (specific date formats, nested structures, domain-specific IDs) that a general model gets subtly wrong often enough to matter
- **Latency-sensitive routing** — fine-tuning a small model to reliably route to the right tool, reserving a larger model only for cases the small model is uncertain about

## Dataset Format for Tool-Calling SFT

```json
{
  "messages": [
    {"role": "system", "content": "You have access to: get_order_status, cancel_order, update_address."},
    {"role": "user", "content": "Cancel order 88213, customer says it's the wrong item"},
    {"role": "assistant", "content": null, "tool_calls": [
      {"name": "cancel_order", "arguments": {"order_id": "88213", "reason": "wrong_item"}}
    ]}
  ]
}
```

Include a representative range of phrasing for the same intent — "cancel this", "I don't want it anymore", "please refund and cancel" should all map to the correct tool call, and the training data needs that phrasing diversity or the fine-tuned model will overfit to a narrow set of trigger phrases.

## Include Negative Examples: When Not to Call a Tool

```json
{
  "messages": [
    {"role": "user", "content": "What's your return policy?"},
    {"role": "assistant", "content": "Our return policy allows returns within 30 days..."}
  ]
}
```

A tool-calling fine-tune trained only on positive examples (every example calls a tool) tends to over-trigger tool calls even for questions that should just be answered directly — mix in a meaningful proportion of no-tool-call examples.

## Include Argument Error Recovery

```json
{
  "messages": [
    {"role": "assistant", "content": null, "tool_calls": [{"name": "get_order_status", "arguments": {"order_id": "ABC"}}]},
    {"role": "tool", "content": "Error: order_id must be numeric"},
    {"role": "assistant", "content": "Could you confirm the numeric order number? The one you provided doesn't look like a valid order ID."}
  ]
}
```

Training on the recovery path, not just the happy path, is what makes a fine-tuned tool-calling model gracefully handle malformed input instead of retrying the same broken call repeatedly.

## Evaluating Tool-Calling Fine-Tunes Specifically

Beyond general response quality, score tool-calling fine-tunes on argument-level correctness, not just "did it call the right tool":

```python
def score_tool_call(predicted: dict, expected: dict) -> float:
    if predicted["name"] != expected["name"]:
        return 0.0
    matching_args = sum(1 for k, v in expected["arguments"].items() if predicted["arguments"].get(k) == v)
    return matching_args / len(expected["arguments"])
```

## The Comparison Worth Running First

Before committing to this fine-tune, benchmark your current prompting-based tool-call accuracy against your actual reliability requirement. Fine-tuning a model for tool use is a meaningful investment — confirm prompting genuinely can't reach the bar first, since a better tool schema (from April's tool-design post) sometimes closes the gap without any training at all.

---

*Part of the [Fine-Tuning series]({{ site.baseurl }}/tags/fine-tuning-series/) — next: [multi-task fine-tuning]({{ site.baseurl }}/posts/multi-task-fine-tuning-one-model-many-skills/) — one model handling several distinct capabilities at once.*
