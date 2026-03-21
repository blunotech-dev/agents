---
name: document-chunking
description: Implements chunking strategies for documents before embedding or retrieval — covers fixed-size, semantic, recursive, and sliding window approaches with overlap and metadata preservation. Use when building RAG pipelines, document ingestion, vector store population, or any artifact where documents must be split before embedding.
category: "Ai Integration"
---

# Document Chunking

## What This Skill Covers

Non-obvious failure modes and decisions in chunking. Assumes you know what chunking is and why you need it.

---

## Strategy Selection

Choose once per corpus, not per document:

| Strategy | Use when |
|---|---|
| Fixed-size | Homogeneous content (logs, transcripts, tabular notes) |
| Recursive | General prose — best default |
| Semantic | High-variance documents where topic boundaries matter (research papers, long-form articles) |
| Sliding window | Query targets local context across boundaries (dense technical docs, code) |

**Non-obvious:** mixing strategies in the same vector index breaks retrieval consistency. Score distributions become incomparable across chunk types. Pick one per index.

---

## Fixed-Size Chunking

Split by character or token count at hard boundaries.

```js
function fixedChunk(text, size = 512, overlap = 50) {
  const chunks = [];
  let start = 0;
  while (start < text.length) {
    chunks.push(text.slice(start, start + size));
    start += size - overlap;
  }
  return chunks;
}
```

**Non-obvious:** chunk on tokens, not characters, when your embedding model has a token limit. A 512-char chunk is ~128 tokens in English but can be 400+ tokens in code or CJK text. Exceeding the model's token limit silently truncates the embedding.

Cheap token estimate: `Math.ceil(text.length / 4)` for English prose. For code, use `text.length / 3`.

---

## Recursive Chunking (Default Choice)

Split by a priority hierarchy of separators, falling back to smaller ones until chunks fit the target size:

```js
const SEPARATORS = ["\n\n", "\n", ". ", " ", ""];

function recursiveChunk(text, maxSize = 512, overlap = 50, separators = SEPARATORS) {
  if (text.length <= maxSize) return [text];

  const sep = separators.find(s => text.includes(s)) ?? "";
  const splits = sep ? text.split(sep) : [...text];

  const chunks = [];
  let current = "";

  for (const split of splits) {
    const candidate = current ? current + sep + split : split;
    if (candidate.length <= maxSize) {
      current = candidate;
    } else {
      if (current) chunks.push(current);
      // split is itself too large — recurse
      if (split.length > maxSize) {
        chunks.push(...recursiveChunk(split, maxSize, overlap, separators.slice(1)));
        current = "";
      } else {
        current = split;
      }
    }
  }
  if (current) chunks.push(current);

  return applyOverlap(chunks, overlap, sep);
}
```

**Non-obvious:** the overlap must be applied after splitting, not during — re-joining adjacent chunks with overlap is simpler and less error-prone than trying to back up the cursor mid-split.

---

## Semantic Chunking

Group sentences until a cosine similarity drop signals a topic boundary. Requires embedding each sentence — expensive but produces the most coherent chunks.

```js
async function semanticChunk(sentences, embedFn, threshold = 0.85) {
  const embeddings = await Promise.all(sentences.map(embedFn));
  const chunks = [];
  let group = [sentences[0]];

  for (let i = 1; i < sentences.length; i++) {
    const sim = cosineSimilarity(embeddings[i - 1], embeddings[i]);
    if (sim >= threshold) {
      group.push(sentences[i]);
    } else {
      chunks.push(group.join(" "));
      group = [sentences[i]];
    }
  }
  if (group.length) chunks.push(group.join(" "));
  return chunks;
}
```

**Non-obvious pitfalls:**
- Threshold is corpus-dependent. 0.85 is a starting point; calibrate by inspecting boundary sentences on a sample. Too high = over-split. Too low = giant chunks.
- Pre-split into sentences with a real sentence splitter, not `text.split(". ")`. Abbreviations ("Dr.", "e.g.") break naive splits.
- This embeds N sentences per document at ingest time. Budget accordingly — it costs the same as embedding N chunks but produces fewer of them.

