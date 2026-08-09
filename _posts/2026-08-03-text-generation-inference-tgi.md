---
title: "Deploying Models with Text Generation Inference (TGI)"
date: 2026-08-03 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, tgi, huggingface, docker]
---

TGI, Hugging Face's inference server, solves the same core problems as vLLM — continuous batching, efficient memory management — with tighter integration into the Hugging Face ecosystem and a Docker-first deployment model worth comparing directly rather than assuming interchangeability.

## Deploying with Docker

```bash
docker run --gpus all --shm-size 1g -p 8080:80 \
  -v $PWD/models:/data \
  ghcr.io/huggingface/text-generation-inference:latest \
  --model-id meta-llama/Llama-3-8b-Instruct \
  --max-total-tokens 4096 \
  --max-batch-total-tokens 16384
```

The Docker-first packaging is TGI's most distinctive practical difference from vLLM — a single container image with sensible defaults, versus vLLM's Python-library-first approach (though vLLM also ships containers). For teams already standardized on containerized deployment pipelines, this reduces integration friction.

## The API

```python
import requests

response = requests.post(
    "http://localhost:8080/generate",
    json={"inputs": "Explain KV caching in one sentence.", "parameters": {"max_new_tokens": 100, "temperature": 0.7}},
)
print(response.json()["generated_text"])
```

TGI also exposes an OpenAI-compatible endpoint (`/v1/chat/completions`) for the same drop-in replacement pattern as vLLM — worth using directly for consistency with existing application code, rather than TGI's native endpoint shape, unless you need TGI-specific parameters the compatible endpoint doesn't expose.

## Native Quantization Support

```bash
--quantize bitsandbytes-nf4  # or gptq, awq, eetq
```

TGI's quantization support ties directly back into May's fine-tuning series quantization post — running a QLoRA-quantized or GPTQ-quantized model in TGI is a straightforward flag, making it a natural serving target for models fine-tuned using the techniques from that series.

## Guidance and Structured Generation

TGI includes built-in support for constraining output to a JSON schema server-side, rather than relying purely on prompt instructions:

```python
response = requests.post(
    "http://localhost:8080/generate",
    json={
        "inputs": "Extract the invoice total.",
        "parameters": {"grammar": {"type": "json", "value": invoice_schema}},
    },
)
```

This is a stronger guarantee than the prompt-and-validate pattern from the structured-output posts earlier in this roadmap — the server itself constrains token generation to only produce schema-valid output, eliminating the retry-on-validation-failure loop entirely for supported cases.

## TGI vs vLLM: Choosing Between Them

Both are strong, actively maintained options solving the same core problem — the practical decision often comes down to ecosystem fit (Hugging Face Hub integration favors TGI, broader model architecture support and community momentum currently favors vLLM) and specific feature needs (TGI's built-in grammar constraints, vLLM's broader quantization and hardware backend support). Benchmark both against your specific model and workload before committing, following the same task-specific evaluation discipline as any other infrastructure choice in this roadmap.

## Monitoring TGI in Production

TGI exposes Prometheus-compatible metrics out of the box — queue depth, batch size, time-to-first-token — feeding directly into the dashboard patterns from June's observability series, now at the infrastructure layer rather than the application layer.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — next: [NVIDIA Triton]({{ site.baseurl }}/posts/nvidia-triton-inference-server/), for teams needing multi-framework model serving beyond just LLMs.*
