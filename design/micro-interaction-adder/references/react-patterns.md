# React Micro-Interaction Patterns

Supplement to `pattern-library.md` — React-specific techniques.

---

## 1. Class-Toggle Entrance (No Library)

The simplest approach: add a CSS class after mount via `useEffect`.

```jsx
import { useState, useEffect } from 'react';

function FadeIn({ children, delay = 0 }) {
  const [visible, setVisible] = useState(false);

  useEffect(() => {
    const id = setTimeout(() => setVisible(true), delay);
    return () => clearTimeout(id);
  }, [delay]);

  return (
    <div
      style={{
        opacity:    visible ? 1 : 0,
        transform:  visible ? 'translateY(0)' : 'translateY(12px)',
        transition: `opacity 300ms ease-out ${delay}ms, transform 300ms ease-out ${delay}ms`,
      }}
    >
      {children}
    </div>
  );
}
```

---

## 2. Framer Motion — Core Patterns

Install: `npm install framer-motion`

### Button with spring press

```jsx
import { motion } from 'framer-motion';

<motion.button
  whileHover={{ scale: 1.04, y: -2 }}
  whileTap={{ scale: 0.96 }}
  transition={{ type: 'spring', stiffness: 400, damping: 17 }}
>
  Click me
</motion.button>
```

### Card lift

```jsx
<motion.div
  whileHover={{ y: -4, boxShadow: '0 8px 24px rgba(0,0,0,0.15)' }}
  transition={{ duration: 0.2, ease: [0.25, 0.46, 0.45, 0.94] }}
>
  Card content
</motion.div>
```

### Staggered list entrance

```jsx
const container = {
  hidden: {},
  show: {
    transition: {
      staggerChildren: 0.07,
      delayChildren: 0.1,
    }
  }
};

const item = {
  hidden: { opacity: 0, y: 16 },
  show:   { opacity: 1, y: 0, transition: { duration: 0.3, ease: 'easeOut' } }
};

<motion.ul variants={container} initial="hidden" animate="show">
  {items.map(i => (
    <motion.li key={i.id} variants={item}>{i.label}</motion.li>
  ))}
</motion.ul>
```

### Modal / dialog entrance

```jsx
<AnimatePresence>
  {isOpen && (
    <motion.div
      key="modal"
      initial={{ opacity: 0, scale: 0.96, y: 12 }}
      animate={{ opacity: 1, scale: 1, y: 0 }}
      exit={{ opacity: 0, scale: 0.96, y: 8 }}
      transition={{ duration: 0.2, ease: [0.34, 1.56, 0.64, 1] }}
    >
      {children}
    </motion.div>
  )}
</AnimatePresence>
```

---

## 3. CSS Modules Approach (No JS Library)

Keep animations purely in CSS, toggle class with `useState`:

```css
/* Card.module.css */
.card {
  transition: transform 200ms cubic-bezier(0.25, 0.46, 0.45, 0.94),
              box-shadow 200ms ease-out;
}
.card:hover  { transform: translateY(-4px); box-shadow: 0 8px 24px rgba(0,0,0,0.14); }
.card:active { transform: translateY(-1px); box-shadow: 0 2px 6px rgba(0,0,0,0.10); }
```

---

## 4. `useReducedMotion` Hook

Always respect the user's system preference:

```jsx
import { useReducedMotion } from 'framer-motion';

function AnimatedCard({ children }) {
  const reduce = useReducedMotion();

  return (
    <motion.div
      whileHover={reduce ? {} : { scale: 1.03, y: -3 }}
      whileTap={reduce ? {} : { scale: 0.97 }}
    >
      {children}
    </motion.div>
  );
}
```

Or for pure CSS approach, wrap all keyframe/transform CSS in:

```css
@media (prefers-reduced-motion: no-preference) {
  /* motion-heavy CSS here */
}
```

---

## 5. Button Ripple in React

```jsx
import { useRef } from 'react';

function RippleButton({ children, ...props }) {
  const btnRef = useRef(null);

  const handleClick = (e) => {
    const btn  = btnRef.current;
    const rect = btn.getBoundingClientRect();
    const ripple = document.createElement('span');
    ripple.className = 'ripple';
    ripple.style.left = (e.clientX - rect.left) + 'px';
    ripple.style.top  = (e.clientY - rect.top)  + 'px';
    btn.appendChild(ripple);
    ripple.addEventListener('animationend', () => ripple.remove());
    props.onClick?.(e);
  };

  return (
    <button ref={btnRef} onClick={handleClick} {...props}
      style={{ position: 'relative', overflow: 'hidden' }}>
      {children}
    </button>
  );
}
```

---

## 6. Skeleton Loader Component

```jsx
function Skeleton({ width = '100%', height = 20, borderRadius = 4 }) {
  return (
    <div
      style={{
        width, height, borderRadius,
        background: 'linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%)',
        backgroundSize: '200% 100%',
        animation: 'shimmer 1.5s infinite ease-in-out',
      }}
    />
  );
}
// Add @keyframes shimmer to global CSS (see pattern-library.md #9)
```

---

## Tailwind + React: `cn()` Pattern for Conditional Transitions

Using `clsx` or `cn`:

```jsx
import { cn } from '@/lib/utils'; // or clsx

<button
  className={cn(
    'transition-all duration-150',
    'hover:scale-[1.03] hover:-translate-y-0.5',
    'active:scale-[0.97]',
    'focus-visible:ring-2 focus-visible:ring-blue-500 focus-visible:ring-offset-2',
    isDisabled && 'opacity-50 pointer-events-none'
  )}
>
```