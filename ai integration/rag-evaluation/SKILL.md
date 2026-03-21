---
name: rag-evaluation
description: Evaluates RAG pipeline quality end-to-end. Use when measuring retrieval precision/recall, answer faithfulness, hallucination rate, or comparing RAG configurations. Trigger for any task involving RAG quality assessment, LLM-as-judge evaluation, RAGAS metrics, groundedness checks, or A/B comparison of retrieval or generation changes.
category: "Ai Integration"
---

# rag-evaluation

Covers the non-obvious measurement traps, metric choices, and eval infrastructure decisions for RAG pipelines. Assumes you have a working RAG pipeline and want to measure it.

---

## 1. The Three Failure Modes and Which Metrics Catch Them

RAG fails in three distinct ways. Each requires different metrics — conflating them produces misleading aggregate scores.

| Failure mode | Example | Metric |
|---|---|---|
| Retrieval misses relevant doc | Right answer exists, never retrieved | Recall@k |
| Retrieval returns irrelevant docs | Retrieved docs don't support the answer | Precision@k, Context Relevance |
| Generation hallucinates | Docs retrieved correctly, answer fabricated | Faithfulness, Answer Grounding |

**Non-obvious:** A high faithfulness score with low retrieval recall is a trap — the model is faithfully generating from the wrong context. Always report retrieval and generation metrics together; never report faithfulness alone.

---

## 2. Building an Eval Set (The Hard Part)

### Minimum viable eval set
20-50 questions with labeled relevant chunk IDs. Fewer than 20 produces high-variance metrics; above 50 adds marginal signal for most pipelines.

### Question types to include deliberately
Don't sample randomly from user logs — logs oversample easy queries. Explicitly include:
- **Multi-hop**: answer requires combining two chunks (`"What did X say about Y in the context of Z?"`)
- **Negative**: answer is genuinely not in the corpus (`"Does the doc cover topic W?"` — correct answer: no)
- **Exact-match**: queries with specific IDs, names, version numbers
- **Abstractive**: requires synthesis, not extraction

Negative questions are the most commonly omitted. Without them, you can't measure hallucination on unanswerable queries — a critical failure mode in production.

### Synthetic question generation (when you lack real queries)
Use the LLM to generate questions from each chunk, then filter:

```python
async def generate_questions(chunk_text, client, n=3):
    resp = await client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=400,
        messages=[{"role": "user", "content": f"""Generate {n} questions that are answered by this passage.
Include at least one that requires the specific details in this passage (not general knowledge).
Respond with a JSON array of strings only.

Passage:
{chunk_text}"""}]
    )
    import json, re
    raw = resp.content[0].text
    cleaned = re.sub(r'```json|```', '', raw).strip()
    return json.loads(cleaned)
```

**Non-obvious filtering step:** After generating, run your RAG pipeline on each synthetic question and discard any where the source chunk doesn't appear in top-5 results. These are unanswerable by your current retrieval — they'll only measure retrieval failure, not generation quality, which contaminates your faithfulness scores.

---

## 3. Retrieval Metrics

### Recall@k and Precision@k

```python
def retrieval_metrics(retrieved_ids: list[str], relevant_ids: set[str], k: int):
    retrieved_k = retrieved_ids[:k]
    hits = sum(1 for id in retrieved_k if id in relevant_ids)
    return {
        "recall@k": hits / len(relevant_ids) if relevant_ids else 0,
        "precision@k": hits / k,
        "mrr": next(
            (1 / (i + 1) for i, id in enumerate(retrieved_k) if id in relevant_ids),
            0
        )
    }
```

**MRR vs. Recall@k:** MRR rewards finding the relevant doc early (rank 1 > rank 5). Use MRR when users read only the top result. Use Recall@k when the LLM receives all k chunks — position doesn't matter if everything gets passed to context.

### Context Relevance (LLM-judged)
Measures whether retrieved chunks are actually useful, not just topically related.

```python
CONTEXT_RELEVANCE_PROMPT = """Given this question and retrieved passage, rate relevance on a scale of 1-3.
1 = Not relevant (passage does not help answer the question)
2 = Partially relevant (passage contains related info but not the answer)
3 = Directly relevant (passage contains information needed to answer)

Question: {question}
Passage: {passage}

Respond with JSON: {{"score": <1|2|3>, "reason": "<one sentence>"}}"""
```

**Non-obvious:** Average context relevance across all k retrieved chunks, not just the top-1. A retriever that returns 1 great chunk and 4 garbage chunks has the same top-1 score as one that returns 5 useful chunks — but they produce very different generation quality.

---

## 4. Generation Metrics

### Faithfulness (hallucination detection)

Faithfulness measures whether every claim in the answer is grounded in the retrieved context — not whether the answer is correct.

```python
FAITHFULNESS_PROMPT = """You are evaluating whether an AI answer is grounded in provided sources.

Sources:
{context}

Answer to evaluate:
{answer}

For each factual claim in the answer, determine if it is:
- SUPPORTED: explicitly stated or directly inferable from sources
- UNSUPPORTED: not found in sources or contradicts sources

Respond with JSON:
{{
  "claims": [
    {{"claim": "<text>", "verdict": "SUPPORTED"|"UNSUPPORTED", "source_snippet": "<quote or null>"}}
  ],
  "faithfulness_score": <0.0-1.0>  // fraction of claims that are SUPPORTED
}}"""
```

**Critical distinction — faithfulness vs. correctness:**
- Faithful but wrong: context contains an error, model repeats it faithfully
- Correct but unfaithful: model uses parametric knowledge beyond the context

For production RAG, faithfulness is the right metric. Correctness requires a ground-truth answer, which is expensive to label at scale.

