---
title: "Knowledge Graphs for Agent Reasoning"
date: 2026-10-20 08:00:00 +0530
categories: [AI, Agentic Frameworks]
tags: [agents, deep-dive-series, knowledge-graphs, python]
mermaid: true
---

Yesterday's comparison established when a knowledge graph is the right tool. This post covers actually building one and giving an agent the ability to query and reason over it.

## Extracting a Knowledge Graph from Unstructured Text

{% raw %}
```python
class Entity(BaseModel):
    name: str
    type: Literal["person", "organization", "project", "date"]

class Relationship(BaseModel):
    source: str
    relation: str
    target: str

def extract_graph_from_text(text: str) -> dict:
    response = llm.chat([{
        "role": "user",
        "content": f"Extract entities and relationships from this text as JSON:\n{text}\n"
                    f"Entities: [{{name, type}}]. Relationships: [{{source, relation, target}}]."
    }], temperature=0)
    return json.loads(response.content)
```
{% endraw %}

This is the practical, LLM-based approach to the extraction problem yesterday's post flagged as knowledge graphs' main setup cost — using a model to extract structured entities and relationships from text, rather than a purely rule-based NLP pipeline, trading some accuracy for dramatically less upfront engineering.

## Storing the Graph

```python
def store_graph_facts(entities: list[Entity], relationships: list[Relationship]):
    for entity in entities:
        graph_db.merge_node(entity.name, labels=[entity.type])
    for rel in relationships:
        graph_db.merge_edge(rel.source, rel.target, relation_type=rel.relation)
```

Using `merge` rather than `create` matters — entities and relationships extracted from many documents over time need deduplication against existing graph nodes (the same entity mentioned in different documents should become one node, not duplicates), which most graph databases support natively via merge/upsert semantics.

## Querying the Graph for Multi-Hop Reasoning

{% raw %}
```python
def query_multi_hop(start_entity: str, max_hops: int = 2) -> list[dict]:
    return graph_db.query(f"""
        MATCH path = (start {{name: $start_entity}})-[*1..{max_hops}]-(connected)
        RETURN path
    """, start_entity=start_entity)
```
{% endraw %}

This is the capability yesterday's post identified as the graph's core advantage — traversing explicit relationships up to N hops away from a starting entity, answering exactly the "who does the manager of X report to" class of question a vector store structurally can't handle.

## Giving an Agent Graph-Query Tools

```python
def build_graph_query_tool() -> dict:
    return {
        "name": "query_knowledge_graph",
        "description": "Query relationships between entities. Use for questions about how things connect "
                       "(reporting structures, project ownership, dependencies), not general fact lookup.",
        "function": lambda entity, max_hops=2: query_multi_hop(entity, max_hops),
    }
```

Exposing graph queries as an agent tool follows exactly April's tool-design principles — the description explicitly scopes when to use this tool versus general retrieval, preventing the model from reaching for graph queries on tasks better served by simple vector search.

## Handling Entity Disambiguation

```python
def disambiguate_entity(mention: str, context: str) -> str | None:
    candidates = graph_db.fuzzy_search_nodes(mention)
    if len(candidates) == 1:
        return candidates[0].id
    if len(candidates) > 1:
        return resolve_with_context(mention, candidates, context)  # LLM-assisted disambiguation
    return None  # no match — might be a new entity
```

Yesterday's post flagged disambiguation as a knowledge graph weakness — "Alice" mentioned in a new document might refer to an existing graph node or a genuinely new person, and getting this wrong either creates duplicate nodes or incorrectly merges two different people's information, a real accuracy risk worth explicit handling rather than naive exact-string matching.

## Keeping the Graph Current

```python
async def incremental_graph_update(new_document: str):
    extracted = extract_graph_from_text(new_document)
    for entity in extracted["entities"]:
        entity.id = disambiguate_entity(entity.name, new_document) or create_new_node(entity)
    store_graph_facts(extracted["entities"], extracted["relationships"])
```

This connects to July's document AI pipeline pattern — incremental graph construction as new documents arrive, rather than a full rebuild, following the same idempotent, incremental-processing discipline as any other production data pipeline in this roadmap.

## Evaluating Graph-Based Reasoning Quality

Build a golden set specifically of multi-hop questions with known-correct answers (June's evaluation discipline), and measure whether the graph-query tool actually improves accuracy on this specific question class relative to vector-store-only retrieval — validating yesterday's comparison empirically against your own data rather than trusting the general argument alone.

---

*Part of the [Agentic Framework Deep Dives series]({{ site.baseurl }}/tags/deep-dive-series/) — next: [GraphRAG]({{ site.baseurl }}/posts/graphrag-knowledge-graphs-retrieval/), combining this with retrieval directly.*
