---
title: "Content Moderation for User-Generated Prompts"
date: 2026-09-08 08:00:00 +0530
categories: [AI, Security]
tags: [security, ai-security-series, content-moderation, python]
---

Yesterday's frameworks provide the tooling; this post covers applying it specifically to the input side — moderating what users submit before it ever reaches your model, distinct from the output-filtering post that follows tomorrow.

## Why Input Moderation Is a Distinct Layer

Input moderation and output filtering serve different purposes: input moderation prevents abusive or policy-violating requests from consuming resources and potentially manipulating the model at all, while output filtering (tomorrow) catches cases where a request seemed fine but produced a problematic response. Both are needed — neither substitutes for the other.

## A Basic Moderation Pipeline

```python
def moderate_input(user_input: str) -> dict:
    moderation_result = moderation_api.check(user_input)
    categories_flagged = [cat for cat, score in moderation_result.items() if score > MODERATION_THRESHOLDS[cat]]
    return {"allowed": len(categories_flagged) == 0, "flagged_categories": categories_flagged}
```

Most major providers offer a dedicated, purpose-built moderation endpoint separate from their general chat models — faster and cheaper than using a full LLM call for classification, and worth using as the first-line check before a request even reaches your main model.

## Category-Specific Thresholds, Not One Global Bar

```python
moderation_thresholds = {
    "hate_speech": 0.3,       # low tolerance
    "self_harm": 0.2,          # very low tolerance — route to crisis resources instead of blocking silently
    "violence": 0.4,
    "sexual_content": 0.5,     # threshold depends heavily on product context
}
```

Different categories warrant different sensitivity and different *responses* — flagged self-harm content shouldn't just be blocked, it should ideally route to crisis resources or a specific safe response, a materially different design decision than simply refusing a request.

## Handling Borderline Cases Without Over-Blocking

```python
def handle_moderation_result(result: dict, user_input: str) -> str:
    if not result["allowed"]:
        if "self_harm" in result["flagged_categories"]:
            return get_crisis_resources_response()
        return "I can't help with that request."
    return None  # proceed normally
```

Over-aggressive moderation has real costs — legitimate creative writing, medical questions, or historical discussion can trigger naive keyword-based or overly conservative classifier thresholds, frustrating genuine users. Tune thresholds against a labeled test set spanning both genuinely problematic content and legitimate edge cases, the same evaluation discipline from June applied to moderation specifically.

## Context-Aware Moderation

```python
def moderate_with_context(user_input: str, conversation_history: list[dict], product_context: str) -> dict:
    # A question about medication dosage means something different in a medical app vs a general chatbot
    return moderation_api.check(user_input, context={"product_type": product_context, "prior_turns": conversation_history[-3:]})
```

The same input can be entirely appropriate in one product context and genuinely concerning in another — a general-purpose moderation threshold tuned without product context will systematically over- or under-moderate depending on your specific use case.

## Logging and Reviewing Moderation Decisions

```python
def log_moderation_decision(user_input_hash: str, result: dict, action_taken: str):
    moderation_log.record({"input_hash": user_input_hash, "flagged_categories": result["flagged_categories"],
                            "action": action_taken, "timestamp": now()})
```

Hash, don't store, the raw flagged input for routine logging — retain enough to analyze patterns and false-positive rates without creating a growing store of sensitive user content. Periodic human review of a sample of moderation decisions (echoing June's human-evaluation workflow) is what catches threshold miscalibration before it becomes a significant user-trust problem.

## Repeated-Offense Handling

Beyond per-request moderation, track patterns across a user's request history — a single flagged request might be a false positive or a momentary lapse, while a sustained pattern of policy-violating attempts is a different problem warranting account-level action, not just per-message blocking.

---

*Part of the [AI Security series]({{ site.baseurl }}/tags/ai-security-series/) — next: [output filtering]({{ site.baseurl }}/posts/output-filtering-unsafe-model-responses/), catching what gets through despite input moderation.*
