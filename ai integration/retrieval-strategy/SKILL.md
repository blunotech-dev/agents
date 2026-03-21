---
name: retrieval-strategy
description: Implements optimized retrieval strategies for RAG and search pipelines. Use when building or improving document retrieval, choosing between similarity search, hybrid search, reranking, or MMR, diagnosing retrieval quality issues, or tuning recall/precision tradeoffs in LLM-powered search. Trigger for any task involving vector search, embedding retrieval, semantic search, BM25, reranking, or RAG pipeline optimization.
category: "Ai Integration"
---

# retrieval-strategy

Covers the non-obvious decisions, failure modes, and tuning levers across retrieval approaches. Assumes familiarity with embeddings and basic vector DB APIs.

---

## 1. Choosing a Strategy

| Signal | Use |
|---|---|
| Queries are natural language, corpus is prose | Similarity search alone (baseline) |
| Queries contain exact terms, model names, IDs, codes | Hybrid (semantic + keyword) |
| Top-k recall is good but ranking is wrong | Add reranker |
| Results are redundant / clustered around one subtopic | MMR |
| Long documents, multi-hop questions | Hierarchical retrieval |

Don't default to hybrid + rerank everywhere. Each layer adds latency and a new failure surface. Start minimal, instrument, then add layers where metrics show it helps.

---

## 2. Similarity Search — Non-Obvious Failure Modes

**The curse of high dimensionality in cosine similarity:** At 1536+ dims, cosine distances cluster tightly — top-100 and top-1000 results can have near-identical scores. This makes score thresholding unreliable. Use rank cutoff (`top_k`) not score threshold unless you normalize per-query.

**Query-document embedding asymmetry:** Most embedding models are trained with short queries paired against longer passages. Embedding a full paragraph as the query degrades recall. If your "query" is a document chunk (e.g., find similar chunks), use a bi-encoder trained for symmetric similarity (e.g., `all-mpnet-base-v2`) not an asymmetric retrieval model (e.g., `text-embedding-3-large`).

**Chunking strategy matters more than embedding model choice:**
- Overlapping chunks (50% overlap) outperform non-overlapping for boundary-spanning content
- Chunk at semantic boundaries (paragraphs, sections), not fixed token counts
- Parent-child chunking: embed small chunks for precision, return parent chunks for context. Non-obvious: store parent_id on each child chunk, then after retrieval do a single bulk fetch of parent chunks — don't embed parents.

```js
// Parent-child retrieval pattern
const childResults = await vectorDB.query(queryEmbedding, { topK: 20 });
const parentIds = [...new Set(childResults.map(r => r.metadata.parent_id))];
const fullChunks = await docStore.getBatch(parentIds); // return these to LLM
```

---

## 3. Hybrid Search (Semantic + Keyword)

**Why BM25 catches what semantic misses:** Semantic search generalizes — "myocardial infarction" matches "heart attack". That's usually good. But for exact product names, error codes, version numbers, and proper nouns, semantic similarity dilutes specificity. BM25 scores exact token matches.

**Reciprocal Rank Fusion (RRF) over score normalization:**
Score-level fusion requires normalizing across two different distributions (cosine similarity vs. BM25 TF-IDF). RRF avoids this entirely — it only uses rank positions:

```python
def rrf(rankings: list[list[str]], k=60) -> list[str]:
    scores = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)

# k=60 is empirically robust; lower k amplifies top ranks more aggressively
```

**Non-obvious: BM25 requires the same tokenizer as your index.** If your vector DB tokenizes on ingest, make sure your query-time BM25 uses identical preprocessing (lowercasing, stemming, stopwords). Mismatch is a silent failure — results look plausible but recall drops.

**When to weight semantic vs. keyword:** Don't tune weights per-query type statically. Instead, detect query type:
- Query contains quoted strings, version numbers, or uppercase acronyms → boost keyword weight
- Query is a full sentence or question → boost semantic weight

---

## 4. Reranking

**Rerankers are cross-encoders; retrievers are bi-encoders.** The retriever compares pre-computed embeddings (fast, approximate). The reranker sees (query, document) as a pair and scores relevance jointly (slow, accurate). Never rerank more than you need to — rerank top-20 to 40, not top-200.

**Cohere Rerank vs. local cross-encoder:**
- Cohere: lower latency for large batches, handles multilingual, costs per call
- `cross-encoder/ms-marco-MiniLM-L-6-v2` (HuggingFace): free, ~15ms/pair on CPU, good for English

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

def rerank(query, candidates, top_n=5):
    pairs = [(query, c['text']) for c in candidates]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
    return [doc for doc, _ in ranked[:top_n]]
