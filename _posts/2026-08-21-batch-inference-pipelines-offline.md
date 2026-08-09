---
title: "Batch Inference Pipelines for Offline Workloads"
date: 2026-08-21 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, batch-processing, python]
mermaid: true
---

Nearly every post this month has optimized for interactive, real-time serving. Not every workload needs that — the document-processing pipeline from July, a nightly re-embedding job, a bulk content classification run — and treating these as real-time requests wastes both engineering effort and money.

## Why Batch Workloads Need a Different Architecture

```mermaid
flowchart LR
    A[Job queue: 100k documents] --> B[Batch scheduler]
    B --> C[Large batch size, no latency constraint]
    C --> D[Maximum throughput inference]
    D --> E[Results written to storage]
```

An interactive request needs a response in under a few seconds. A batch job processing 100,000 documents overnight has hours of budget — that slack is exactly what lets batch inference use much larger batch sizes and accept much higher per-request latency in exchange for dramatically better cost-per-token throughput.

## Using Provider Batch APIs

Most major providers offer a dedicated batch API tier at a meaningful cost discount, in exchange for processing within a longer window (commonly same-day, not real-time):

```python
def submit_batch_job(requests: list[dict]) -> str:
    batch_file = create_batch_input_file(requests)
    batch = client.batches.create(input_file_id=batch_file.id, endpoint="/v1/chat/completions", completion_window="24h")
    return batch.id

def poll_batch_status(batch_id: str) -> dict:
    status = client.batches.retrieve(batch_id)
    if status.status == "completed":
        return download_batch_results(status.output_file_id)
    return {"status": status.status}
```

This is often the single highest-leverage cost optimization available for any workload from earlier in this roadmap that doesn't need real-time response — the document extraction pipeline (July), synthetic data generation (May), and golden-set evaluation runs (June) are all natural fits for batch API pricing.

## Self-Hosted Batch Inference

For self-hosted deployments, vLLM's offline batch inference mode maximizes throughput without the request-serving overhead of the API server:

```python
from vllm import LLM, SamplingParams

llm = LLM(model="meta-llama/Llama-3-8b-Instruct", max_num_seqs=1024)  # much larger than typical serving config
prompts = load_all_prompts_for_batch_job()
outputs = llm.generate(prompts, SamplingParams(temperature=0, max_tokens=500))
```

The `max_num_seqs=1024` here versus the more conservative values from Sunday's dynamic batching post reflects the different priority — throughput, not per-request latency, since nobody's watching a batch job in real time.

## Job Orchestration and Failure Handling

```python
def run_batch_pipeline(job_id: str, items: list[dict]):
    checkpoint = load_checkpoint(job_id)  # resume from where a previous run left off
    remaining = [item for item in items if item["id"] not in checkpoint["completed"]]
    for chunk in chunked(remaining, 100):
        results = process_chunk(chunk)
        save_checkpoint(job_id, completed_ids=[r["id"] for r in results])
        store_results(results)
```

The same idempotent, checkpointed processing pattern from July's document AI pipeline applies directly here — a batch job processing millions of items needs to survive a mid-run failure without restarting from zero, exactly the state-management discipline from April's agent posts applied at pipeline scale.

## Cost Comparison: Batch vs Real-Time Serving

```python
def batch_vs_realtime_savings(item_count: int, tokens_per_item: int, realtime_rate: float, batch_rate: float) -> float:
    realtime_cost = item_count * tokens_per_item * realtime_rate
    batch_cost = item_count * tokens_per_item * batch_rate
    return realtime_cost - batch_cost
```

Run this comparison explicitly for any workload currently using real-time serving that doesn't actually need real-time latency — it's a common and easy-to-miss cost optimization, since the same application code path is often reused for both interactive and background use cases without anyone revisiting whether batch pricing applies.

## When Batch Processing Isn't the Right Fit

For workloads needing intermediate results to drive further processing decisions in real time (an agent that needs a tool result before its next reasoning step), batch inference's latency isn't compatible — batch is specifically for workloads where the full input set is known upfront and results aren't needed until the whole batch completes.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — next: [serving multiple LoRA adapters]({{ site.baseurl }}/posts/serving-multiple-lora-adapters-scale/) from one base model at production scale, extending May's serving post.*
