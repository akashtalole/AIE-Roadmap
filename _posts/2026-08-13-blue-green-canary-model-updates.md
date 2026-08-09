---
title: "Blue-Green and Canary Deployments for Model Updates"
date: 2026-08-13 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, deployment, canary]
mermaid: true
---

June's canary release post covered this at the application layer — rolling out a prompt or model change gradually with quality guardrails. This post covers the same problem at the infrastructure layer: swapping the actual serving deployment underneath, with zero-downtime as the added constraint.

## Blue-Green: Two Complete Environments

```mermaid
flowchart LR
    A[Load balancer] -->|100% traffic| B[Blue: current version]
    C[Green: new version] -.deployed, not yet receiving traffic.-> A
```

Deploy the new model version as a fully separate environment (green) alongside the current one (blue), verify it's healthy, then switch the load balancer to route 100% of traffic to green atomically. If anything's wrong, switch back to blue instantly — no redeployment needed for rollback, just a routing change.

```python
def blue_green_switch(load_balancer, new_environment: str):
    verify_health(new_environment, min_healthy_checks=5)
    load_balancer.switch_target(new_environment)
    monitor_for_regressions(new_environment, window_minutes=10)
    if detected_regression():
        load_balancer.switch_target(get_previous_environment())
```

The instant-rollback property is blue-green's main advantage over a rolling deployment — rollback is a routing change, not a redeploy, which matters enormously for GPU workloads given yesterday's post on slow GPU node provisioning.

## Canary: Gradual Traffic Shift, Not All-at-Once

```python
canary_stages = [5, 25, 50, 100]  # percent of traffic

def canary_deploy(new_version: str, stages: list[int]):
    for pct in stages:
        shift_traffic_percentage(new_version, pct)
        wait_and_monitor(duration_minutes=15)
        if guardrails_violated(new_version):
            rollback_immediately(new_version)
            return "rolled_back"
    return "fully_deployed"
```

This is the same canary discipline from June's application-layer post, applied at the infrastructure/model-serving layer — the guardrails here should include the infrastructure-specific metrics from this week (latency, throughput, GPU utilization) alongside the quality metrics from June.

## Blue-Green vs Canary: Resource Cost Tradeoff

Blue-green requires running two full environments simultaneously during the transition — double the GPU capacity temporarily, which for expensive GPU infrastructure is a real cost consideration absent from typical CPU-service blue-green deployments. Canary avoids this by shifting traffic gradually within roughly the same total capacity, at the cost of a slower, more gradual rollout.

```python
def estimate_blue_green_cost_overhead(current_gpu_cost_per_hour: float, transition_duration_hours: float) -> float:
    return current_gpu_cost_per_hour * transition_duration_hours  # the "double capacity" period cost
```

## Model-Specific Rollback Considerations

Unlike a typical code rollback, rolling back a model version can also mean invalidating any in-flight state that assumed the new model's behavior — check for compatibility issues with cached responses (August's semantic caching post) or any fine-tuned adapter (May's series) that was trained against a specific base model version before assuming rollback is fully clean.

```python
def safe_rollback(previous_version: str, current_version: str):
    if adapter_incompatible(current_version, previous_version):
        invalidate_dependent_adapters(current_version)
    invalidate_cache_entries_from_version(current_version)
    switch_to_version(previous_version)
```

## Automating the Decision, Not Just the Mechanics

The real value of either pattern comes from wiring automatic rollback to the guardrail metrics, not from the deployment mechanics alone — a canary or blue-green deployment with only manual monitoring provides much less protection than one with the automated guardrail-triggered rollback from June's canary post, now applied consistently at both the application and infrastructure layers.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — next: [disaster recovery planning]({{ site.baseurl }}/posts/disaster-recovery-planning-ai-services/) for when rollback alone isn't enough.*
