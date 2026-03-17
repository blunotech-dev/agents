---
name: consistent-border-radius
description: Normalizes rounded corners across a file so buttons, inputs, cards, modals, badges, and all UI elements share the exact same curvature. Use this skill whenever the user mentions inconsistent border radii, wants to unify rounded corners, asks to make UI elements look more cohesive, or says things like "make the corners match", "fix the rounding", "unify border radius", "standardize my rounded corners", or "buttons and cards don't match". Also trigger when the user pastes a CSS/HTML/JSX/TSX file and asks for a design consistency pass, border radius is one of the first things to normalize.
category: "Design"
---

# Consistent Border Radius

This skill audits a frontend file (CSS, HTML, JSX, TSX, Vue, Svelte, or plain JS) and rewrites all border-radius values so every UI element — buttons, inputs, cards, modals, badges, tooltips, dropdowns — uses the same curvature token.

---

## When to Use

Trigger on any of these signals:
- User mentions mismatched, inconsistent, or ugly rounded corners
- User wants a "design consistency pass" or "visual polish"
- User says elements "don't match", "look off", or "feel inconsistent"
- File has scattered `border-radius` values (`4px` here, `8px` there, `0.5rem` somewhere else)
- User is cleaning up a design system or component library

---

## Step 1 — Audit the File

Read through the entire file and collect **every** border-radius value. Build a mental inventory:

| Selector / Element | Current Value |
|--------------------|---------------|
| `.btn`             | `4px`         |
| `.card`            | `12px`        |
| `input`            | `6px`         |
| `.badge`           | `9999px`      |
| `.modal`           | `8px`         |

Note which values are:
- **Structural radii** — small/medium values for rectangles (buttons, inputs, cards, modals)
- **Pill/full-round radii** — large values like `9999px`, `50%`, `100px` used on tags, avatars, toggles

Do **not** flatten pill radii into the structural radius — they serve a different semantic purpose.

---

## Step 2 — Choose the Canonical Radius

Pick **one** canonical value for structural elements. Use this priority order:

1. **User-specified** — if they said "use 8px" or "I want `0.5rem`", use that.
2. **Design system token** — if the file already has a `--border-radius` or `--radius` CSS variable, adopt it.
3. **Most common value** — pick the value that already appears the most.
4. **Sensible default** — if values are all over the place, default to `8px` (or `0.5rem`) for a modern, balanced look.

Always confirm the chosen value with a brief inline comment in your output, e.g.:
```
/* Canonical border-radius: 8px — applied consistently across all UI elements */
```

---

## Step 3 — Normalize

Apply the canonical radius to all structural elements. Rules:

### CSS / SCSS / Less
Replace scattered values with the canonical one. If there's already a variable, redirect everything to it. If there isn't one, **create one** at the `:root` level:

```css
:root {
  --radius: 8px;
}

.btn      { border-radius: var(--radius); }
.card     { border-radius: var(--radius); }
input     { border-radius: var(--radius); }
.modal    { border-radius: var(--radius); }
```

### Tailwind CSS (HTML / JSX / TSX)
Replace mixed utility classes (`rounded`, `rounded-md`, `rounded-xl`, `rounded-lg`) with a single consistent class. Map to the closest Tailwind token:

| Canonical px | Tailwind class  |
|--------------|-----------------|
| 2px          | `rounded-sm`    |
| 4px          | `rounded`       |
| 6px          | `rounded-md`    |
| 8px          | `rounded-lg`    |
| 12px         | `rounded-xl`    |
| 16px         | `rounded-2xl`   |
| 9999px       | `rounded-full`  ← keep this for pills/avatars

Leave `rounded-full` alone — it's intentional.

### Inline styles (JSX / TSX)
Replace `style={{ borderRadius: '...' }}` values with the canonical one.

### CSS-in-JS (styled-components, emotion, etc.)
Replace string/template literal values:
```js
// Before
border-radius: 12px;
border-radius: ${theme.radii.sm};

// After
border-radius: ${theme.radii.base}; /* or var(--radius) */
```

---

## Step 4 — Preserve Intentional Exceptions

**Never change** these — they're semantically distinct:
- `border-radius: 50%` → circles (avatars, icons)
- `border-radius: 9999px` / `rounded-full` → pill shapes (tags, toggles, chips)
- `border-radius: 0` → deliberately sharp corners (e.g., full-bleed banners, dividers)
- Per-corner overrides like `border-top-left-radius` used for design effects (e.g., chat bubbles, tabs)

If you're unsure whether a value is intentional, leave it and add a comment:
```css
/* NOTE: kept at 50% — appears intentional (circle avatar) */
```

---

## Step 5 — Output

Return the **full modified file** with:
1. A CSS variable or token introduced at the top (if CSS-based)
2. All structural radii normalized
3. A brief summary comment near the top explaining the canonical value chosen
4. Inline `/* unchanged — intentional */` comments on any preserved exceptions

Also provide a short **change summary** in your response (outside the code block):
```
Applied canonical border-radius of 8px (var(--radius)) to 14 elements.
Preserved: 2× rounded-full (pill badges), 1× 50% (avatar circle).
```

---

## Edge Cases & Notes

- **Mixed units** (px vs rem): Normalize to whichever unit dominates. If the project uses `rem`, convert `8px → 0.5rem`. Use `1rem = 16px` as the conversion rate.
- **Shorthand vs longhand**: `border-radius: 8px 8px 8px 8px` → collapse to `border-radius: 8px`.
- **Nested components**: If a card has an inner image that needs `border-radius: 8px 8px 0 0` to flush with the card top — keep that, it's structural.
- **Multiple files**: If the user shares multiple files, apply the same canonical token across all of them and note any cross-file inconsistencies.
- **No radii found**: If the file has no border-radius values at all, say so clearly and ask if they'd like to add a consistent scheme from scratch.

---

## Example Transformation

### Before
```css
.btn       { border-radius: 4px; }
.card      { border-radius: 12px; }
input      { border-radius: 6px; }
.tag       { border-radius: 9999px; }
.modal     { border-radius: 10px; }
.avatar    { border-radius: 50%; }
```

### After
```css
:root {
  --radius: 8px; /* Canonical border-radius — applied consistently */
}

.btn       { border-radius: var(--radius); }
.card      { border-radius: var(--radius); }
input      { border-radius: var(--radius); }
.tag       { border-radius: 9999px; }  /* unchanged — intentional pill */
.modal     { border-radius: var(--radius); }
.avatar    { border-radius: 50%; }    /* unchanged — intentional circle */
```

**Summary**: Applied `8px` (var(--radius)) to 4 elements. Preserved 2 intentional exceptions.