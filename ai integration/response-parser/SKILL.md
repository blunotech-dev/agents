---
name: response-parser
description: Parses structured output from LLM responses. Use when extracting JSON from markdown-fenced LLM output, handling partial or streamed JSON, recovering from malformed model responses, or building robust pipelines that consume Claude/OpenAI structured outputs. Trigger for any task involving LLM output parsing, JSON extraction from model responses, stream parsing, or structured output error recovery.
category: "Ai Integration"
---

# response-parser

Covers the non-obvious failure modes and patterns for parsing structured output from LLM responses. Assumes basic JSON/fetch knowledge.

---

## 1. Fence Extraction — What Actually Goes Wrong

Models don't always produce clean ` ```json ... ``` `. Real-world variants:

```
```JSON          ← uppercase
``` json         ← space after backticks
```json\n\n{     ← double newline before content
{                ← no fence at all (common with strong system prompts)
Here's your JSON:\n```json   ← preamble before fence
```

**Extraction regex that handles all of these:**

```js
function extractJSON(raw) {
  // Try fenced first (case-insensitive, optional space, optional language label)
  const fenced = raw.match(/```(?:json)?\s*\n?([\s\S]*?)\n?```/i);
  if (fenced) return fenced[1].trim();

  // Fall back: find first { or [ and match to its closing pair
  const start = raw.search(/[[{]/);
  if (start === -1) return null;
  return extractBalanced(raw, start);
}
```

**Why `extractBalanced` over `lastIndexOf`:** Models sometimes trail with commentary after the closing brace. `lastIndexOf('}')` grabs the wrong position if there's a JSON object inside a string value that closes later.

```js
function extractBalanced(str, start) {
  const open = str[start];
  const close = open === '{' ? '}' : ']';
  let depth = 0;
  for (let i = start; i < str.length; i++) {
    if (str[i] === open) depth++;
    else if (str[i] === close) {
      depth--;
      if (depth === 0) return str.slice(start, i + 1);
    }
  }
  return null; // unbalanced — triggers error recovery
}
```

---

## 2. Stream / Partial Parse

**The problem:** `JSON.parse` is all-or-nothing. On a stream, you have partial text mid-flight.

**Two strategies depending on use case:**

### Strategy A: Buffer-and-parse (simplest, no deps)
Parse only when the stream ends. Safe for non-interactive use.

```js
let buffer = '';
for await (const chunk of stream) {
  buffer += chunk;
}
const json = JSON.parse(extractJSON(buffer));
```

### Strategy B: Incremental parse for live UI updates
Use `partial-json` (npm) — handles incomplete JSON gracefully:

```js
import { parse, Allow } from 'partial-json';

let buffer = '';
for await (const chunk of stream) {
  buffer += chunk;
  const partial = extractJSON(buffer) ?? buffer;
  try {
    // Allow.ALL lets trailing commas and missing closing braces through
    const live = parse(partial, Allow.ALL);
    renderUI(live); // update UI optimistically
  } catch {
    // still too incomplete — skip this tick
  }
}
// Final authoritative parse
const final = JSON.parse(extractJSON(buffer));
```

**Critical non-obvious detail:** Always do a strict `JSON.parse` at end-of-stream even if incremental succeeded. `partial-json` is lenient; it will silently accept things like `{"a": undefined}` that are invalid JSON.

---

## 3. Error Recovery on Malformed Output

LLMs produce specific malformed patterns. Handle them in order of frequency:

### Pattern 1: Trailing commas
```js
// Before parse
const cleaned = str.replace(/,(\s*[}\]])/g, '$1');
```

### Pattern 2: Single quotes instead of double quotes
Only safe if values don't contain apostrophes. Otherwise, retry with reprompt (see below).
```js
const cleaned = str.replace(/'/g, '"');
```

### Pattern 3: Unquoted keys
```js
const cleaned = str.replace(/([{,]\s*)(\w+)(\s*:)/g, '$1"$2"$3');
```

### Pattern 4: Truncated response (hit max_tokens)
Detectable: `finish_reason === 'max_tokens'` or `stop_reason === 'max_tokens'`.
Don't try to parse — it's structurally incomplete. Instead, continue the generation:

```js
if (response.stop_reason === 'max_tokens') {
  // Continue from where it left off
  const continuation = await anthropic.messages.create({
    messages: [
      ...originalMessages,
      { role: 'assistant', content: response.content[0].text },
      { role: 'user', content: 'Continue exactly where you left off.' }
    ]
  });
  combined = response.content[0].text + continuation.content[0].text;
}
```

### Recovery chain (run in order, stop on first success):
```js
async function robustParse(raw, retryFn) {
  const attempts = [
    () => JSON.parse(extractJSON(raw)),
    () => JSON.parse(extractJSON(applyCleaners(raw))),
    () => retryFn?.(), // re-prompt the model
  ];
  for (const attempt of attempts) {
    try { return await attempt(); } catch {}
  }
  throw new Error('parse_failed');
}
```

---

## 4. Schema Validation After Parse

Parsing succeeds doesn't mean the shape is right. Models hallucinate missing fields or wrong types.

**Lightweight validation without a library:**
```js
function validate(obj, schema) {
  // schema: { fieldName: expectedType | 'array' }
  for (const [key, type] of Object.entries(schema)) {
    if (!(key in obj)) throw new Error(`missing field: ${key}`);
    const actual = Array.isArray(obj[key]) ? 'array' : typeof obj[key];
    if (actual !== type) throw new Error(`${key}: expected ${type}, got ${actual}`);
  }
}
```

If validation fails, retry with the error message injected back into the prompt — models self-correct well when told exactly what was wrong.

---

## 5. Non-Obvious Gotchas

**Escape sequences in streamed output:** Models sometimes emit literal `\n` as two characters (`\` + `n`) inside JSON strings, which is valid JSON — but if you're rendering it before parse, it'll look wrong. Don't unescape pre-parse.

**BOM / zero-width spaces:** Copy-paste from some environments prepends `\uFEFF`. Strip it:
```js
const clean = raw.replace(/^\uFEFF/, '');
```

**Numbers as strings:** Models frequently stringify numbers (`"count": "42"`). If your schema expects a number, coerce after validation, not before — otherwise you mask the model's bad behavior instead of correcting the prompt.

**Nested JSON-as-string:** Models sometimes return `{"data": "{\"key\": \"value\"}"}` — valid JSON where the value is a JSON-encoded string. Double-parse intentionally only if your schema expects it; otherwise it's a prompt issue.

---

## 6. Prompting to Reduce Parse Failures (Upstream Fix)

Most parse failures are preventable at the prompt level:

- End system prompt with: `"Respond only with valid JSON. No preamble, no explanation, no markdown fences."`
- If fences are preferred (for readability in logs): `"Wrap your JSON in a single \`\`\`json code block. Nothing before or after."`
- For complex schemas, include a one-shot example in the prompt — reduces structural errors significantly more than schema descriptions alone.
- Avoid asking for JSON and explanation in the same message. Two separate calls is more reliable than "also explain your reasoning in a 'reason' field."