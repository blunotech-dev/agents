---
name: typographic-hierarchy
description: Improve typography by adjusting font sizes, weights, spacing, and contrast to create clear visual hierarchy and readability. Use when text feels flat, unstructured, or when the user asks to refine headings, type scale, or overall readability.
category: "Design"
---

# Typographic Hierarchy Skill

This skill establishes clear, intentional typographic hierarchy — using scale, weight, spacing, color contrast, and rhythm to guide the reader's eye and communicate structure at a glance.

---

## Core Concepts

Typographic hierarchy works across **six levers**. You rarely need all six — pick the fewest levers that achieve the contrast needed:

| Lever | Effect |
|---|---|
| **Size** | The primary signal. Double the size = double the dominance. |
| **Weight** | Bold reads first. 700+ for dominant labels, 300–400 for supporting copy. |
| **Color / Opacity** | Muted gray for secondary text. High-contrast black/white for focal points. |
| **Letter-spacing** | Tight for large display text. Wide (0.05–0.2em) for ALL-CAPS labels. |
| **Line-height** | Looser (1.6–1.8) for body. Tighter (1.0–1.2) for large headings. |
| **Space (margin/padding)** | Space above a heading = visual importance. Use generously. |

> **Rule of thumb**: Each level of the hierarchy should be *unmistakably* different from the next. If you squint and two levels look similar, increase the contrast.

---

## Type Scale Systems

### Modular Scale (recommended for precision)
Pick a **ratio** and a **base size**, then multiply up and divide down.

Common ratios:
- `1.25` — Major Third (subtle, compact)
- `1.333` — Perfect Fourth (balanced, versatile)
- `1.414` — Augmented Fourth (editorial, dramatic)
- `1.5` — Perfect Fifth (loud, impactful)
- `1.618` — Golden Ratio (classic, organic feel)

**Example — Perfect Fourth (1.333) at base 16px:**
```
xs:   10px  (0.625rem)   — captions, meta
sm:   12px  (0.75rem)    — labels, tags
base: 16px  (1rem)       — body
md:   21px  (1.333rem)   — subheadings
lg:   28px  (1.777rem)   — section headings
xl:   37px  (2.369rem)   — page headings
2xl:  50px  (3.157rem)   — hero / display
```

### Practical Defaults (when in doubt, use these)
```css
--text-xs:   0.75rem;   /* 12px */
--text-sm:   0.875rem;  /* 14px */
--text-base: 1rem;      /* 16px */
--text-lg:   1.25rem;   /* 20px */
--text-xl:   1.5rem;    /* 24px */
--text-2xl:  2rem;      /* 32px */
--text-3xl:  2.5rem;    /* 40px */
--text-4xl:  3.5rem;    /* 56px */
```

---

## Weight Pairings

| Role | Weight | Notes |
|---|---|---|
| Display / Hero | 800–900 | Black or ExtraBold — for maximum punch |
| H1 / Page title | 700 | Bold |
| H2 / Section heading | 600–700 | SemiBold–Bold |
| H3 / Subheading | 500–600 | Medium–SemiBold |
| Body | 400 | Regular |
| Caption / Meta | 300–400 | Light or Regular; often muted color |
| Labels / Tags | 500–600 + uppercase | Uppercase + tracking lifts small text |

**Anti-pattern**: Never use the same weight for two adjacent levels of hierarchy. Even one step (e.g., 400→500) matters when combined with size contrast.

---

## Color & Contrast Hierarchy

Use opacity or gray tones to create a foreground/midground/background for text:

```css
/* Dark theme example */
--text-primary:   rgba(255, 255, 255, 1.00);   /* headings, key labels */
--text-secondary: rgba(255, 255, 255, 0.70);   /* body, descriptions */
--text-tertiary:  rgba(255, 255, 255, 0.45);   /* captions, timestamps */
--text-disabled:  rgba(255, 255, 255, 0.25);   /* placeholders */

/* Light theme example */
--text-primary:   rgba(0, 0, 0, 0.92);
--text-secondary: rgba(0, 0, 0, 0.60);
--text-tertiary:  rgba(0, 0, 0, 0.38);
--text-disabled:  rgba(0, 0, 0, 0.20);
```

**Accent color for hierarchy**: Use a brand or accent color *sparingly* — only for one level (usually links, CTAs, or key labels). Overusing accent color flattens hierarchy.

---

## Spacing & Rhythm

Vertical rhythm is as important as size. Space *above* a heading signals a new section; space *below* links it to what follows.

```css
/* Heading spacing pattern */
h1 { margin-top: 0;      margin-bottom: 0.5em; }
h2 { margin-top: 2.5em;  margin-bottom: 0.4em; }
h3 { margin-top: 2em;    margin-bottom: 0.3em; }
p  { margin-top: 0;      margin-bottom: 1em;   }
```

**Tight-to-loose rhythm**: Large display text → tight line-height (1.0–1.2). Body copy → comfortable (1.5–1.7). Long-form prose → generous (1.7–1.9).

---

## Implementation Patterns

