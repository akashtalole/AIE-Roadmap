---
title: "Serverless AI: Running LLM Workloads on Lambda and Cloud Functions"
date: 2026-11-10 08:00:00 +0530
categories: [AI, Cloud AI]
tags: [cloud-business-series, serverless, python]
---

August's infrastructure series assumed always-on servers (containers, Kubernetes). For many LLM application workloads — API-calling logic, not self-hosted model inference — serverless functions are a genuinely simpler, often cheaper alternative worth understanding explicitly.

## What Serverless Fits Well for AI Workloads

```python
serverless_appropriate_workloads = {
    "api_gateway_endpoint": "a request handler that calls a model provider API and returns — no local inference",
    "async_processing": "document processing triggered by an S3 upload (July's document pipeline, event-triggered)",
    "webhook_handlers": "responding to A2A task callbacks or MCP server events",
    "scheduled_batch_jobs": "nightly evaluation runs (June's series), scheduled agent tasks",
}
```

Since these workloads call out to a model provider's API rather than running inference locally, they don't need the GPU infrastructure from August's series at all — a serverless function calling the Anthropic or OpenAI API is functionally identical in resource needs to any other lightweight API-calling service.

## A Lambda Handler for an LLM-Backed Endpoint

```python
import json
import anthropic

client = anthropic.Anthropic()

def lambda_handler(event, context):
    body = json.loads(event["body"])
    response = client.messages.create(
        model="claude-sonnet-5", max_tokens=1024,
        messages=[{"role": "user", "content": body["prompt"]}],
    )
    return {"statusCode": 200, "body": json.dumps({"response": response.content[0].text})}
```

## Cold Start Considerations

```python
def minimize_cold_start_impact():
    # Initialize the client outside the handler — reused across warm invocations
    return anthropic.Anthropic()  # module-level, not inside lambda_handler

client = minimize_cold_start_impact()
```

Cold starts (the delay when a serverless function spins up from zero) are a real latency concern for interactive use cases — following June's latency-budget discipline, measure cold-start-inclusive P99 latency specifically, and consider provisioned concurrency (paying to keep a minimum number of instances warm) for latency-sensitive endpoints, echoing August's "never scale to zero for latency-sensitive workloads" principle applied to serverless specifically.

## Timeout Constraints and Long-Running Agents

```python
lambda_config = {"timeout_seconds": 900}  # AWS Lambda's maximum, as of current limits
```

Serverless functions have hard maximum execution time limits — directly conflicting with October's long-running agent patterns (Temporal, durable workflows) for anything exceeding that limit. The practical pattern: use serverless for the individual, bounded steps of a longer workflow, with a separate durable-execution layer (Temporal, or a workflow engine per October's posts) orchestrating across multiple serverless function invocations for anything that might run longer.

```python
@activity.defn
async def call_llm_lambda_activity(prompt: str) -> str:
    return await invoke_lambda_function("llm-handler", {"prompt": prompt})  # bounded, single Lambda call
```

## Cost Comparison: Serverless vs Always-On

```python
def serverless_vs_container_cost(requests_per_month: int, avg_duration_ms: float) -> dict:
    serverless_cost = requests_per_month * (avg_duration_ms / 1000) * LAMBDA_GB_SECOND_RATE
    container_cost = ALWAYS_ON_CONTAINER_MONTHLY_COST  # fixed regardless of traffic
    return {"serverless": serverless_cost, "always_on": container_cost,
            "better_choice": "serverless" if serverless_cost < container_cost else "always_on"}
```

Serverless pricing (pay per invocation and duration) tends to win for spiky or low-volume traffic; an always-on container (August's series) tends to win once volume is high and consistent enough that the fixed cost is regularly fully utilized — run this comparison against your actual traffic pattern, following June's cost-quality evaluation discipline applied to infrastructure choice.

## Concurrency Limits and Rate Limiting Interaction

```python
def respect_provider_limits_in_serverless(concurrent_executions: int, provider_rate_limit: int):
    if concurrent_executions > provider_rate_limit:
        raise ConfigurationError("Lambda concurrency exceeds provider rate limit — configure reserved concurrency")
```

A serverless function can scale out very quickly under load — potentially faster than a model provider's rate limits can absorb, unlike August's more gradual container-based autoscaling. Explicitly capping concurrent executions (reserved concurrency in AWS terms) to stay within provider rate limits is a serverless-specific consideration worth configuring deliberately, not discovering during a traffic spike.

## Observability for Serverless AI Functions

Apply June's tracing and cost-attribution discipline directly — serverless functions integrate with cloud-native tracing (AWS X-Ray, Azure Application Insights) that can feed the same dashboards, with the added benefit that per-invocation cost is naturally granular and easy to attribute per August's cost-attribution post.

---

*Part of the [Cloud AI Platforms & Business of AI series]({{ site.baseurl }}/tags/cloud-business-series/) — next: [cost modeling for AI products]({{ site.baseurl }}/posts/cost-modeling-ai-products-framework/), shifting from infrastructure to the business layer.*
