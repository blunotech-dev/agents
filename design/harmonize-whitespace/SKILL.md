---
name: harmonize-whitespace
description: Align all spacing (padding, margins, gaps) to a consistent 4pt/8pt grid. Use when spacing feels off, inconsistent, cramped, or unbalanced, or when the user asks for a spacing scale or alignment fix.
category: "Design"
---

# Harmonize Whitespace

This skill enforces a **strict 4pt/8pt spatial grid**. Every padding, margin, gap, and size value snaps to a multiple of 4px. No arbitrary values like `13px`, `22px`, or `37px` — ever.

---

## The Rule

> **All spacing is a multiple of 4. Prefer multiples of 8 for major spacing.**

The 4pt grid is the foundation. The 8pt grid (every other step) handles most layout spacing. The 4pt steps exist for tight, fine-grained control (icon padding, badge insets, input inner spacing).

---

## The Scale

Define this token set at the root of every project:

```css
:root {
  --space-1:  4px;   /* micro — icon padding, badge insets */
  --space-2:  8px;   /* xs — tight inline gaps, input padding */
  --space-3:  12px;  /* sm — compact component padding */
  --space-4:  16px;  /* md — standard component padding (base unit) */
  --space-5:  20px;  /* — use sparingly, prefer 16 or 24 */
  --space-6:  24px;  /* lg — card padding, section gaps */
  --space-8:  32px;  /* xl — between components */
  --space-10: 40px;  /* 2xl — section padding */
  --space-12: 48px;  /* 3xl — large section gaps */
  --space-16: 64px;  /* 4xl — page-level vertical rhythm */
  --space-20: 80px;  /* 5xl — hero/feature sections */
  --space-24: 96px;  /* 6xl — major page sections */
}
```

> **Note on `--space-5` (20px):** It's on the 4pt grid but not the 8pt grid. Use it only when 16px is too tight and 24px is too loose — typically for asymmetric padding (e.g., 20px horizontal, 16px vertical on buttons). Prefer 8pt steps everywhere else.

---

## Tailwind Users

If using Tailwind, the default spacing scale is already 4pt-based. Use these classes:

| Token | px  | Tailwind class |
|-------|-----|----------------|
| 1     | 4   | `p-1`, `m-1`, `gap-1` |
| 2     | 8   | `p-2`, `m-2`, `gap-2` |
| 3     | 12  | `p-3`, `m-3`, `gap-3` |
| 4     | 16  | `p-4`, `m-4`, `gap-4` |
| 6     | 24  | `p-6`, `m-6`, `gap-6` |
| 8     | 32  | `p-8`, `m-8`, `gap-8` |
| 10    | 40  | `p-10`, `m-10`, `gap-10` |
| 12    | 48  | `p-12`, `m-12`, `gap-12` |
| 16    | 64  | `p-16`, `m-16`, `gap-16` |
| 20    | 80  | `p-20`, `m-20`, `gap-20` |
| 24    | 96  | `p-24`, `m-24`, `gap-24` |

**Never use arbitrary Tailwind values** like `p-[13px]`, `mt-[22px]`, or `gap-[7px]`.

---

## Application Rules by Context

### Component Padding

| Component type | Recommended padding |
|---|---|
| Compact / dense (tables, badges, tags) | `--space-1` / `--space-2` (4–8px) |
| Buttons (small) | `8px 12px` → `var(--space-2) var(--space-3)` |
| Buttons (default) | `12px 20px` or `12px 24px` → `var(--space-3) var(--space-5/6)` |
| Buttons (large) | `16px 32px` → `var(--space-4) var(--space-8)` |
| Input fields | `8px 12px` or `12px 16px` |
| Cards (small) | `--space-4` (16px) all sides |
| Cards (default) | `--space-6` (24px) all sides |
| Cards (large / feature) | `--space-8` to `--space-10` |
| Modals / dialogs | `--space-8` (32px) |
| Page horizontal padding | `--space-6` to `--space-16` depending on breakpoint |