### CSS Custom Properties (Design System pattern)
```css
:root {
  /* Scale */
  --type-display: clamp(2.5rem, 6vw, 5rem);
  --type-h1:      clamp(1.75rem, 4vw, 3rem);
  --type-h2:      clamp(1.25rem, 2.5vw, 2rem);
  --type-h3:      1.125rem;
  --type-body:    1rem;
  --type-small:   0.875rem;
  --type-caption: 0.75rem;

  /* Weights */
  --weight-display: 800;
  --weight-heading: 700;
  --weight-subhead: 600;
  --weight-body:    400;
  --weight-caption: 400;

  /* Leading */
  --leading-tight:  1.1;
  --leading-snug:   1.35;
  --leading-normal: 1.6;
  --leading-loose:  1.8;

  /* Tracking */
  --tracking-tight:  -0.03em;
  --tracking-normal:  0em;
  --tracking-wide:    0.06em;
  --tracking-wider:   0.12em;
}
```

### Tailwind Utility Mapping
```
Display:    text-5xl md:text-7xl font-black tracking-tight leading-none
H1:         text-3xl md:text-5xl font-bold tracking-tight leading-tight
H2:         text-2xl md:text-3xl font-semibold leading-snug
H3:         text-xl font-semibold leading-snug
Body:       text-base font-normal leading-relaxed text-gray-700
Caption:    text-sm font-normal text-gray-400 leading-normal
Label/tag:  text-xs font-semibold uppercase tracking-widest text-gray-500
```

### React Component (reusable type system)
```jsx
const typeStyles = {
  display: 'text-6xl font-black tracking-tight leading-none',
  h1:      'text-4xl font-bold tracking-tight leading-tight',
  h2:      'text-2xl font-semibold leading-snug',
  h3:      'text-xl font-semibold leading-snug',
  body:    'text-base leading-relaxed',
  small:   'text-sm leading-normal',
  caption: 'text-xs text-gray-400',
  label:   'text-xs font-semibold uppercase tracking-widest',
};

const Text = ({ as: Tag = 'p', variant = 'body', className = '', ...props }) => (
  <Tag className={`${typeStyles[variant]} ${className}`} {...props} />
);
```

---

## Auditing Existing Typography

When asked to review or fix existing type hierarchy, check these in order:

1. **Squint test** — Blur your eyes. Can you read the structure? If not, contrast is insufficient.
2. **Size gap** — Is there at least a 1.25× difference between adjacent levels? If heading and body look close in size, bump the heading.
3. **Weight flattening** — Are multiple levels using `font-weight: 400`? Add weight variation.
4. **Color sameness** — Is all text the same darkness? Introduce secondary/tertiary tones.
5. **Spacing starvation** — Do headings have enough breathing room above them? Tighten the bond between heading and its following content; open space above.
6. **Responsive collapse** — Does hierarchy survive on mobile? Use `clamp()` or responsive variants to preserve ratios at small sizes.

---

## Common Mistakes & Fixes

| Mistake | Fix |
|---|---|
| H1 and H2 look almost identical | Increase H1 size by at least 1.5× H2; add extra weight or tighter tracking |
| Body text too light/thin | Set body to 400 weight; avoid 300 for long-form copy |
| Uppercase headings feel heavy | Reduce weight to 600; increase letter-spacing to 0.08–0.12em |
| Display text looks cramped | Set `line-height: 0.95–1.1` and `letter-spacing: -0.03em` |
| All text is same color | Apply 3-tier opacity system (100%, 65%, 40%) |
| Mobile hierarchy collapses | Use `clamp()` for fluid scaling; test at 375px width |
| Too many font sizes | Collapse to 4–5 levels max; enforce the scale |

---

## Font Pairing for Hierarchy

When font selection is in scope:

- **Display + Body**: Pair a high-contrast serif (for display) with a clean sans (for body). E.g., Playfair Display + Source Sans, Fraunces + Inter.
- **Weight contrast within one family**: Use a variable font (e.g., Satoshi, Cabinet Grotesk, Geist) and span weights 300–900.
- **Monospace accent**: Use a mono font for labels/code/metadata — creates instant visual separation from prose.

> See `references/font-pairs.md` for curated pairings with Google Fonts import snippets.

---

## Accessibility Notes

- Minimum body text: **16px** (never go below 14px for readable content)
- WCAG AA contrast: **4.5:1** for body text, **3:1** for large text (18px+ or 14px bold)
- Never rely on color alone to establish hierarchy — always pair with size or weight
- Use semantic HTML (`h1`–`h6`, `p`, `small`) even when visual styles differ

---

## Checklist Before Shipping

- [ ] Each heading level is unmistakably different from the next (size + weight)
- [ ] Body text is 400 weight, 15–17px, line-height 1.5–1.7
- [ ] Secondary/caption text is visually subordinate (lighter color or smaller)
- [ ] Headings have appropriate space above them
- [ ] Hierarchy survives on mobile (375px viewport)
- [ ] Contrast ratios pass WCAG AA
- [ ] No more than 5 distinct type sizes in use