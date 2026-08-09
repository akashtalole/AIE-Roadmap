---
title: "Containerizing LLM Applications for Production"
date: 2026-08-24 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, docker, kubernetes]
---

Every deployment pattern this month — autoscaling, blue-green, multi-region — assumes a containerized application underneath. This post covers the specific considerations for containerizing LLM applications well, which differ meaningfully from typical web-service containerization.

## The Layered Dockerfile Pattern

```dockerfile
FROM nvidia/cuda:12.4.0-runtime-ubuntu22.04 AS base
RUN apt-get update && apt-get install -y python3.11 python3-pip

FROM base AS dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM dependencies AS runtime
COPY app/ /app
WORKDIR /app
CMD ["python3", "serve.py"]
```

Separating the CUDA base layer, dependency installation, and application code into distinct layers means a code-only change rebuilds quickly (reusing the cached dependency layer), while a dependency change only rebuilds from that layer forward — meaningfully faster iteration than a flat single-stage Dockerfile, especially given how large GPU-enabled base images already are.

## Model Weights: Bake In or Mount at Runtime?

```dockerfile
# Option A: bake weights into the image — larger image, but self-contained and immutable
COPY models/llama-3-8b-instruct /app/models/

# Option B: mount at runtime — smaller image, but requires the weights to be available wherever it runs
VOLUME /app/models
```

Baking weights into the image produces a large image (tens of gigabytes) but guarantees exact reproducibility — the container is a complete, versioned artifact. Mounting at runtime keeps images smaller and lets multiple containers share one weights volume, at the cost of an external dependency on weight availability at deploy time. For most production deployments, mounting from fast object storage (with local caching) balances these tradeoffs better than baking in multi-gigabyte weights on every image build.

## GPU Access in Containers

```yaml
# docker-compose.yml
services:
  inference-server:
    image: my-vllm-server:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

Requires the NVIDIA Container Toolkit installed on the host — a common source of "works on my machine, fails in CI/production" issues when the toolkit isn't consistently present across environments, worth verifying explicitly as part of environment setup rather than assumed.

## Resource Limits Specific to GPU Workloads

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    memory: "24Gi"
    cpu: "4"
```

Unlike CPU/memory, GPU resources in Kubernetes are typically requested as whole units (you can't request "0.5 GPU" without specialized time-slicing configuration) — this affects bin-packing efficiency and is a real consideration for the capacity planning covered later this month.

## Health Checks That Actually Verify Model Readiness

```python
@app.get("/health")
async def health_check():
    if not model_loaded:
        return JSONResponse({"status": "loading"}, status_code=503)
    test_result = await run_quick_inference_check()
    if not test_result["success"]:
        return JSONResponse({"status": "unhealthy"}, status_code=503)
    return {"status": "healthy"}
```

A health check that only verifies the HTTP server is up, without verifying the model actually loaded and can run inference, will report healthy during the multi-minute model-loading window from the autoscaling post — leading a load balancer to route real traffic to a container that isn't actually ready yet.

## Image Size and Startup Time

Large images with baked-in weights directly affect the slow-scale-up problem from the autoscaling post — every second spent pulling a multi-gigabyte image is a second of unavailable capacity during a scale-up event. Layer caching, mounting weights externally, and image registry proximity to your compute (same region, ideally same availability zone) all measurably reduce this.

## Security Scanning for AI-Specific Images

Beyond standard container vulnerability scanning, verify model weight provenance (checksums against known-good sources) as part of the build pipeline — a supply-chain concern covered in depth in September's security series, worth building into the containerization process from the start rather than retrofitting later.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — next: [infrastructure as code with Terraform]({{ site.baseurl }}/posts/infrastructure-as-code-terraform-ai/), managing everything this month has covered declaratively.*
