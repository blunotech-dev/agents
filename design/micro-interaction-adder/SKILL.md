---
name: micro-interaction-adder
description: Add polished CSS micro-interactions like hover effects, transitions, and feedback states to improve UI feel. Use when the user asks for animations, better UX, or when the interface feels static, plain, or unresponsive.
category: "Design"
---

# Micro-Interaction Adder

This skill layers **production-grade CSS micro-interactions** onto frontend code — new or existing.
The goal: every element a user touches should *respond*, making the interface feel alive, crafted,
and trustworthy.

---

## Philosophy

Micro-interactions are not decoration. They:
- **Confirm intent** — "yes, I saw you click that"
- **Communicate state** — loading, disabled, selected, error
- **Reward attention** — subtle delight for the observant
- **Establish hierarchy** — animated elements feel more important

**Rules of thumb:**
1. Duration: `150–300ms` for most UI. `400–600ms` for entrances. Never over `800ms` for interactive feedback.
2. Easing: prefer `cubic-bezier` curves over `ease-in-out` defaults. Use spring-feel curves for scale.
3. Less is more: one well-executed hover beats five mediocre ones. Pick the 3–5 highest-impact moments.
4. Respect `prefers-reduced-motion` — always wrap animation-heavy CSS in the appropriate media query.

---

## Step-by-Step Workflow

### 1. Audit the UI

Before adding anything, scan the code for:
- Interactive elements (buttons, links, inputs, checkboxes, cards, nav items)
- State-bearing elements (loading, error, success, disabled)
- Structural elements that could benefit from entrance animations (modals, dropdowns, toasts)

### 2. Select Interaction Patterns

Choose from the **Pattern Library** below. Don't apply everything — pick what fits the component's
purpose and the overall aesthetic (see references/pattern-library.md for full recipes).

### 3. Inject the CSS

- Add transitions to base state, NOT to `:hover`/`:focus` state. This ensures smooth exit animations too.
- Use CSS custom properties (`--transition-fast`, `--color-focus-ring`) for consistency.
- Keep interaction CSS co-located with component styles, not in a separate "animations.css" blob.

### 4. Validate

Check:
- [ ] Interactions work on keyboard focus (`:focus-visible`), not just mouse hover
- [ ] Disabled states suppress all transitions (`pointer-events: none; opacity: 0.5`)
- [ ] `prefers-reduced-motion` fallback exists for all scale/translate animations
- [ ] No layout shifts — use `transform` and `opacity` only (GPU-composited)

---

## Core CSS Variables (inject into `:root`)

```css
:root {
  /* Timing */
  --duration-instant:  100ms;
  --duration-fast:     150ms;
  --duration-base:     200ms;
  --duration-slow:     300ms;
  --duration-enter:    400ms;

  /* Easing */
  --ease-out:          cubic-bezier(0.0, 0.0, 0.2, 1);
  --ease-in-out:       cubic-bezier(0.4, 0.0, 0.2, 1);
  --ease-spring:       cubic-bezier(0.34, 1.56, 0.64, 1); /* slight overshoot */
  --ease-snappy:       cubic-bezier(0.25, 0.46, 0.45, 0.94);

  /* Focus ring */
  --color-focus-ring:  #3b82f6;          /* adjust to brand color */
  --focus-ring-width:  3px;
  --focus-ring-offset: 2px;

  /* Shadows for lift effects */
  --shadow-resting:    0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.08);
  --shadow-hover:      0 4px 12px rgba(0,0,0,0.15), 0 2px 6px rgba(0,0,0,0.10);
  --shadow-active:     0 1px 2px rgba(0,0,0,0.10);
}
```

---

## Pattern Library (Quick Reference)

Full recipes with variants are in `references/pattern-library.md`.

| Pattern              | Elements                        | Key Properties                        |
|----------------------|---------------------------------|---------------------------------------|
| Hover Scale          | Cards, images, avatars          | `transform: scale(1.03)`             |
| Button Press         | All buttons                     | `scale(0.96)` on `:active`           |
| Focus Ring           | Inputs, buttons, links          | `outline` + `box-shadow` on `:focus-visible` |
| Link Underline Sweep | `<a>` tags, nav items           | `background-size` pseudo-element trick |
| Card Lift            | Content cards                   | `transform: translateY(-3px)` + shadow |
| Icon Wiggle          | Icon-only buttons, alerts       | `@keyframes wiggle` on hover         |
| Checkbox Spring      | Custom checkboxes               | `scale` + `opacity` on check state  |
| Input Focus Expand   | Text inputs, textareas          | `border-color` + `box-shadow` glow   |
| Skeleton Loader      | Loading states                  | `@keyframes shimmer` gradient sweep  |
| Fade + Slide Entrance| Modals, dropdowns, toasts       | `opacity` + `translateY` on mount    |
| Ripple Effect        | Buttons (Material-style)        | Pseudo-element radial expand         |
| Toggle Switch        | Boolean toggles                 | Smooth `translateX` + color shift    |
| Progress Bar Fill    | Upload, step indicators         | `width` + `background` transition    |
| Tooltip Appear       | Tooltips, popovers              | `opacity` + `scale(0.95)` origin     |

---

## Accessibility: `prefers-reduced-motion`

**Always** include this block when using scale, translate, or entrance animations:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

For interactions that still need *some* feedback (e.g., focus rings, color changes), leave
`color`, `background-color`, `border-color`, and `opacity` transitions intact — only strip
`transform` and motion-heavy `@keyframes`.

---

## Framework-Specific Notes

### Plain HTML/CSS
- Use the raw CSS patterns from `references/pattern-library.md` directly.
- Add `will-change: transform` to frequently animated elements (use sparingly).

### React
- For entrance animations on mount/unmount, use CSS classes + `useState` toggle, or the
  `motion` library (`framer-motion`). See `references/react-patterns.md`.
- Avoid `useEffect` + `setTimeout` hacks for animations — use CSS `@keyframes` with a
  class toggle instead.
- `transition-all` is a code smell — always specify the property explicitly.

### Tailwind CSS
- Use `transition`, `duration-{n}`, `ease-{type}`, `hover:scale-{n}`, `active:scale-{n}`,
  `focus-visible:ring-{n}` utilities.
- For custom cubic-bezier curves, extend `tailwind.config.js` `transitionTimingFunction`.
- For entrance animations not in core Tailwind, write a thin `@layer utilities` block.

### Vue / Svelte
- Use `<Transition>` (Vue) or `transition:` directive (Svelte) for enter/leave.
- CSS custom properties approach works identically.

---

## Common Mistakes to Avoid

| ❌ Mistake                              | ✅ Fix                                           |
|-----------------------------------------|--------------------------------------------------|
| `transition: all 0.3s`                 | Specify only changed properties                  |
| Putting `transition` on `:hover` only  | Put on base state for smooth exit                |
| `transform` + `position: fixed` combo  | Creates new stacking context — test carefully   |
| `box-shadow` transitions on many items | Expensive — prefer `filter: drop-shadow` or pseudo-element shadow |
| Forgetting `focus-visible`             | Don't style `:focus` alone — it fires on click  |
| `animation: spin infinite` in lists    | Decimates performance — use `will-change` sparingly |

---

## Read Next

- **`references/pattern-library.md`** — Full copy-paste CSS for every pattern in the table above,
  with light/dark variants and customization notes.
- **`references/react-patterns.md`** — React-specific hooks, `framer-motion` snippets, and
  class-toggle entrance patterns.