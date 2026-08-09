---
title: "Choosing GPUs for LLM Inference: A Practical Guide"
date: 2026-08-05 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, gpu, hardware]
---

Every inference server this week runs on top of a hardware decision that's easy to get wrong in either direction — over-provisioning wastes budget, under-provisioning caps throughput or forces unwanted quantization tradeoffs. This post covers the practical sizing math.

## The Two Numbers That Matter Most

- **Memory capacity** — determines what model size (and at what quantization level, from May's series) fits at all
- **Memory bandwidth** — determines inference speed, since LLM inference is typically memory-bandwidth-bound, not compute-bound

```python
def estimate_memory_needed(param_count_billions: float, precision_bytes: int, kv_cache_overhead_gb: float) -> float:
    weights_gb = param_count_billions * precision_bytes
    return weights_gb + kv_cache_overhead_gb + 2  # +2GB buffer for activation memory and overhead

# A 7B model in bf16 (2 bytes/param): 14GB weights + KV cache overhead
# The same model in 4-bit (0.5 bytes/param): 3.5GB weights + KV cache overhead
```

This is the direct hardware consequence of May's quantization post — a 4-bit quantized model doesn't just save memory in the abstract, it can be the difference between needing a single mid-range GPU and needing a multi-GPU setup for the same model.

## KV Cache Memory Scales With Concurrency

```python
def kv_cache_memory_gb(num_layers: int, hidden_dim: int, seq_len: int, batch_size: int, precision_bytes: int = 2) -> float:
    # Simplified: 2 (K and V) * layers * hidden_dim * seq_len * batch_size * precision_bytes
    return (2 * num_layers * hidden_dim * seq_len * batch_size * precision_bytes) / 1e9
```

This is why "does the model fit" isn't the only sizing question — the KV cache for many concurrent requests at long context lengths can dwarf the model weights themselves, directly determining how many concurrent requests (`max_num_seqs` from Sunday's vLLM post) a given GPU can actually serve.

## A Practical Sizing Table

| Model size | Precision | Approx. memory needed | Typical GPU tier |
|---|---|---|---|
| 7-8B | bf16 | ~16-20GB | Single mid-range GPU (16-24GB VRAM) |
| 7-8B | 4-bit | ~5-8GB | Single entry-level GPU |
| 13-14B | bf16 | ~28-32GB | Single high-end GPU (24-48GB VRAM) |
| 70B | bf16 | ~140GB+ | Multi-GPU |
| 70B | 4-bit | ~35-40GB | Single high-end GPU or dual mid-range |

## Cloud GPU vs Owned Hardware

```python
def build_vs_own_decision(monthly_gpu_hours_needed: float, cloud_hourly_rate: float, owned_gpu_cost: float, amortization_months: int = 24) -> dict:
    cloud_monthly_cost = monthly_gpu_hours_needed * cloud_hourly_rate
    owned_monthly_cost = owned_gpu_cost / amortization_months
    return {"cloud": cloud_monthly_cost, "owned": owned_monthly_cost, "breakeven_months": owned_gpu_cost / cloud_monthly_cost}
```

For variable or unpredictable load, cloud GPU rental avoids both the capital outlay and the risk of hardware sitting idle during low-traffic periods. For sustained, predictable high utilization, owned or reserved-instance hardware can be meaningfully cheaper over time — run this comparison with your actual projected utilization, not assumed constant peak load.

## Multi-GPU Considerations

Serving a model too large for a single GPU requires tensor or pipeline parallelism, splitting the model's layers or attention heads across GPUs — this adds inter-GPU communication overhead and needs high-bandwidth interconnects (NVLink) to avoid becoming the new bottleneck. Prefer fitting on a single GPU via quantization when quality allows it; multi-GPU serving is a real engineering step up in operational complexity.

## Benchmarking Before Committing

Provisioning decisions should be validated against your actual model and workload with a real benchmark run, not spec-sheet numbers alone — the same "measure, don't assume" discipline from June's evaluation series, applied to hardware sizing rather than model quality.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — next: [the KV cache explained]({{ site.baseurl }}/posts/kv-cache-explained-throughput/) in full depth, since it's been referenced all week without a complete explanation yet.*
