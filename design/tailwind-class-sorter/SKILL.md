---
name: tailwind-class-sorter
description: Sort Tailwind CSS utility classes into a clear, consistent order (layout, spacing, sizing, typography, visual). Use when classes are messy, hard to read, or when the user asks to clean up or organize Tailwind code.
category: "Design"
---

# Tailwind Class Sorter

Sorts Tailwind CSS utility classes into a consistent, human-readable hierarchy that matches how developers reason about UI: structure first, then spacing, sizing, text, and finally appearance.

---

## The Canonical Sort Order

Classes are sorted into **6 buckets**, processed strictly left-to-right. Within each bucket, classes preserve their original relative order (stable sort) unless they are duplicates (which are removed).

### Bucket Order

| # | Bucket | What Goes Here |
|---|--------|----------------|
| 1 | **Layout** | Display, position, z-index, overflow, flex/grid setup, columns |
| 2 | **Spacing** | Margin, padding (all sides, axes, individual) |
| 3 | **Sizing** | Width, height, min/max-width, min/max-height, aspect ratio |
| 4 | **Typography** | Font, text size, weight, leading, tracking, alignment, color (`text-*` colors), decoration |
| 5 | **Colors & Background** | Background color/image/gradient, opacity |
| 6 | **Borders & Effects** | Border, ring, shadow, rounded, outline, transition, animation, cursor, pointer-events |

Arbitrary values (e.g. `w-[320px]`, `top-[var(--offset)]`) belong to the same bucket as their base utility.

Responsive variants (`sm:`, `md:`, `lg:`, `xl:`, `2xl:`) and state variants (`hover:`, `focus:`, `active:`, `disabled:`, `dark:`, `group-hover:`, etc.) stay attached to their base class and sort with it.

---

## Detailed Prefix → Bucket Mapping

Use this as the lookup table when sorting:

### Bucket 1 — Layout
```
block inline inline-block flex inline-flex grid inline-grid hidden table
contents flow-root list-item
static relative absolute fixed sticky
inset- top- right- bottom- left-
z-
overflow- overscroll-
float- clear-
object-
flex- flex-row flex-col flex-wrap flex-nowrap justify- items- self- place-
gap- gap-x- gap-y-
grid-cols- grid-rows- col- row- auto-cols- auto-rows-
order-
columns-
container
isolate isolation-
mix-blend- bg-blend-
group peer (standalone modifiers)
```

### Bucket 2 — Spacing
```
m- mx- my- mt- mr- mb- ml-
p- px- py- pt- pr- pb- pl-
space-x- space-y-
```

### Bucket 3 — Sizing
```
w- min-w- max-w-
h- min-h- max-h-
size-
aspect-
```

### Bucket 4 — Typography
```
font- (font-sans, font-bold, font-mono, etc.)
text-xs text-sm text-base text-lg text-xl text-2xl ...  (size)
text-left text-center text-right text-justify text-start text-end  (alignment)
text-{color}-{shade}  (text color lives near text size/weight for readability)
leading- tracking- indent-
uppercase lowercase capitalize normal-case
truncate line-clamp- whitespace- break-
list-  (list-disc, list-decimal, etc.)
underline no-underline line-through decoration-
subpixel-antialiased antialiased
italic not-italic
tabular-nums
```

### Bucket 5 — Colors & Background
```
bg-  (bg-red-500, bg-white, bg-transparent, bg-gradient-to-*, etc.)
from- via- to-  (gradient stops)
opacity-
fill- stroke-
```

### Bucket 6 — Borders & Effects
```
border border-t border-r border-b border-l border-x border-y
border-{color}-{shade} border-{width}
rounded- rounded-t- rounded-r- rounded-b- rounded-l- rounded-tl- ...
outline- outline-offset-
ring- ring-offset-
shadow-
divide- divide-{color}
transition- duration- ease- delay-
animate-
scale- rotate- translate- skew-
origin-
cursor-
pointer-events-
select-
resize-
appearance-
will-change-
sr-only not-sr-only
```

---

## Algorithm

```
1. Tokenize the class string by whitespace.
2. Deduplicate — keep first occurrence, silently drop repeats.
3. For each token, strip variant prefixes (sm:, hover:, etc.) to get the base utility.
4. Assign each token to a bucket (1–6) using the mapping above.
   - Unknown or custom classes → append at the END, after bucket 6.
5. Stable-sort within each bucket (preserve original relative order).
6. Concatenate: bucket1 bucket2 bucket3 bucket4 bucket5 bucket6 unknowns.
7. Return as a single space-separated string.
```

---

## Output Format Rules

- **Raw class string input** → return only the sorted class string.
- **HTML/JSX input** → return the full element or component with sorted classes inline. Preserve all other code exactly; only reorder the class attribute values.
- **Do not rename, remove (except duplicates), or add** any classes.
- **Multiple elements** → sort each element's classes independently.
- Preserve quote style (`"` vs `'` vs template literals).
- End with a one-line summary: *"Sorted 14 classes across 5 buckets; removed 1 duplicate (`text-white`)."*

---

## Examples

### Example 1 — Raw class string

**Input:**
```
shadow-md text-white p-4 flex bg-blue-600 rounded-lg font-bold hover:bg-blue-700 text-sm items-center gap-2 w-full
```

**Output:**
```
flex items-center gap-2 p-4 w-full text-sm font-bold text-white bg-blue-600 hover:bg-blue-700 rounded-lg shadow-md
```

**Reasoning:**
- `flex items-center gap-2` → Layout
- `p-4` → Spacing
- `w-full` → Sizing
- `text-sm font-bold text-white` → Typography
- `bg-blue-600 hover:bg-blue-700` → Colors & Background
- `rounded-lg shadow-md` → Borders & Effects

---

### Example 2 — JSX className

**Input:**
```jsx
<button className="hover:bg-green-700 font-semibold rounded px-6 py-2 bg-green-500 text-white inline-flex shadow items-center">
  Submit
</button>
```

**Output:**
```jsx
<button className="inline-flex items-center px-6 py-2 font-semibold text-white bg-green-500 hover:bg-green-700 rounded shadow">
  Submit
</button>
```

---

### Example 3 — Responsive & state variants

**Input:**
```
md:flex-row flex-col sm:text-lg focus:ring-2 border text-base p-2 bg-white md:p-6
```

**Output:**
```
flex-col md:flex-row p-2 md:p-6 text-base sm:text-lg bg-white border focus:ring-2
```

---

## Edge Cases

| Situation | Handling |
|-----------|----------|
| Duplicate classes | Keep first occurrence; silently remove subsequent ones |
| Unknown/custom class (e.g. `my-custom-btn`) | Append after bucket 6 |
| `!important` modifier (`!p-4`) | Sort with base utility bucket; keep `!` prefix intact |
| `group`, `peer` standalone | Treat as Layout |
| `group-hover:`, `peer-focus:` variants | Sort with base utility; prefix stays attached |
| Arbitrary values (`w-[320px]`, `bg-[--brand]`) | Sort with base utility bucket |
| Empty or whitespace-only input | Return empty string |
| Conflicting classes (e.g. `text-sm text-lg`) | Preserve both; note the conflict to the user |