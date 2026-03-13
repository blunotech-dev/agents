---
name: code-complexity-explainer
description: Explain complex or legacy code in plain English, including what it does, why it likely exists, and how it can be safely modified or refactored. Use when users paste confusing, messy, undocumented, or inherited code and ask what it does, how to change it, what might break, or request comments/documentation. Outputs a structured, human-readable breakdown rather than just inline comments
category: "Documentation"
---

# Code Complexity Explainer

Produces a clear, structured plain-English breakdown of complex or legacy code — what it does, why it exists, and how to safely modify it.

---

## Output Structure

Always produce the following five sections in order. Tailor the depth to the complexity of the code: a 10-line function needs less than a 300-line module.

### 1. 🧭 One-Line Summary
One sentence. What does this code accomplish at the highest level?

> Example: *"This function normalizes a user's billing address and deduplicates it against previously stored entries before writing to the database."*

---

### 2. 📖 Plain-English Walkthrough
Step-by-step prose explanation of what the code does, in the order it executes. Avoid jargon where possible; define it when unavoidable. Use numbered steps or short paragraphs.

Guidelines:
- Follow the actual control flow (loops, conditions, early returns)
- Call out non-obvious logic explicitly (bitwise tricks, magic numbers, regex patterns, etc.)
- Translate cryptic variable names into what they actually represent
- Note any side effects (writes to disk, network calls, mutations, global state)

---

### 3. 🏛️ Why This Probably Exists
Infer the historical/business reason this code was written the way it was. Acknowledge uncertainty where appropriate.

Look for signals like:
- Legacy compatibility shims (old API contracts, deprecated libraries)
- Performance workarounds (manual caching, avoided abstractions)
- Defensive coding against specific past bugs ("this handles the edge case where X")
- Framework-specific idioms that may be unfamiliar
- Organizational patterns (this might be a copy-paste of a common internal pattern)

If the intent is genuinely unclear, say so and offer two or three plausible hypotheses.

---

### 4. ⚠️ What to Watch Out For
Highlight the specific risks and gotchas before anyone edits this code.

Cover:
- **Hidden dependencies**: What external state, globals, or implicit assumptions does this rely on?
- **Fragile logic**: Parts of the code that are load-bearing but don't look like they are
- **Side effects**: Mutations, I/O, or state changes that callers might not expect
- **Implicit contracts**: Undocumented assumptions about input format, ordering, or environment
- **Regression risk**: What's most likely to break if modified carelessly?

Format as a bulleted list for scannability.

---

### 5. 🔧 How to Safely Modify It
Concrete, actionable guidance for the most common modification scenarios.

Always include:
- What to do **before** touching the code (e.g., write a characterization test, check for callers, read related config)
- The **safest first change** to make (e.g., extract a helper, rename a variable, add a guard clause)
- Patterns to **avoid** when modifying (e.g., don't inline this loop — order matters)
- If a full refactor is warranted, suggest a **phased approach**

---

## Handling Special Cases

### Very short code (< 15 lines)
Skip the full five-section structure. Give a 2–3 paragraph conversational explanation covering: what it does, any non-obvious behavior, and one tip for safe modification.

### Very long code (> 200 lines, multi-function modules)
- Start with a **module-level summary** before diving in
- Group the walkthrough by logical section/function rather than line-by-line
- In "What to Watch Out For", focus on **inter-function dependencies** and **shared state**

### Unknown language or framework
- Name the language/framework if identifiable; say so if not
- Explain any language-specific idioms that would be unfamiliar to someone coming from a mainstream language (Python, JS, Java)

### Code with obvious bugs
- Explain what the code *tries* to do, then note the bug separately with a `🐛 Possible Bug` callout
- Do not silently "fix" the explanation to hide the bug

---

## Tone and Style

- Write for a smart developer who is **unfamiliar with this specific codebase**, not a beginner
- Be direct. Avoid filler phrases like "Great question!" or "This code is quite interesting"
- Use **bold** to highlight key terms, variable names, and risk areas
- Prefer concrete examples over abstract descriptions when explaining behavior
- If something is genuinely unclear even after analysis, say so honestly rather than guessing confidently

---

## Example Invocation

**User:** *"I found this in our codebase. No one knows what it does. Can you explain it?"*

```python
def xform(d, _cache={}):
    k = tuple(sorted(d.items()))
    if k not in _cache:
        _cache[k] = {v: i for i, v in enumerate(sorted(set(d.values())))}
    return {key: _cache[k][val] for key, val in d.items()}
```

**Expected output:** Full five-section breakdown covering: label-encoding a dict (summary), the mutable default arg cache trick (walkthrough), why this pattern exists for performance (why it exists), the shared-cache gotcha across calls (watch out), and how to safely replace with a class or `functools.lru_cache` (how to modify).