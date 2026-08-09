---
title: "Langfuse: Open-Source LLM Observability"
date: 2026-06-14 08:00:00 +0530
categories: [AI, Evaluation]
tags: [evaluation, evaluation-series, langfuse, observability, python]
---

LangSmith is tightly coupled to the LangChain ecosystem and is closed-source, hosted-first. Langfuse covers similar ground — tracing, evaluation, prompt management — as an open-source, self-hostable alternative, relevant if data residency or vendor lock-in are concerns for your organization.

## Instrumenting a Call

```python
from langfuse.decorators import observe
from langfuse.openai import openai  # drop-in wrapped client captures traces automatically

@observe()
def handle_support_request(message: str) -> str:
    response = openai.chat.completions.create(
        model="gpt-4.1",
        messages=[{"role": "user", "content": message}],
    )
    return response.choices[0].message.content
```

The `@observe()` decorator plus a wrapped provider client is the whole instrumentation cost for basic tracing — comparable ergonomics to LangSmith's `@traceable`, without requiring a LangChain dependency.

## Self-Hosting for Data Residency

```yaml
# docker-compose.yml excerpt
services:
  langfuse:
    image: langfuse/langfuse:latest
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/langfuse
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
    ports:
      - "3000:3000"
```

Self-hosting means trace data — which can include sensitive user input and model output — never leaves your own infrastructure, directly relevant to the compliance considerations covered later in September's AI security series for regulated industries.

## Prompt Management

Langfuse includes versioned prompt management as a first-class feature, letting you edit and version prompts outside your codebase and fetch them at runtime:

```python
prompt = langfuse.get_prompt("support-agent-system-prompt", version=4)
response = generate(system_prompt=prompt.compile(product_name="Acme Cloud"), user_input=message)
```

This decouples prompt iteration from code deploys — a non-engineer on the team can update a prompt version without a PR, while the version history still gives you the rollback and audit trail a code-managed prompt would have.

## Evaluation Integration

```python
from langfuse import Langfuse
langfuse = Langfuse()

def score_and_log(trace_id: str, response: str, context: dict):
    score = measure_faithfulness(response, context)
    langfuse.score(trace_id=trace_id, name="faithfulness", value=score)
```

Scores attached directly to traces make it possible to filter and analyze production traffic by quality score in the same dashboard used for latency and cost — "show me all low-faithfulness traces from the last 24 hours" becomes a direct query rather than a separate offline analysis.

## Choosing Between Langfuse, LangSmith, and Rolling Your Own

Langfuse is the strongest default when you want observability decoupled from a specific agent framework and value self-hosting; LangSmith is strongest if you're already deep in the LangChain/LangGraph ecosystem and want the tightest integration; a hand-rolled solution (this month's earlier posts) makes sense only at very small scale or with genuinely unusual requirements neither platform covers well.

---

*Part of the [Evaluation series]({{ site.baseurl }}/tags/evaluation-series/) — next: [Arize Phoenix]({{ site.baseurl }}/posts/arize-phoenix-tracing-debugging/), with a stronger focus on ML-style drift and embedding-level debugging.*
