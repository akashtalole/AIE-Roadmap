---
title: "CrewAI Deep Dive: Custom Tools and Memory Backends"
date: 2026-10-03 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [crewai, deep-dive-series, python, memory]
---

April's CrewAI posts covered basics and hierarchical processes. This post goes deeper into two areas that matter for production use: building genuinely custom tools beyond simple functions, and configuring CrewAI's memory system to persist across sessions rather than resetting every run.

## Building a Custom Tool Class

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field

class DatabaseQueryInput(BaseModel):
    customer_id: str = Field(description="The customer ID to look up")

class CustomerLookupTool(BaseTool):
    name: str = "customer_lookup"
    description: str = "Look up a customer's account details by ID"
    args_schema: type[BaseModel] = DatabaseQueryInput

    def _run(self, customer_id: str) -> str:
        record = db.get_customer(customer_id)
        return json.dumps(record) if record else "Customer not found"
```

Subclassing `BaseTool` instead of using a plain decorated function gives you a strongly-typed `args_schema` (directly implementing April's tool-design principles on argument constraints) and a place to add tool-level logic like caching or rate limiting that a simple function wrapper doesn't cleanly support.

## Configuring Memory Backends

```python
from crewai import Crew
from crewai.memory import LongTermMemory, ShortTermMemory, EntityMemory
from crewai.memory.storage import RAGStorage

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    memory=True,
    long_term_memory=LongTermMemory(storage=RAGStorage(embedder_config={"provider": "openai"})),
    short_term_memory=ShortTermMemory(storage=RAGStorage(embedder_config={"provider": "openai"})),
    entity_memory=EntityMemory(storage=RAGStorage(embedder_config={"provider": "openai"})),
)
```

CrewAI's three memory types map directly onto March's memory taxonomy — `short_term_memory` is working memory within a single crew run, `long_term_memory` persists insights across separate runs, and `entity_memory` specifically tracks facts about recurring entities (customers, projects) the crew encounters repeatedly.

## Custom Memory Storage Backends

```python
from crewai.memory.storage.base_rag_storage import BaseRAGStorage

class PostgresRAGStorage(BaseRAGStorage):
    def save(self, value: str, metadata: dict):
        embedding = embed(value)
        db.execute("INSERT INTO crew_memory (content, embedding, metadata) VALUES (%s, %s, %s)",
                   (value, embedding, json.dumps(metadata)))

    def search(self, query: str, limit: int = 5) -> list[dict]:
        query_embedding = embed(query)
        return db.query_similar("crew_memory", query_embedding, limit)
```

CrewAI's default storage is local-disk-based, fine for experimentation but not suitable for the multi-instance, durable deployment patterns from August's infrastructure series — a custom Postgres or dedicated vector-store-backed implementation is what production deployment actually requires.

## Custom Tool Error Handling and Retries

```python
class ResilientAPITool(BaseTool):
    name: str = "external_api_call"
    description: str = "Calls an external pricing API"

    def _run(self, product_id: str) -> str:
        for attempt in range(3):
            try:
                return call_external_api(product_id)
            except TransientError:
                time.sleep(2 ** attempt)
        return "Unable to retrieve pricing after multiple attempts — please try again later"
```

Building retry logic and graceful degradation directly into the tool implementation, rather than relying on the agent to handle a raw exception gracefully, produces more reliable crews — the agent should see a clean, informative failure message it can reason about, not a raw stack trace.

## Testing Custom Tools in Isolation

```python
def test_customer_lookup_tool():
    tool = CustomerLookupTool()
    result = tool._run(customer_id="12345")
    assert "error" not in result.lower()
    parsed = json.loads(result)
    assert parsed["id"] == "12345"
```

Applying April's tool-testing-in-isolation principle directly — a `BaseTool` subclass's `_run` method is a plain function call from a test's perspective, testable with the same rigor as any other piece of application logic, independent of whether the agent ever calls it correctly.

## Memory Privacy Considerations

Given September's security series, any persistent memory storing entity information about customers needs the same PII and retention discipline as any other data store — apply access scoping (memory shouldn't leak across tenants in a multi-tenant deployment) and retention policy consistently with the rest of the system.

---

*Part of the [Agentic Framework Deep Dives series]({{ site.baseurl }}/tags/deep-dive-series/) — next: [CrewAI Flows for deterministic control]({{ site.baseurl }}/posts/crewai-flows-deterministic-control/), when you need more precision than the standard process types offer.*