### Gap / Layout Spacing

| Context | Recommended gap |
|---|---|
| Inline icon + text | `--space-2` (8px) |
| Form field stack | `--space-4` to `--space-6` (16–24px) |
| Card grid | `--space-6` to `--space-8` (24–32px) |
| Section stack on page | `--space-12` to `--space-20` (48–80px) |
| Hero / CTA sections | `--space-16` to `--space-24` (64–96px) |

### Vertical Rhythm (Typography)

Line heights and paragraph spacing should also snap to the grid:

```css
/* Body text: 16px size, 24px line-height (both on grid) */
body { font-size: 16px; line-height: 24px; }

/* Headings */
h1 { font-size: 48px; line-height: 56px; margin-bottom: 24px; }
h2 { font-size: 32px; line-height: 40px; margin-bottom: 16px; }
h3 { font-size: 24px; line-height: 32px; margin-bottom: 12px; }
h4 { font-size: 20px; line-height: 28px; margin-bottom: 8px; }

/* Paragraph spacing */
p + p { margin-top: 16px; }
```

---

## Common Anti-Patterns to Fix

### ❌ Arbitrary values
```css
/* Bad */
padding: 13px 17px;
margin-top: 22px;
gap: 7px;
border-radius: 6px; /* OK — border-radius has its own conventions, see below */
```

```css
/* Good */
padding: 12px 16px;
margin-top: 24px;
gap: 8px;
```

### ❌ Inconsistent asymmetry
```css
/* Bad — random asymmetry */
padding: 14px 20px 10px 18px;
```

```css
/* Good — intentional asymmetry, all on-grid */
padding: 12px 24px 16px 24px;
```

### ❌ Off-grid line heights
```css
/* Bad */
line-height: 1.45; /* resolves to 23.2px at 16px — off-grid */
```

```css
/* Good */
line-height: 1.5; /* resolves to 24px at 16px — on-grid */
```

---

## Border Radius

Border radius is **not** strictly on the spacing grid — it follows its own visual scale. However, use consistent tokens rather than arbitrary values:

```css
:root {
  --radius-sm: 4px;   /* subtle rounding — inputs, badges */
  --radius-md: 8px;   /* default — cards, buttons */
  --radius-lg: 12px;  /* softer — modals, panels */
  --radius-xl: 16px;  /* pronounced — floating elements */
  --radius-full: 9999px; /* pill — tags, toggles */
}
```

---

## Responsive Scaling

Scale spacing up at larger breakpoints — always staying on-grid:

```css
.container {
  padding: var(--space-4);               /* 16px mobile */
}

@media (min-width: 768px) {
  .container { padding: var(--space-8); } /* 32px tablet */
}

@media (min-width: 1280px) {
  .container { padding: var(--space-16); } /* 64px desktop */
}
```

Spacing jumps between breakpoints should be **at least one full step** on the scale — don't change padding from 16px to 20px across breakpoints; go 16px → 24px or 16px → 32px.

---

## Working with Other Skills

- **`frontend-design`**: This skill governs spacing. `frontend-design` governs layout composition, typography choices, and motion. They are fully compatible — apply both simultaneously.
- **When auditing existing UI**: Check every `padding`, `margin`, `gap`, `top/right/bottom/left` offset, and `width/height` value against the scale. Flag any value not divisible by 4.

---

## Checklist Before Delivering

- [ ] All `padding` values are multiples of 4px
- [ ] All `margin` values are multiples of 4px
- [ ] All `gap` / `grid-gap` / `column-gap` / `row-gap` values are multiples of 4px
- [ ] All positional offsets (`top`, `left`, etc.) are multiples of 4px
- [ ] No arbitrary Tailwind spacing values (`p-[Xpx]` etc.)
- [ ] Line heights resolve to multiples of 4px at their given font size
- [ ] Spacing scales up correctly across breakpoints (full steps, not micro-bumps)
- [ ] `--space-*` CSS variables (or Tailwind equivalents) used consistently — no magic numbers inline