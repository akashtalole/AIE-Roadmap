---
title: "PII Detection and Redaction in LLM Pipelines"
date: 2026-09-06 08:00:00 +0530
categories: [AI, Security]
tags: [security, ai-security-series, pii, python]
---

PII handling has been referenced as a to-do across this roadmap — training data cleaning in May, trace logging in June — without a dedicated implementation. This post is that implementation, covering detection and redaction as a reusable pipeline component.

## Detection: Pattern-Based and Model-Based, Combined

```python
import re

PII_PATTERNS = {
    "email": r"[\w.+-]+@[\w-]+\.[\w.-]+",
    "phone": r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",
    "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
    "credit_card": r"\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b",
}

def pattern_based_pii_scan(text: str) -> list[dict]:
    findings = []
    for pii_type, pattern in PII_PATTERNS.items():
        for match in re.finditer(pattern, text):
            findings.append({"type": pii_type, "span": match.span(), "value": match.group()})
    return findings
```

Pattern matching reliably catches structured PII (emails, phone numbers, SSNs) but misses unstructured personal information — names, addresses, or context-dependent sensitive details ("the patient with the rare condition diagnosed last Tuesday") that only a model-based approach can reasonably catch.

```python
def model_based_pii_scan(text: str) -> list[dict]:
    response = llm.chat([{
        "role": "user",
        "content": f"Identify all personally identifiable information in this text. "
                    f"Return JSON list of {{type, text_span, sensitivity: 'high'|'medium'|'low'}}.\n\n{text}"
    }], temperature=0)
    return json.loads(response.content)
```

Combining both — pattern matching for high-confidence structured PII, model-based scanning for unstructured or contextual PII — catches more than either alone, at the added cost and latency of an extra model call for the second layer.

## Redaction Strategies

```python
def redact_text(text: str, findings: list[dict], strategy: str = "mask") -> str:
    sorted_findings = sorted(findings, key=lambda f: f["span"][0], reverse=True)
    for finding in sorted_findings:
        start, end = finding["span"]
        replacement = {
            "mask": f"[REDACTED_{finding['type'].upper()}]",
            "generalize": generalize_value(finding),  # e.g. exact age -> age range
            "hash": f"[{hashlib.sha256(finding['value'].encode()).hexdigest()[:8]}]",  # consistent but irreversible
        }[strategy]
        text = text[:start] + replacement + text[end:]
    return text
```

`hash` is worth calling out specifically — it lets you preserve *consistency* (the same underlying value always redacts to the same token, useful for maintaining referential structure in a dataset) without preserving the actual sensitive value, useful for training data (May's series) where you want a model to learn patterns without memorizing real identifiers.

## Where to Apply This in the Pipeline

```python
pii_checkpoints = {
    "training_data_ingestion": "before any real user data enters a fine-tuning dataset (May's series)",
    "trace_logging": "before storage in observability platforms (June's series)",
    "long_term_memory_writes": "before any fact enters an agent's persistent memory (March's series)",
    "output_to_third_parties": "before agent output reaches an external system (this week's exfiltration post)",
}
```

Applying redaction at multiple checkpoints, not just one, matters because PII can enter a system through many paths — a golden set example (June) built from a real production incident needs the same scrutiny as training data.

## Balancing Redaction Against Utility

Over-aggressive redaction can destroy the information a feature actually needs — redacting a customer's name from a support ticket makes it harder for an agent to personalize a response appropriately. Define redaction policy per data flow, not a single blanket rule: full redaction for training data and long-term storage, more selective redaction (or none) for in-session context where the data is needed for the immediate task and won't persist.

## Testing Detection Coverage

Build a golden set of text containing known PII in varied, realistic phrasings (following June's evaluation discipline) and measure recall — missed PII is the failure mode that matters most here, since a false positive (over-redacting something benign) is a much smaller cost than a false negative letting real PII through.

## Compliance Connection

This pipeline is the direct technical implementation underlying the GDPR and HIPAA compliance posts later this month — "how do we actually find and handle PII in our systems" is the practical question those regulatory frameworks require an answer to.

---

*Part of the [AI Security series]({{ site.baseurl }}/tags/ai-security-series/) — next: [guardrails frameworks compared]({{ site.baseurl }}/posts/guardrails-frameworks-compared/), for teams wanting a packaged solution rather than hand-rolled detection like this.*
