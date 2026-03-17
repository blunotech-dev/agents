# Pattern Library — Full CSS Recipes

All patterns use the CSS custom properties defined in the main SKILL.md `:root` block.
Each pattern includes: base snippet, variant notes, and dark mode adjustments.

---

## 1. Hover Scale (Cards, Images, Avatars)

```css
.card {
  transition:
    transform var(--duration-base) var(--ease-spring),
    box-shadow var(--duration-base) var(--ease-out);
}

.card:hover {
  transform: scale(1.03);
  box-shadow: var(--shadow-hover);
}

.card:active {
  transform: scale(1.01);
  box-shadow: var(--shadow-resting);
}
```

**Variants:**
- Images: use `scale(1.05)` inside `overflow: hidden` container for zoom-without-grow effect
- Avatars: `scale(1.08)` + `box-shadow: 0 0 0 3px var(--color-focus-ring)`
- Subtle: `scale(1.015)` for dense lists where large movement is distracting

---

## 2. Button Press Feedback

```css
.btn {
  transition:
    transform var(--duration-fast) var(--ease-spring),
    background-color var(--duration-fast) var(--ease-out),
    box-shadow var(--duration-fast) var(--ease-out);
  user-select: none;
}

.btn:hover {
  transform: translateY(-1px);
  box-shadow: var(--shadow-hover);
}

.btn:active {
  transform: scale(0.96) translateY(0);
  box-shadow: var(--shadow-active);
}

.btn:disabled {
  pointer-events: none;
  opacity: 0.5;
  transform: none;
}
```

**Variants:**
- Ghost/outline buttons: add `background-color` transition from transparent to semi-transparent
- Icon buttons (square): `scale(0.9)` on active for more dramatic press feel
- Destructive buttons (delete): try `scale(0.95)` + brief `background-color` flash on active

---

## 3. Focus Ring (Accessible, Keyboard-Only)

```css
/* Remove default outline, replace with custom */
.focusable {
  outline: none;
  transition:
    box-shadow var(--duration-fast) var(--ease-out),
    border-color var(--duration-fast) var(--ease-out);
}

.focusable:focus-visible {
  box-shadow:
    0 0 0 var(--focus-ring-offset) white,
    0 0 0 calc(var(--focus-ring-offset) + var(--focus-ring-width)) var(--color-focus-ring);
}

/* Dark mode: swap the white gap ring for dark bg color */
@media (prefers-color-scheme: dark) {
  .focusable:focus-visible {
    box-shadow:
      0 0 0 var(--focus-ring-offset) #1a1a1a,
      0 0 0 calc(var(--focus-ring-offset) + var(--focus-ring-width)) var(--color-focus-ring);
  }
}
```

**Note:** Never use `:focus` alone — it fires on mouse click too. Always `:focus-visible`.

---

## 4. Link Underline Sweep

```css
.link {
  text-decoration: none;
  background-image: linear-gradient(currentColor, currentColor);
  background-position: 0% 100%;
  background-repeat: no-repeat;
  background-size: 0% 2px;
  transition: background-size var(--duration-base) var(--ease-out);
}

.link:hover,
.link:focus-visible {
  background-size: 100% 2px;
}
```

**Variants:**
- Reverse sweep (right to left): use `background-position: 100% 100%` on base, `0% 100%` on hover, and animate `background-size` from `0% 2px` to `100% 2px`
- Thick underline: change `2px` to `3px` or `4px`
- Colored: replace `currentColor` with a brand color

---

## 5. Card Lift

```css
.card {
  box-shadow: var(--shadow-resting);
  transition:
    transform var(--duration-slow) var(--ease-out),
    box-shadow var(--duration-slow) var(--ease-out);
}

.card:hover {
  transform: translateY(-4px);
  box-shadow: var(--shadow-hover);
}
```

**Performance note:** If rendering many cards, consider using a pseudo-element for the shadow
and animating its `opacity` instead of animating `box-shadow` directly:

```css
.card::after {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  box-shadow: 0 8px 24px rgba(0,0,0,0.18);
  opacity: 0;
  transition: opacity var(--duration-slow) var(--ease-out);
  pointer-events: none;
}

.card:hover::after {
  opacity: 1;
}
```

---

## 6. Icon Wiggle / Shake

```css
@keyframes wiggle {
  0%, 100% { transform: rotate(0deg); }
  20%       { transform: rotate(-12deg); }
  40%       { transform: rotate(10deg); }
  60%       { transform: rotate(-8deg); }
  80%       { transform: rotate(6deg); }
}

.icon-btn:hover .icon {
  animation: wiggle 0.5s var(--ease-in-out);
}

/* Or for a subtle bob: */
@keyframes bob {
  0%, 100% { transform: translateY(0); }
  50%       { transform: translateY(-3px); }
}

.icon-btn:hover .icon {
  animation: bob 0.6s var(--ease-in-out) infinite;
}
```

---

## 7. Checkbox / Radio Spring

```css
/* Assumes a custom checkbox with a visible checkmark element */
.checkbox__checkmark {
  transform: scale(0);
  opacity: 0;
  transition:
    transform var(--duration-fast) var(--ease-spring),
    opacity var(--duration-fast) var(--ease-out);
}

input[type="checkbox"]:checked + .checkbox__checkmark {
  transform: scale(1);
  opacity: 1;
}

/* The box itself */
.checkbox__box {
  transition:
    border-color var(--duration-fast) var(--ease-out),
    background-color var(--duration-fast) var(--ease-out);
}

input[type="checkbox"]:checked + .checkbox__box {
  border-color: var(--color-focus-ring);
  background-color: var(--color-focus-ring);
}
```

---

## 8. Input Focus Expand (Glow)

