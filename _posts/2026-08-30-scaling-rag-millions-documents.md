---
title: "Scaling RAG Infrastructure to Millions of Documents"
date: 2026-08-30 08:00:00 +0530
categories: [AI, Infrastructure]
tags: [infrastructure, ai-infra-series, rag, vector-database, python]
mermaid: true
---

March's RAG series covered retrieval quality at a scale where a single vector index handles everything comfortably. This post covers what changes when a knowledge base grows into the millions of documents — the point where naive infrastructure choices start to break down.

## Where the Naive Approach Breaks

```python
# Fine at 10K documents, problematic at 10M
index = build_flat_index(all_document_embeddings)  # exhaustive search over every vector
results = index.search(query_embedding, top_k=5)   # scans every vector — doesn't scale
```

Exhaustive nearest-neighbor search scans every vector in the index — fine at small scale, but search latency grows linearly with index size, becoming unacceptable well before millions of documents.

## Approximate Nearest Neighbor Indexes

```python
import faiss

# HNSW (Hierarchical Navigable Small World) — the standard for large-scale ANN search
index = faiss.IndexHNSWFlat(embedding_dim, 32)  # 32 = graph connectivity parameter
index.add(all_embeddings)
distances, indices = index.search(query_embedding, k=5)
```

HNSW trades a small amount of recall accuracy for dramatically faster search at scale — this is what the vector database comparison post from March referenced without full depth; at millions of documents, an ANN index isn't an optimization, it's a requirement for acceptable latency.

## Sharding the Index

```python
def shard_by_hash(doc_id: str, num_shards: int) -> int:
    return int(hashlib.sha256(doc_id.encode()).hexdigest(), 16) % num_shards

def sharded_search(query_embedding, shards: list, top_k: int = 5) -> list:
    per_shard_results = [shard.search(query_embedding, top_k) for shard in shards]  # parallel across shards
    return merge_and_rerank(per_shard_results, top_k)
```

Beyond what a single machine's memory can hold, sharding the index across multiple nodes — querying all shards in parallel and merging results — is the standard scaling pattern, directly analogous to the horizontal scaling covered for inference serving earlier this month, now applied to the retrieval layer.

## Incremental Indexing at Scale

```python
async def incremental_index_pipeline(document_stream):
    async for batch in batched(document_stream, size=1000):
        embeddings = await embed_batch_async(batch)
        await vector_store.upsert_batch(embeddings)
        await update_bm25_index(batch)  # keyword index kept in sync alongside vector index
```

Full reindexing becomes prohibitively expensive at millions of documents — production systems at this scale need incremental indexing pipelines that add and update documents continuously, with the idempotent, checkpointed processing discipline from July's document pipeline post applied at much larger scale.

## Hybrid Search Becomes More Important, Not Less

```python
def hybrid_search_at_scale(query: str, top_k: int = 5) -> list:
    vector_results = ann_index.search(embed(query), top_k=50)  # wider candidate net
    keyword_results = bm25_index.search(query, top_k=50)
    return rerank_and_merge(vector_results, keyword_results, top_k)
```

At large scale, pure vector search's recall limitations (from ANN's approximate nature) compound with the general semantic-vs-exact-match gap — combining with keyword search (BM25) and reranking becomes more valuable, not less, as scale increases and exhaustive verification of every result becomes infeasible.

## Cost at Scale: Embedding and Storage

```python
def estimate_scaling_cost(num_documents: int, avg_chunks_per_doc: int, embedding_dim: int) -> dict:
    total_vectors = num_documents * avg_chunks_per_doc
    storage_gb = total_vectors * embedding_dim * 4 / 1e9  # 4 bytes per float32 dimension
    embedding_cost = total_vectors * EMBEDDING_COST_PER_VECTOR
    return {"storage_gb": storage_gb, "one_time_embedding_cost": embedding_cost}
```

At millions of documents, both the one-time embedding cost and ongoing storage cost become material line items worth explicit budgeting — connecting directly to this month's cost attribution posts, now applied to the retrieval infrastructure layer rather than just inference.

## Monitoring Retrieval Infrastructure at Scale

Extend June's RAG evaluation (recall@k, faithfulness) with infrastructure-specific metrics — index build/update latency, shard balance, and ANN recall degradation over time as the index grows — since retrieval quality at scale depends on infrastructure health in ways a small-scale RAG deployment never has to consider.

---

*Part of the [AI Infrastructure series]({{ site.baseurl }}/tags/ai-infra-series/) — closing tomorrow with [a reference architecture for production AI infrastructure]({{ site.baseurl }}/posts/reference-architecture-production-ai-infrastructure/) tying the whole month together.*