```

**Non-obvious: reranker scores are not calibrated probabilities.** A score of 0.9 vs. 0.3 doesn't mean 3x more relevant — it's an ordinal rank signal. Don't threshold on reranker scores; always take top-n by rank.

**Reranker failure mode — length bias:** Cross-encoders trained on MS MARCO tend to prefer longer documents (more token overlap opportunity). If your corpus has mixed-length chunks, normalize chunk length before reranking or accept this bias consciously.

---

## 5. MMR (Maximal Marginal Relevance)

**When to use:** Results are relevant but redundant — you retrieve 5 chunks that all say the same thing from the same section. MMR trades some relevance for diversity.

**The formula:**
```
MMR(d) = λ · sim(query, d) − (1−λ) · max(sim(d, already_selected))
```

**Implementation:**
```python
import numpy as np

def mmr(query_emb, candidate_embs, candidate_docs, top_k=5, lambda_=0.5):
    selected, remaining = [], list(range(len(candidate_docs)))
    
    while len(selected) < top_k and remaining:
        if not selected:
            # First pick: pure relevance
            best = max(remaining, key=lambda i: np.dot(query_emb, candidate_embs[i]))
        else:
            sel_embs = candidate_embs[selected]
            best = max(
                remaining,
                key=lambda i: (
                    lambda_ * np.dot(query_emb, candidate_embs[i])
                    - (1 - lambda_) * np.max(sel_embs @ candidate_embs[i])
                )
            )
        selected.append(best)
        remaining.remove(best)
    
    return [candidate_docs[i] for i in selected]
```

**λ tuning:**
- `λ = 1.0` → pure similarity (no diversity)
- `λ = 0.5` → balanced (good default)
- `λ = 0.3` → aggressive diversity (useful for broad survey questions)

**Non-obvious: MMR requires embeddings at rerank time,** not just scores. If your vector DB only returns scores (not vectors), you'll need a second embed call for the candidates. Cache candidate embeddings during retrieval instead of re-embedding.

---

## 6. Hierarchical / Multi-Stage Retrieval

For long documents or multi-hop questions where a single chunk won't contain the answer:

**Stage 1 — document-level retrieval:** Embed document summaries or titles, retrieve relevant documents.
**Stage 2 — passage-level retrieval:** Within retrieved documents only, run similarity search on chunks.

This avoids noisy cross-document chunk comparison and dramatically reduces search space for stage 2.

**Contextual compression (non-obvious pattern):** After retrieval, use an LLM to extract only the sentence(s) within each chunk that are relevant to the query before passing to the final LLM. Reduces context bloat without losing the retrieved chunks.

```python
async def compress(chunk_text, query, client):
    resp = await client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": f"Extract only the sentences relevant to: '{query}'\n\nPassage:\n{chunk_text}\n\nIf nothing is relevant, respond with 'NONE'."
        }]
    )
    result = resp.content[0].text
    return None if result.strip() == "NONE" else result
```

---

## 7. Diagnosing Retrieval Quality

**Instrument before tuning.** Log for every query:
- `retrieved_ids`: what was returned
- `rank_of_relevant`: where the known-good doc appeared (requires labeled eval set)
- `mrr`: mean reciprocal rank across eval set
- `recall@k`: fraction of relevant docs in top-k

**Fast eval set construction:** Take 20-30 real queries, manually label top-3 relevant chunks. Run retrieval, compute MRR. This catches 80% of real issues.

**Common diagnosis patterns:**

| Symptom | Likely cause | Fix |
|---|---|---|
| Relevant doc is rank 8-15, not top-5 | Retriever recall fine, ranking wrong | Add reranker |
| Relevant doc not in top-100 | Embedding mismatch or chunking issue | Check chunk size, try different embedding model |
| All top-5 results from same document | No diversity | Add MMR or deduplicate by doc_id |
| Exact-match queries miss obvious results | No keyword component | Add BM25 hybrid |
| Good results but LLM says "not found" | Chunk too small, context truncated | Switch to parent-child chunking |

---

## 8. Latency vs. Quality Tradeoffs

| Optimization | Latency impact | Quality impact |
|---|---|---|
| Reduce top-k from 100 → 20 | −60% vector search | Recall drops if reranker relies on it |
| Async parallel retrieval (semantic + BM25 simultaneously) | Cuts hybrid latency by ~40% | None |
| Cache embeddings for repeated queries | Near-zero for cache hits | None |
| Rerank only if retriever score spread is low | Skips reranker ~30% of calls | Minimal — low spread means ranking is uncertain anyway |
| Use smaller embedding model for ANN index | Faster ingest + search | Slight recall drop; test empirically |

**Async hybrid retrieval pattern:**
```js
const [semanticResults, keywordResults] = await Promise.all([
  vectorDB.query(queryEmbedding, { topK: 40 }),
  bm25Index.search(queryText, { topK: 40 })
]);
const fused = rrf([semanticResults.map(r => r.id), keywordResults.map(r => r.id)]);
```