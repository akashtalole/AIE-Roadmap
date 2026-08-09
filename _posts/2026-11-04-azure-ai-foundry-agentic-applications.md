---
title: "Azure AI Foundry for Agentic Applications"
date: 2026-11-04 08:00:00 +0530
categories: [AI, Cloud AI]
tags: [cloud-business-series, azure, agents, python]
mermaid: true
---

Yesterday's post covered Azure OpenAI's model access layer. Azure AI Foundry sits above it — Microsoft's platform specifically for building, evaluating, and deploying agentic applications, worth understanding as a distinct layer from raw model access.

## What Foundry Adds Beyond Raw Model Access

```python
foundry_capabilities = {
    "agent_service": "managed agent orchestration with built-in tool integration",
    "prompt_flow": "visual and code-based pipeline authoring, similar in spirit to Haystack's DAG model (Oct 10)",
    "evaluation": "built-in evaluators overlapping with June's evaluation series",
    "model_catalog": "access to Azure OpenAI models plus open-weight models from Hugging Face and others",
    "content_safety": "Azure's guardrails offering, comparable to Bedrock's",
}
```

## Building an Agent with Azure AI Agent Service

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient.from_connection_string(
    credential=DefaultAzureCredential(), conn_str=project_connection_string,
)

agent = project_client.agents.create_agent(
    model="gpt-4.1",
    name="support-agent",
    instructions="You help customers with order status and returns.",
    tools=[{"type": "function", "function": order_status_tool_schema}],
)

thread = project_client.agents.create_thread()
project_client.agents.create_message(thread_id=thread.id, role="user", content="Where's my order?")
run = project_client.agents.create_run(thread_id=thread.id, assistant_id=agent.id)
```

This follows the same thread-and-run model as OpenAI's Assistants API — persistent conversation threads with an agent definition attached, functionally similar to LangGraph's checkpointed sessions (October 1) but managed entirely server-side by Azure rather than requiring your own checkpoint store.

## Prompt Flow for Multi-Step Pipelines

```python
# Prompt Flow's DAG definition, conceptually similar to Haystack (Oct 10) or LangGraph (April)
flow_definition = {
    "nodes": [
        {"name": "retrieve", "type": "python", "source": "retrieve.py"},
        {"name": "generate", "type": "llm", "connection": "azure_openai_connection", "prompt": "generate_prompt.jinja2"},
    ],
    "connections": [{"from": "retrieve.output", "to": "generate.context"}],
}
```

Prompt Flow's visual pipeline builder is aimed at making this DAG-based pipeline construction accessible to a broader team beyond pure engineers — a genuinely different audience tradeoff than LangGraph's code-first approach, worth considering when your team includes non-engineers who need to iterate on pipeline logic.

## Built-In Evaluation

```python
from azure.ai.evaluation import RelevanceEvaluator, GroundednessEvaluator

relevance_eval = RelevanceEvaluator(model_config=azure_openai_config)
groundedness_eval = GroundednessEvaluator(model_config=azure_openai_config)

results = evaluate(
    data="golden_set.jsonl",
    evaluators={"relevance": relevance_eval, "groundedness": groundedness_eval},
)
```

`GroundednessEvaluator` is Azure's implementation of March's RAGAS faithfulness metric — worth calibrating against your own human-labeled examples (June's calibration discipline) rather than trusting any managed evaluator's scores blindly, the same caution that applies to any LLM-as-judge implementation regardless of vendor.

## Content Safety Integration

```python
from azure.ai.contentsafety import ContentSafetyClient

result = content_safety_client.analyze_text(text=user_input, categories=["Hate", "SelfHarm", "Violence", "Sexual"])
```

Azure's Content Safety service is directly comparable to September's guardrails framework comparison — worth benchmarking against Llama Guard, Guardrails AI, and hand-rolled moderation on your specific risk categories rather than defaulting to whichever cloud platform you're already using for model access.

## When Foundry's Managed Agent Layer Fits

Foundry's agent service earns its adoption when your organization wants Azure-native, low-code-friendly agent building with built-in evaluation and observability, particularly for teams that value Prompt Flow's visual authoring for cross-functional collaboration. For teams needing the deeper control and framework flexibility from April and October's deep dives, direct use of LangGraph, CrewAI, or AutoGen (which can still call Azure OpenAI as the underlying model) remains the more flexible choice.

---

*Part of the [Cloud AI Platforms & Business of AI series]({{ site.baseurl }}/tags/cloud-business-series/) — next: [Google Vertex AI]({{ site.baseurl }}/posts/google-vertex-ai-models-grounding-agents/), the third major cloud platform.*