### Answer Relevance
Measures whether the answer actually addresses the question, not whether it's grounded.

```python
ANSWER_RELEVANCE_PROMPT = """Does this answer address the question asked?

Question: {question}
Answer: {answer}

Score 1-3:
1 = Doesn't answer the question (off-topic or evasive)
2 = Partially answers (addresses some aspect but incomplete or indirect)
3 = Directly answers the question

Respond with JSON: {{"score": <1|2|3>, "reason": "<one sentence>"}}"""
```

**Non-obvious:** Low answer relevance with high faithfulness means the retriever returned irrelevant chunks and the model faithfully summarized them instead of saying "I don't know." This is a retrieval problem, not a generation problem — but it shows up as a generation metric failure.

---

## 5. Hallucination Detection on Unanswerable Queries

This is the most commonly skipped eval and the most damaging in production.

```python
UNANSWERABLE_PROMPT = """The following context was retrieved for this question.
Determine if the context contains sufficient information to answer the question.

Question: {question}
Context: {context}

Then evaluate the answer:
Answer: {answer}

Respond with JSON:
{{
  "answerable_from_context": true|false,
  "answer_admits_uncertainty": true|false,  // did the answer say "I don't know" or similar
  "hallucinated": true|false  // answerable=false AND answer_admits_uncertainty=false
}}"""
```

**Track hallucination rate separately:**
```python
hallucination_rate = sum(r['hallucinated'] for r in results) / len(unanswerable_queries)
```

A pipeline with 90% faithfulness on answerable queries and 60% hallucination rate on unanswerable ones is dangerous in production — the aggregate faithfulness score hides it.

---

## 6. LLM-as-Judge: Reducing Bias

**Positional bias:** Judges rate the first option higher in A/B comparisons. Always run both orderings and average:
```python
async def compare(q, answer_a, answer_b, context, client):
    score_ab = await judge(q, answer_a, answer_b, context, client)  # A first
    score_ba = await judge(q, answer_b, answer_a, context, client)  # B first
    # score_ab["preferred_first"] and score_ba["preferred_first"] should cancel out
    a_wins = (score_ab["winner"] == "A") + (score_ba["winner"] == "B")
    return "A" if a_wins >= 1 else "B"
```

**Verbosity bias:** Judges prefer longer answers. If comparing retrieval strategies (same generator), this cancels out. If comparing prompts or models, control for length explicitly in the judge prompt: `"Ignore answer length. Focus only on factual accuracy and relevance."`

**Self-evaluation bias:** Using the same model as judge that generated the answers inflates scores. Use a different model or a stronger one (e.g., generate with Haiku, judge with Sonnet).

**Calibrate your judge:** Before using LLM-as-judge at scale, test it on 10-20 hand-labeled examples. Calculate judge-human agreement (Cohen's kappa). Below 0.6 kappa, your judge is unreliable — refine the prompt before running evals.

---

## 7. Baseline Comparison

Never report RAG metrics in isolation. Always compare against:

1. **No-RAG baseline**: same LLM, same question, no context — measures what retrieval actually adds
2. **Oracle baseline**: same LLM with the ground-truth chunk injected — upper bound on what better retrieval could achieve
3. **Previous config**: your last pipeline version — validates that changes are improvements

```python
async def run_baselines(questions, pipeline, oracle_chunks, client):
    results = {}
    for q in questions:
        rag_answer = await pipeline.answer(q)
        no_rag_answer = await generate(q, context="", client=client)
        oracle_answer = await generate(q, context=oracle_chunks[q['id']], client=client)
        results[q['id']] = {
            "rag": rag_answer,
            "no_rag": no_rag_answer,
            "oracle": oracle_answer
        }
    return results
```

**Oracle gap analysis:** `oracle_faithfulness - rag_faithfulness` tells you how much headroom retrieval improvement would give you. If the gap is small (<0.05), improving the generator will help more than improving retrieval. If the gap is large (>0.15), fix retrieval first.

---

## 8. Eval Infrastructure

**Run evals async and in parallel** — LLM judge calls are the bottleneck:

```python
import asyncio

async def eval_all(samples, judge_fn, concurrency=10):
    sem = asyncio.Semaphore(concurrency)
    async def bounded(s):
        async with sem:
            return await judge_fn(s)
    return await asyncio.gather(*[bounded(s) for s in samples])
```

**Persist raw judge outputs, not just scores.** When a metric drops between runs, you need the per-claim verdicts and reasons to diagnose why. Scores alone tell you something broke; raw outputs tell you what.

**Score instability:** LLM judges have non-zero variance. Run each sample twice on high-stakes evals and flag disagreements (where two runs give different verdicts) for human review rather than averaging blindly.

**Cost estimation before running:**
```python
# Rough estimate: faithfulness eval ~ 800 tokens in + 300 out per sample
# At claude-haiku pricing, 50 samples ≈ $0.02
# At claude-sonnet pricing, 50 samples ≈ $0.20
# Run haiku for iteration; run sonnet for final eval before shipping
```

---

## 9. Metric Summary and Reporting

Report these together — never a single number:

| Metric | What it measures | Acceptable threshold (typical) |
|---|---|---|
| Recall@5 | Retrieval completeness | > 0.75 |
| Precision@5 | Retrieval relevance | > 0.60 |
| MRR | Retrieval ranking quality | > 0.65 |
| Context Relevance | Retrieved chunk usefulness | > 2.3 / 3.0 |
| Faithfulness | Answer groundedness | > 0.85 |
| Answer Relevance | Answer addresses question | > 2.5 / 3.0 |
| Hallucination Rate (unanswerable) | Confabulation on unknowns | < 0.15 |

Thresholds are starting points; calibrate against your oracle baseline and acceptable user experience for your domain.