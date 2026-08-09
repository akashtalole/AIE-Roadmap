---
title: "Code-Generating Agents: Sandboxing and Execution Safety"
date: 2026-04-23 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, agentic-frameworks-series, sandboxing, security, python]
---

AutoGen's `code_execution_config` from earlier this month treated code execution as a first-class tool. That's genuinely powerful — a model writing and running code can solve problems no fixed toolset anticipated — and genuinely dangerous if the execution environment isn't properly isolated.

## Never Execute Generated Code on Your Host

This should go without saying, but it's worth being explicit: `exec()` or `subprocess.run()` on raw model output, on the same machine running your application, is a direct path to arbitrary code execution by anything that can influence the prompt — including, transitively, retrieved documents or user input the model incorporates into the code it writes.

## Sandboxing Options, Weakest to Strongest

```python
# Weakest — no isolation, never do this in production
exec(generated_code)

# Better — a subprocess with resource limits, still shares the kernel
subprocess.run(["python3", "-c", generated_code], timeout=10, resource_limits=...)

# Strong — a container with no network, minimal filesystem, dropped capabilities
docker_client.containers.run(
    "python:3.12-slim", command=["python3", "-c", generated_code],
    network_disabled=True, mem_limit="256m", cpu_period=100000, cpu_quota=50000,
    read_only=True, remove=True, timeout=10,
)

# Strongest — a microVM (Firecracker, gVisor) for full kernel-level isolation
```

Container isolation with network disabled and a short timeout covers the large majority of legitimate use cases — data analysis, calculations, quick scripts. Reach for microVM-level isolation only when the agent runs genuinely untrusted, adversarial input at scale.

## What to Restrict Beyond the Sandbox Boundary

- **No network access** unless the specific task requires it, and if it does, allowlist specific destinations rather than opening it fully
- **No filesystem access** beyond a scratch directory that's wiped after each run
- **Hard timeouts and memory limits** — a generated infinite loop or memory leak shouldn't be able to exhaust host resources
- **No secrets in the execution environment** — API keys and credentials the agent's *other* tools use should never be reachable from inside the code sandbox

## Reviewing Output, Not Just Isolating Execution

Sandboxing prevents the code from damaging anything outside itself, but it doesn't guarantee the code is *correct*. Feed execution errors back to the model as an observation and let it retry, capped at a small number of attempts:

```python
def run_with_retry(code: str, max_attempts: int = 3) -> str:
    for attempt in range(max_attempts):
        result = execute_in_sandbox(code)
        if result.exit_code == 0:
            return result.stdout
        code = fix_code(code, error=result.stderr)  # ask the model to fix its own error
    raise CodeExecutionFailed(f"Failed after {max_attempts} attempts")
```

## The Line Between "Powerful" and "Reckless"

Code-generating agents solve a real class of problems — data wrangling, one-off calculations, format conversions — that a fixed toolset can't anticipate. The engineering discipline required is proportional to that power: every one of the isolation layers above exists because someone shipped without it and found out why the hard way.

---

*Part of the [Agentic Frameworks series]({{ site.baseurl }}/tags/agentic-frameworks-series/) — next: [browser-using agents]({{ site.baseurl }}/posts/browser-using-agents-web-automation/), which carry the same sandboxing discipline into a different domain.*
