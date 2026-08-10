---
title: "Fine-Tuning Embedding Models for Domain-Specific Retrieval"
date: 2026-05-24 08:00:00 +0530
categories: [AI, Fine-Tuning]
tags: [fine-tuning, fine-tuning-series, embeddings, rag, python]
mermaid: true
---

Everything so far this month fine-tuned a generative model. Embedding models — the retrieval backbone of every RAG system from March — can be fine-tuned too, and it's often a higher-leverage investment than fine-tuning the generator, since retrieval quality caps everything downstream of it.

```mermaid
flowchart LR
    A[Query-document pairs] --> D[Contrastive training]
    B[Hard negatives] --> D
    C[Synthetic queries] --> D
    D --> E[Domain-tuned embedding model]
    E --> F[recall@k / MRR against held-out queries]
```

Contrastive training pulls genuinely similar pairs together and pushes hard negatives apart, and the payoff is measured directly in retrieval metrics inside the actual pipeline — not embedding quality in isolation.

## Why General-Purpose Embeddings Fall Short

A general-purpose embedding model (E5, OpenAI's `text-embedding-3`, from the earlier RAG series comparison) is trained on broad web text similarity. It doesn't know that in your domain, "the widget failed to initialize" and "startup error on the widget" should embed close together, while in a different domain "failed to initialize" and "declined to start" might mean something importantly different.

## The Training Objective: Contrastive Learning

Embedding fine-tuning uses contrastive loss — pull genuinely similar pairs closer together in embedding space, push dissimilar pairs apart:

```python
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader

model = SentenceTransformer("intfloat/e5-base-v2")

train_examples = [
    InputExample(texts=["widget failed to initialize", "startup error on the widget"], label=1.0),
    InputExample(texts=["widget failed to initialize", "billing invoice discrepancy"], label=0.0),
]
train_dataloader = DataLoader(train_examples, shuffle=True, batch_size=16)
train_loss = losses.CosineSimilarityLoss(model)

model.fit(train_objectives=[(train_dataloader, train_loss)], epochs=3, warmup_steps=100)
```

## Where Training Pairs Come From

- **Query-document pairs from real search logs** — a user's query paired with the document they actually clicked or that resolved their issue is strong positive-pair signal
- **Hard negatives, not random negatives** — pairing a query with a *plausible but wrong* document teaches the model much more than pairing it with an obviously unrelated one; mine these from documents that rank highly but weren't the right answer
- **Synthetic query generation** — use an LLM to generate realistic queries a document would answer, the same synthetic-data pattern from earlier this month applied to retrieval pairs specifically

```python
def generate_hard_negatives(query: str, correct_doc: str, candidate_pool: list[str], k: int = 3) -> list[str]:
    scored = [(doc, embed_similarity(query, doc)) for doc in candidate_pool if doc != correct_doc]
    scored.sort(key=lambda x: -x[1])
    return [doc for doc, _ in scored[:k]]  # highest-scoring wrong documents = hardest negatives
```

## Evaluating Retrieval Quality Directly

Don't evaluate the fine-tuned embedding model in isolation — evaluate it inside the actual retrieval pipeline, using recall@k and MRR against a held-out set of real queries with known correct documents:

```python
def evaluate_retrieval(model, eval_queries: list[dict], k: int = 5) -> dict:
    hits_at_k = 0
    for item in eval_queries:
        results = search_with_model(model, item["query"], top_k=k)
        if item["correct_doc_id"] in [r.id for r in results]:
            hits_at_k += 1
    return {"recall_at_k": hits_at_k / len(eval_queries)}
```

## Why This Is Often Higher Leverage Than Fine-Tuning the Generator

A RAG system's answer quality is capped by retrieval quality — no amount of generator fine-tuning fixes a wrong or missing document in the retrieved context. If your RAG evaluations (RAGAS faithfulness and relevancy scores from March) point to retrieval, not generation, as the weak link, embedding fine-tuning directly addresses the actual bottleneck.

---

*Part of the [Fine-Tuning series]({{ site.baseurl }}/tags/fine-tuning-series/) — next: [hyperparameter tuning]({{ site.baseurl }}/posts/hyperparameter-tuning-fine-tuning-jobs/) for LLM fine-tuning runs generally.*
