---
name: context-injection
description: Injects retrieved context into prompts correctly — covers context formatting, source attribution, context window budget management, and relevance filtering. Use when building RAG pipelines, document Q&A, search-augmented generation, or any artifact where retrieved chunks must be fed into a Claude prompt reliably.
category: "Ai Integration"
---

# Context Injection

## What This Skill Covers

Non-obvious correctness and efficiency problems when injecting external context into Claude prompts. Assumes you already have retrieved chunks and know how to call the API.

---

## Formatting: Structure the Model Can Parse

Unstructured context blobs degrade answer quality. Wrap each chunk so the model can reason about provenance and boundaries:

```
<context>
<source id="1" title="Stripe Docs: Webhooks" url="https://stripe.com/docs/webhooks">
Webhook events are sent as POST requests. Your endpoint must respond with 2xx within 30 seconds.
</source>
<source id="2" title="Internal Runbook: Payments" url="notion://payments-runbook">
Retry logic is handled by the queue, not the webhook handler.
</source>
</context>
```

**Non-obvious:** plain numbered lists (`1. [doc] text`) work but make attribution harder to enforce. XML tags give the model a reliable anchor to cite from — it will use `id` values in responses when instructed.

Enforce citation in your system prompt:
```
When answering, cite sources using [id] notation. Only use information present in <context>.
```

---

## Placement: System vs User Turn

| Location | When to use |
|---|---|
| System prompt | Static context that applies to every turn (product docs, personas, rules) |
| User turn, before question | Dynamic per-query context (retrieved chunks, search results) |
| After question | Never — models attend less to context placed after the query |

**Non-obvious:** injecting dynamic context into the system prompt works but pollutes it for multi-turn conversations — every subsequent turn carries the stale retrieval. Put per-query context in the user turn, immediately before the question:

```js
messages.push({
  role: "user",
  content: `${formatContext(chunks)}\n\nQuestion: ${userQuery}`
});
```

---

## Budget Management

Context windows are finite. Over-stuffing with low-relevance chunks hurts more than it helps — the model attends less to signal buried in noise.

**Establish a hard token budget before retrieval:**

```js
const TOTAL_BUDGET = 100_000;
const RESPONSE_RESERVE = 2_000;
const SYSTEM_TOKENS = estimateTokens(systemPrompt);
const CONTEXT_BUDGET = TOTAL_BUDGET - SYSTEM_TOKENS - RESPONSE_RESERVE - estimateTokens(userQuery);
```

**Non-obvious:** `estimateTokens` doesn't need to be exact — `text.length / 4` is a good enough approximation for budget allocation. Over-precision here is wasted effort.

Enforce the budget by truncating the chunk list, not individual chunks:

```js
function fitTobudget(chunks, budget) {
  let used = 0;
  return chunks.filter(chunk => {
    const t = estimateTokens(chunk.text);
    if (used + t > budget) return false;
    used += t;
    return true;
  });
}
```

**Non-obvious:** truncating individual chunks mid-sentence is worse than dropping them entirely. A partial chunk often misleads more than a missing one.

---

## Relevance Filtering

Retrieval systems return top-K by vector similarity, but similarity ≠ relevance. Two common failure modes:

1. **Threshold filtering** — drop chunks below a similarity score cutoff before injecting. A chunk with score 0.5 in a top-5 result is often noise:

```js
const MIN_SCORE = 0.7;
const relevant = chunks.filter(c => c.score >= MIN_SCORE);
if (relevant.length === 0) return noContextFallback();
```

2. **Diversity filtering** — near-duplicate chunks waste budget. Deduplicate by content hash or cosine similarity between chunks themselves before injecting.

**Non-obvious:** injecting zero context is often better than injecting low-quality context. Build an explicit `noContextFallback()` path that tells the model to answer from its own knowledge and admit uncertainty — don't silently inject garbage.

---

## Ordering Chunks in the Prompt

Model attention is not uniform — it's stronger at the start and end of context, weaker in the middle (the "lost in the middle" problem).

Ordering strategy:
1. Most relevant chunk → first
2. Second most relevant → last
3. Remaining chunks → middle

```js
function orderForAttention(chunks) {
  if (chunks.length <= 2) return chunks;
  const [first, last, ...middle] = chunks;
  return [first, ...middle, last];
}
```

**Non-obvious:** this only matters when chunks are heterogeneous in relevance. If all chunks are high-relevance, ordering is less important than budget fit.

---

## Multi-turn: Don't Re-inject Stale Context

In a chat loop, context retrieved for turn 1 should not persist into turn 3's prompt. Two patterns:

**Ephemeral injection** (preferred) — context lives only in the turn it was retrieved for:
```js
// Each turn: retrieve fresh, inject, send, discard
const chunks = await retrieve(userMessage);
const augmented = formatContext(chunks) + "\n\n" + userMessage;
messages.push({ role: "user", content: augmented });
```

**Persistent context** — only if the document corpus is fixed and small (e.g., a single uploaded file). Put it in the system prompt once and leave it there.

**Non-obvious:** re-injecting the same chunks every turn inflates cost linearly with conversation length. Cache the formatted context string if you must reuse it.

---

## Source Attribution Output

To get reliable citations, give the model an explicit output contract:

```
System: Always end your response with a ## Sources section listing only the source IDs you actually used. Format: [id]: title
```

Then validate attribution server-side — if the model cites `[id="3"]` but no source with id 3 was injected, it hallucinated a citation. Strip or flag those:

```js
const injectedIds = new Set(chunks.map(c => c.id));
const citedIds = extractCitations(response);  // parse [N] from text
const hallucinated = citedIds.filter(id => !injectedIds.has(id));
if (hallucinated.length > 0) flagForReview(response);
```

---

## Checklist

- [ ] Context wrapped in structured XML with `id` per source
- [ ] Dynamic context injected in user turn, before the query
- [ ] Hard token budget computed before retrieval, not after
- [ ] Chunks dropped whole if over budget, not truncated mid-sentence
- [ ] Similarity threshold applied — no zero-fallback missing
- [ ] Near-duplicate chunks deduplicated before injection
- [ ] High-relevance chunks at start and end (lost-in-middle mitigation)
- [ ] Multi-turn: context not persisted across unrelated turns
- [ ] Citation validation: hallucinated source IDs flagged server-side