```css
.input {
  border: 1.5px solid #d1d5db;
  border-radius: 6px;
  transition:
    border-color var(--duration-fast) var(--ease-out),
    box-shadow var(--duration-fast) var(--ease-out);
}

.input:hover {
  border-color: #9ca3af;
}

.input:focus {
  outline: none;
  border-color: var(--color-focus-ring);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.25); /* match --color-focus-ring with opacity */
}

.input:focus::placeholder {
  opacity: 0.5;
  transition: opacity var(--duration-base) var(--ease-out);
}
```

---

## 9. Skeleton Loader (Shimmer)

```css
@keyframes shimmer {
  0%   { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

.skeleton {
  background: linear-gradient(
    90deg,
    #f0f0f0 25%,
    #e0e0e0 50%,
    #f0f0f0 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite var(--ease-in-out);
  border-radius: 4px;
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  .skeleton {
    background: linear-gradient(
      90deg,
      #2a2a2a 25%,
      #3a3a3a 50%,
      #2a2a2a 75%
    );
    background-size: 200% 100%;
  }
}
```

---

## 10. Fade + Slide Entrance (Modals, Dropdowns, Toasts)

```css
/* Dropdown / popover */
.dropdown {
  opacity: 0;
  transform: translateY(-6px) scale(0.98);
  transition:
    opacity var(--duration-base) var(--ease-out),
    transform var(--duration-base) var(--ease-out);
  pointer-events: none;
}

.dropdown.is-open {
  opacity: 1;
  transform: translateY(0) scale(1);
  pointer-events: auto;
}

/* Toast (enters from bottom) */
@keyframes toast-enter {
  from {
    opacity: 0;
    transform: translateY(16px) scale(0.97);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

.toast {
  animation: toast-enter var(--duration-enter) var(--ease-spring) both;
}

/* Modal backdrop */
@keyframes backdrop-fade {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.modal-backdrop {
  animation: backdrop-fade var(--duration-base) var(--ease-out) both;
}
```

---

## 11. Ripple Effect (Material-style)

```css
.btn {
  position: relative;
  overflow: hidden;
}

/* JS injects a <span class="ripple"> at click position */
.ripple {
  position: absolute;
  border-radius: 50%;
  width: 100px;
  height: 100px;
  margin-top: -50px;
  margin-left: -50px;
  background: rgba(255, 255, 255, 0.35);
  animation: ripple-expand 0.6s var(--ease-out) forwards;
  pointer-events: none;
}

@keyframes ripple-expand {
  from {
    transform: scale(0);
    opacity: 1;
  }
  to {
    transform: scale(4);
    opacity: 0;
  }
}
```

```javascript
// Minimal JS to inject ripple
document.querySelectorAll('.btn').forEach(btn => {
  btn.addEventListener('click', function(e) {
    const ripple = document.createElement('span');
    ripple.className = 'ripple';
    const rect = this.getBoundingClientRect();
    ripple.style.left = (e.clientX - rect.left) + 'px';
    ripple.style.top  = (e.clientY - rect.top)  + 'px';
    this.appendChild(ripple);
    ripple.addEventListener('animationend', () => ripple.remove());
  });
});
```

---

## 12. Toggle Switch

```css
.toggle {
  position: relative;
  width: 44px;
  height: 24px;
  background: #d1d5db;
  border-radius: 9999px;
  transition: background-color var(--duration-base) var(--ease-out);
  cursor: pointer;
}

.toggle.is-on {
  background: var(--color-focus-ring);
}

.toggle__thumb {
  position: absolute;
  top: 3px;
  left: 3px;
  width: 18px;
  height: 18px;
  background: white;
  border-radius: 50%;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
  transition: transform var(--duration-base) var(--ease-spring);
}

.toggle.is-on .toggle__thumb {
  transform: translateX(20px);
}
```

---

## 13. Progress Bar Fill

```css
.progress-bar {
  height: 6px;
  background: #e5e7eb;
  border-radius: 9999px;
  overflow: hidden;
}

.progress-bar__fill {
  height: 100%;
  border-radius: 9999px;
  background: linear-gradient(90deg, #3b82f6, #6366f1);
  transition: width var(--duration-slow) var(--ease-out);
  /* Set width via JS: element.style.width = percentage + '%' */
}
```

---

## 14. Tooltip Appear

```css
.tooltip-wrapper {
  position: relative;
}

.tooltip {
  position: absolute;
  bottom: calc(100% + 6px);
  left: 50%;
  transform: translateX(-50%) scale(0.92);
  opacity: 0;
  pointer-events: none;
  transition:
    opacity var(--duration-fast) var(--ease-out),
    transform var(--duration-fast) var(--ease-spring);
  transform-origin: bottom center;
  white-space: nowrap;
}

.tooltip-wrapper:hover .tooltip,
.tooltip-wrapper:focus-within .tooltip {
  opacity: 1;
  transform: translateX(-50%) scale(1);
}
```

---

## Tailwind Equivalents Cheat Sheet

| CSS Pattern         | Tailwind Classes                                               |
|---------------------|----------------------------------------------------------------|
| Hover scale card    | `transition-transform duration-200 hover:scale-[1.03]`        |
| Button press        | `active:scale-95 transition-transform duration-100`            |
| Focus ring          | `focus-visible:ring-2 focus-visible:ring-blue-500 focus-visible:ring-offset-2 outline-none` |
| Card lift           | `hover:-translate-y-1 hover:shadow-lg transition-all duration-200` |
| Input glow          | `focus:ring-2 focus:ring-blue-500/25 focus:border-blue-500 transition-shadow duration-150` |
| Fade entrance       | `animate-in fade-in slide-in-from-top-2 duration-200` (requires `tailwindcss-animate`) |