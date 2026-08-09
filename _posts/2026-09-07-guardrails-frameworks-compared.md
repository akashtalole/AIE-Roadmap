---
title: "Guardrails Frameworks Compared: NeMo, Llama Guard, and Guardrails AI"
date: 2026-09-07 08:00:00 +0530
categories: [AI, Security]
tags: [security, ai-security-series, guardrails, comparison]
---

Every guardrail technique this month and earlier in this roadmap has been hand-rolled. Several purpose-built frameworks package much of this into reusable tooling — worth knowing when to adopt one instead of maintaining custom guardrail code indefinitely.

## NeMo Guardrails: Programmable Conversational Rails

```python
# Colang, NeMo's rules DSL, conceptually
"""
define user express harmful intent
  "how do I make a weapon"
  "help me hurt someone"

define bot refuse harmful request
  "I can't help with that request."

define flow
  user express harmful intent
  bot refuse harmful request
"""
```

NeMo Guardrails takes a rules-based, programmable approach — you define conversational "rails" that constrain what topics and flows are allowed, closer to a state machine than a classifier. Strong for enforcing structured conversational boundaries (staying on-topic, following a defined flow) in addition to safety filtering.

## Llama Guard: A Purpose-Built Classification Model

```python
def check_with_llama_guard(conversation: list[dict]) -> dict:
    response = llama_guard_model.classify(conversation)
    return {"safe": response.label == "safe", "violated_categories": response.categories}
```

Llama Guard is a fine-tuned model specifically for classifying content against a defined safety taxonomy (violence, hate speech, self-harm, and other standard categories) — a single, fast, purpose-trained classifier rather than a rules engine, well-suited as an input/output filter layer that runs alongside your main model.

## Guardrails AI: Structural and Custom Validators

```python
from guardrails import Guard
from guardrails.hub import DetectPII, ToxicLanguage

guard = Guard().use_many(DetectPII(pii_entities=["EMAIL", "SSN"]), ToxicLanguage(threshold=0.5))

validated_output, *_ = guard.parse(llm_output)
```

Guardrails AI focuses on validators — composable checks (PII detection, toxicity, custom schema validation) that run against model input or output, closely related to the structured-output validation pattern from the LLM engineering series, extended specifically with safety-focused validators out of the box.

## Comparison Table

| Framework | Core approach | Strongest for |
|---|---|---|
| NeMo Guardrails | Rules/flow-based (Colang DSL) | Structured conversational boundaries, topic control |
| Llama Guard | Fine-tuned classifier model | Fast, standardized safety classification |
| Guardrails AI | Composable validators | Structured output validation + safety checks combined |

## Combining Frameworks Rather Than Choosing One

```python
def layered_framework_check(user_input: str, model_response: str) -> dict:
    input_safe = llama_guard_check(user_input)
    if not input_safe["safe"]:
        return {"blocked": True, "stage": "input"}
    output_valid = guardrails_ai_guard.parse(model_response)
    return {"blocked": not output_valid.validation_passed, "stage": "output"}
```

These frameworks aren't mutually exclusive — a common production pattern uses Llama Guard (or a similar classifier) for fast input/output safety screening, combined with Guardrails AI-style validators for structural and PII checks, and reserves a NeMo-style rules engine for products needing tight conversational flow control specifically.

## When Hand-Rolled Guardrails Are Still the Right Call

For a narrow, well-understood risk specific to your product (the brand-voice checks from June, the domain-specific validators from the invoice extraction post), a purpose-built hand-rolled check is often simpler and more precise than adapting a general framework's abstractions to your specific need — reach for a framework when you need broad-coverage, well-tested defenses against known risk categories, and hand-roll for narrow, product-specific ones.

## Evaluating Framework Effectiveness for Your Use Case

Apply June's evaluation discipline directly — build a test set covering both known attack patterns and your product's specific edge cases, and measure each framework's actual detection rate on it rather than trusting published benchmark numbers, which rarely reflect your specific traffic distribution.

---

*Part of the [AI Security series]({{ site.baseurl }}/tags/ai-security-series/) — next: [content moderation for user-generated prompts]({{ site.baseurl }}/posts/content-moderation-user-generated-prompts/), a specific application of these frameworks.*