---

## Sliding Window Chunking

Every chunk overlaps with its neighbors by a fixed stride. Use when a query might land in the middle of a concept that spans a boundary.

```js
function slidingChunk(text, size = 512, stride = 256) {
  // stride < size creates overlap; stride == size is non-overlapping fixed
  const chunks = [];
  let start = 0;
  while (start < text.length) {
    chunks.push({ text: text.slice(start, start + size), start, end: Math.min(start + size, text.length) });
    start += stride;
  }
  return chunks;
}
```

**Non-obvious:** sliding window inflates your index size by `size / stride` factor. A 50% overlap (stride = size/2) doubles index size and retrieval cost. Only use when your query patterns genuinely need boundary-spanning context — otherwise recursive chunking with overlap achieves similar recall at lower cost.

---

## Overlap: What It Does and Doesn't Fix

Overlap ensures that sentences split across chunk boundaries appear in both adjacent chunks, improving recall for queries that land near boundaries.

What overlap does **not** fix:
- Semantic coherence — a chunk can still start mid-argument
- Attribution — overlapping text will appear in two retrieved chunks, potentially returning near-duplicate results

**Non-obvious:** deduplicate retrieved chunks before injection using position metadata (`start`, `end`). If two retrieved chunks overlap by >50% of the shorter one's length, drop the lower-scoring one.

```js
function deduplicateByPosition(chunks) {
  return chunks.filter((c, i) =>
    !chunks.slice(0, i).some(prev =>
      overlapRatio(prev.start, prev.end, c.start, c.end) > 0.5
    )
  );
}
```

---

## Metadata Preservation

Every chunk must carry enough metadata to reconstruct its origin and position. Minimum required fields:

```js
{
  text: "...",
  source: "s3://bucket/file.pdf",   // original document URI
  page: 4,                           // page or section number if available
  start: 1024,                       // character offset in source
  end: 1536,
  chunkIndex: 7,                     // position within document
  totalChunks: 22,
  heading: "Section 3.2"            // nearest heading above chunk, if parseable
}
```

**Non-obvious:** `heading` is high-value for retrieval quality — prepending the nearest section heading to the chunk text before embedding dramatically improves semantic search for structured documents (docs, reports, wikis). It's cheap to extract if you parse the document before chunking.

Prepend pattern:
```js
const enriched = chunk.heading
  ? `${chunk.heading}\n\n${chunk.text}`
  : chunk.text;
// embed `enriched`, store original `chunk.text` for display
```

Embed the enriched version, return the original in results. Never show the prepended heading twice in the UI.

---

## Document-Type Gotchas

**Markdown / HTML** — strip formatting before chunking or split on heading tags. Raw `##` headers and `<div>` noise pollutes embeddings.

**Code** — split on function/class boundaries, not character count. A function split mid-body is nearly useless as a retrieved chunk. Use AST-aware splitting or at minimum split on `\n\n` with a large max size (1024+ tokens).

**PDFs** — page boundaries are not semantic boundaries. Merge across pages before chunking; split by estimated reading flow, not page number.

**Tables** — never chunk through a table row. Either keep the whole table as one chunk (if small) or repeat the header row in every chunk that contains table rows.

---

## Checklist

- [ ] Strategy chosen once per index, not per document
- [ ] Chunk size in tokens, not characters, when embedding model has token limit
- [ ] Recursive: overlap applied after splitting, not during
- [ ] Semantic: threshold calibrated on a sample, not assumed
- [ ] Sliding window: index size inflation accounted for
- [ ] Overlap: near-duplicate retrieved chunks deduplicated by position at query time
- [ ] Every chunk carries `source`, `start`, `end`, `chunkIndex` metadata
- [ ] Heading prepended to chunk text before embedding (not stored, not displayed)
- [ ] Code split on function/class boundaries, not character count
- [ ] Tables kept whole or header row repeated across row-chunks