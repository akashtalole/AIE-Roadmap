---
title: "LangGraph Deep Dive: Custom Checkpointing and Persistence"
date: 2026-10-01 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [langgraph, deep-dive-series, python, state-management]
---

April's LangGraph posts used the default `MemorySaver` checkpointer. Production deployments need durable, queryable persistence — this post covers building a custom checkpointer backed by a real database, and the patterns that come with it.

## Implementing a Custom Checkpointer

```python
from langgraph.checkpoint.base import BaseCheckpointSaver, Checkpoint, CheckpointMetadata

class PostgresCheckpointer(BaseCheckpointSaver):
    def __init__(self, connection_pool):
        self.pool = connection_pool

    async def aput(self, config: dict, checkpoint: Checkpoint, metadata: CheckpointMetadata) -> dict:
        thread_id = config["configurable"]["thread_id"]
        async with self.pool.acquire() as conn:
            await conn.execute(
                "INSERT INTO checkpoints (thread_id, checkpoint_id, data, metadata, created_at) "
                "VALUES ($1, $2, $3, $4, $5) ON CONFLICT (thread_id, checkpoint_id) DO UPDATE SET data = $3",
                thread_id, checkpoint["id"], serialize(checkpoint), serialize(metadata), now(),
            )
        return config

    async def aget(self, config: dict) -> Checkpoint | None:
        thread_id = config["configurable"]["thread_id"]
        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT data FROM checkpoints WHERE thread_id = $1 ORDER BY created_at DESC LIMIT 1", thread_id
            )
            return deserialize(row["data"]) if row else None
```

This directly implements the durable-state requirement from April's state-management post — a Postgres-backed checkpointer survives process restarts and horizontal scaling, unlike the in-memory default that only works within a single process's lifetime.

## Checkpoint History and Time Travel

```python
async def get_checkpoint_history(thread_id: str, limit: int = 20) -> list[Checkpoint]:
    async with pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT data FROM checkpoints WHERE thread_id = $1 ORDER BY created_at DESC LIMIT $2", thread_id, limit
        )
        return [deserialize(row["data"]) for row in rows]

async def resume_from_specific_checkpoint(thread_id: str, checkpoint_id: str):
    config = {"configurable": {"thread_id": thread_id, "checkpoint_id": checkpoint_id}}
    return await app.ainvoke(None, config=config)  # resumes from that specific historical point, not just the latest
```

Storing every checkpoint, not just the latest, enables "time travel" debugging — resuming a graph from any historical point in its execution, invaluable for reproducing and diagnosing a bug that occurred several steps into a long-running workflow, directly extending June's debugging discipline to stateful graph execution.

## Checkpoint Pruning and Retention

```python
async def prune_old_checkpoints(retention_days: int = 30):
    cutoff = now() - timedelta(days=retention_days)
    async with pool.acquire() as conn:
        await conn.execute(
            "DELETE FROM checkpoints WHERE created_at < $1 AND thread_id NOT IN (SELECT thread_id FROM active_threads)",
            cutoff,
        )
```

Unbounded checkpoint accumulation becomes a real storage cost at scale — apply September's data retention discipline directly here, keeping full history for active threads and pruning completed ones past a reasonable retention window.

## Encrypting Sensitive Checkpoint Data

```python
def serialize(checkpoint: Checkpoint) -> bytes:
    raw = json.dumps(checkpoint).encode()
    return encrypt(raw, key=get_checkpoint_encryption_key())
```

Since a checkpoint contains the full conversation and tool-call state — potentially including PII — apply September's encryption and access-control discipline to the checkpoint store itself, not just to the application-layer database it's separate from.

## Multi-Region Checkpoint Replication

For the multi-region deployments from August's infrastructure series, checkpoint storage needs its own replication strategy — a session that started in one region and needs to resume in another (during a regional failover) requires the checkpoint store itself to be replicated or centrally accessible, not siloed per-region.

## Testing Checkpoint Recovery

```python
async def test_checkpoint_recovery():
    config = {"configurable": {"thread_id": "test-recovery"}}
    partial_result = await app.ainvoke(initial_state, config=config, interrupt_after=["step_2"])
    simulate_process_restart()
    resumed_result = await app.ainvoke(None, config=config)
    assert resumed_result == expected_full_result
```

Testing recovery explicitly — not just assuming a checkpointer works because it compiles — is worth treating as a required test alongside the zero-downtime deployment tests from August, since a checkpointing bug only surfaces during an actual restart or failover, exactly the worst time to discover it.

---

*Part of the [Agentic Framework Deep Dives series]({{ site.baseurl }}/tags/deep-dive-series/) — following September's [AI Security series]({{ site.baseurl }}/tags/ai-security-series/). Next: [subgraphs and multi-agent graphs]({{ site.baseurl }}/posts/langgraph-deep-dive-subgraphs-multi-agent/).*
