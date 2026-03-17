---
name: anti-purple-ui
description: Enforce a strict monochrome UI with a single high-contrast accent color, removing generic tech gradients and “AI-style” palettes. Use when the user wants minimal, anti-AI, or non-generic aesthetics, or says the UI looks too techy or generic.
category: "Design"
---

# Anti-Purple UI

This skill enforces a **strict monochrome-plus-one-accent** color discipline. It exists to eliminate the default AI/SaaS aesthetic: purple gradients, blue-violet glows, teal-to-indigo sweeps, frosted glass with color tints, and multi-hued palettes masquerading as "minimal."

---

## The Rule

> **Grayscale everywhere. One accent. No exceptions.**

The entire UI is built from a neutral grayscale scale. A single accent color — chosen deliberately and used sparingly — provides all contrast and emphasis. Everything else is black, white, or gray.

---

## Color System

### Grayscale Scale (CSS variables — always define these)

```css
:root {
  --color-bg:        #0a0a0a;  /* near-black background */
  --color-surface:   #141414;  /* card / panel surface */
  --color-border:    #2a2a2a;  /* subtle dividers */
  --color-muted:     #555555;  /* disabled, placeholder */
  --color-secondary: #888888;  /* secondary text */
  --color-primary:   #e0e0e0;  /* primary text */
  --color-white:     #f5f5f5;  /* headings, high-emphasis */

  /* For light-mode variants, invert the scale: */
  /* --color-bg: #f5f5f5; --color-white: #0a0a0a; etc. */
}
```

### The Accent

Define **one** accent color. Pick it based on context and intended mood. Do not pick it arbitrarily — the choice should feel intentional.

```css
:root {
  --color-accent: /* ONE color only */;
  --color-accent-dim: /* same hue, lower saturation/lightness for hover states */;
}
```

**Good accent choices by mood:**

| Mood / Context | Accent |
|---|---|
| Brutal / industrial | `#ff3300` (raw red) |
| Technical / terminal | `#00ff41` (phosphor green) |
| Financial / serious | `#f0b429` (amber gold) |
| Editorial / print | `#e8113a` (ink red) |
| Cold / clinical | `#00bfff` (ice blue) |
| Luxury / minimal | `#c9a84c` (warm gold) |
| Warning / urgent | `#ff6b00` (signal orange) |
| Scientific | `#7fff00` (chartreuse) |

**Forbidden accent choices** (these ARE the AI aesthetic):
- ❌ Purple (`#7c3aed`, `#6366f1`, `#8b5cf6`, any violet)
- ❌ Blue-violet, indigo, lavender
- ❌ Teal-to-indigo transitions
- ❌ "Brand blue" (`#3b82f6`, `#2563eb`)
- ❌ Pastel anything
- ❌ Multiple accent colors ("just a secondary accent" — no)

---

## Usage Rules

### Accent is for emphasis only
The accent appears on:
- The single most important interactive element (primary CTA button)
- Active / selected state indicators
- One key data point or metric that needs to stand out
- Focused input borders
- Progress / loading indicators

The accent does **not** appear on:
- Backgrounds (other than tiny badges or pills)
- Large surface areas
- Decorative elements
- More than ~10% of total visible UI area

### No gradients. Period.
```css
/* ❌ Never */
background: linear-gradient(135deg, #667eea, #764ba2);
background: linear-gradient(to right, var(--color-accent), purple);

/* ✅ Always */
background: var(--color-surface);
border-left: 3px solid var(--color-accent); /* accent as line, not fill */
```

### No color shadows or glows
```css
/* ❌ Never */
box-shadow: 0 0 20px rgba(124, 58, 237, 0.4);
text-shadow: 0 0 10px #7c3aed;

/* ✅ If shadows, always neutral */
box-shadow: 0 4px 16px rgba(0, 0, 0, 0.5);
```

### No frosted glass with tints
```css
/* ❌ Never */
background: rgba(99, 102, 241, 0.1);
backdrop-filter: blur(12px);

/* ✅ If glass effect, neutral only */
background: rgba(255, 255, 255, 0.04);
backdrop-filter: blur(8px);
border: 1px solid var(--color-border);
```

---

## Contrast & Accessibility

With a monochrome system, contrast must come from **value**, not hue. Ensure:
- Body text (`--color-primary`) hits WCAG AA against `--color-bg`
- Accent color on dark background hits WCAG AA (4.5:1 for text, 3:1 for UI components)
- Don't rely on color alone to communicate state — use icons, labels, or weight changes too

---

## Component Patterns

### Buttons
```css
/* Primary — the ONE place accent lives prominently */
.btn-primary {
  background: var(--color-accent);
  color: var(--color-bg);  /* or --color-white depending on accent lightness */
  border: none;
}

/* Secondary — monochrome only */
.btn-secondary {
  background: transparent;
  border: 1px solid var(--color-border);
  color: var(--color-primary);
}
.btn-secondary:hover {
  border-color: var(--color-primary);
}
```

### Cards / Surfaces
```css
.card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  /* Optional: accent left-border for featured/highlighted card */
}
.card--featured {
  border-left: 3px solid var(--color-accent);
}
```

### Data / Stats
```css
/* Accent only on the ONE metric that matters most */
.stat-value--primary { color: var(--color-accent); }
.stat-value--secondary { color: var(--color-white); }
.stat-label { color: var(--color-secondary); }
```

---

## Light Mode Variant

Invert the scale and keep the same accent discipline:

```css
:root[data-theme="light"] {
  --color-bg:        #f8f8f8;
  --color-surface:   #ffffff;
  --color-border:    #e0e0e0;
  --color-muted:     #b0b0b0;
  --color-secondary: #666666;
  --color-primary:   #1a1a1a;
  --color-white:     #0a0a0a;
  /* --color-accent stays the same */
}
```

---

## Checklist Before Delivering

Before finalizing any UI built under this skill, verify:

- [ ] Zero purple, violet, indigo, or teal-to-purple gradients anywhere
- [ ] Exactly one accent color defined and used
- [ ] Accent covers ≤10% of total visual area
- [ ] No colored shadows or glows
- [ ] No multi-stop gradients (single-color tints at low opacity are OK on surfaces)
- [ ] All interactive states expressed through value/weight changes, not new colors
- [ ] Background and surface colors are neutral grays/blacks/whites only
- [ ] Frosted glass (if used) has no color tint

---

## Working with `frontend-design` skill

This skill **overrides** the color section of `frontend-design`. Everything else in `frontend-design` (typography, layout, motion, spatial composition) still applies. When both are active:

1. Read `frontend-design` for overall creative direction
2. Apply this skill's color system as a strict constraint on top
3. The accent color should be chosen to match the chosen aesthetic tone (e.g., a brutalist layout pairs well with raw red; an editorial layout with ink red or amber)