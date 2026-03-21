---
name: polymorphic-component
description: Build polymorphic React components using an `as` prop with full TypeScript support for prop inference, ref forwarding, and prop conflict resolution. Use when typing components that render different elements, handling refs, or fixing TS issues with polymorphic patterns.
category: "React"
---

# polymorphic-component

Implements the `as` prop pattern with full TypeScript safety — correct HTML prop inference per element, working ref types, and no prop leakage. Skips basics; addresses only the parts that cause real pain.

---

## Phase 1 — Discover

Establish before writing anything:

- Does the component need **ref forwarding**? (If yes, the generic chain is different — see Phase 3.)
- Does the component have **own props that may conflict** with native HTML props? (e.g., a `size` prop clashing with `<input size>`)
- What's the **default element** when `as` is omitted?
- Should `as` accept **custom components**, or only HTML element strings? (Accepting both requires a union constraint.)

---

## Phase 2 — The Core Type Problem (and why naive solutions fail)

### What people try first (and why it breaks)

```ts
// ❌ Loses all HTML prop inference — props is just {}
type Props = { as?: React.ElementType }

// ❌ Hardcoded — doesn't change with `as`
type Props = { as?: keyof JSX.IntrinsicElements } & React.HTMLAttributes<HTMLElement>

// ❌ Gives HTMLElement props regardless of which element — `href` appears on a button
type Props<T extends React.ElementType> = { as?: T } & React.HTMLAttributes<HTMLElement>
```

### The correct foundation

The key is `React.ComponentPropsWithoutRef<T>` — it resolves to the exact prop set for whatever `T` is.

```ts
type AsProp<T extends React.ElementType> = { as?: T }

type PropsToOmit<T extends React.ElementType, OwnProps> = keyof (AsProp<T> & OwnProps)

type PolymorphicComponentProps<
  T extends React.ElementType,
  OwnProps = {}
> = AsProp<T> &
  OwnProps &
  Omit<React.ComponentPropsWithoutRef<T>, PropsToOmit<T, OwnProps>>
```

`PropsToOmit` strips from the native props anything that `OwnProps` already declares — this resolves the conflict problem. Without it, TypeScript errors on conflicting prop names and the resolution is non-deterministic.

---

## Phase 3 — Full Implementation Patterns

### Without ref forwarding (simpler)

```ts
type TextOwnProps = {
  size?: 'sm' | 'md' | 'lg'
  variant?: 'body' | 'heading'
}

type TextProps<T extends React.ElementType = 'span'> = PolymorphicComponentProps<T, TextOwnProps>

// The function signature needs a separate generic — can't inline it
function Text<T extends React.ElementType = 'span'>({
  as,
  size = 'md',
  variant = 'body',
  ...rest
}: TextProps<T>) {
  const Component = as ?? 'span'
  return <Component {...rest} />
}
```

Usage:
```tsx
<Text as="a" href="/home">Link</Text>       // href is valid
<Text as="button" type="submit">Click</Text> // type is valid
<Text href="/home">Link</Text>              // TS error — span has no href ✓
```

### With ref forwarding (the hard part)

`React.forwardRef` doesn't support free generics on its own — the generic is fixed at call time. You need a cast.

```ts
type PolymorphicRef<T extends React.ElementType> = React.ComponentPropsWithRef<T>['ref']

type PolymorphicComponentPropsWithRef<
  T extends React.ElementType,
  OwnProps = {}
> = PolymorphicComponentProps<T, OwnProps> & { ref?: PolymorphicRef<T> }

// The cast is unavoidable — forwardRef's type system can't express free generics
const Text = React.forwardRef(function Text<T extends React.ElementType = 'span'>(
  { as, size, variant, ...rest }: PolymorphicComponentPropsWithRef<T, TextOwnProps>,
  ref: PolymorphicRef<T>
) {
  const Component = (as ?? 'span') as React.ElementType
  return <Component ref={ref} {...rest} />
}) as <T extends React.ElementType = 'span'>(
  props: PolymorphicComponentPropsWithRef<T, TextOwnProps>
) => React.ReactElement | null
```

**Why the cast at the end?** `forwardRef` returns `ForwardRefExoticComponent` with a fixed signature — it can't carry a free generic `T`. The cast restores the generic to the public API without losing the implementation.

**Why cast `as` inside the render?** TypeScript narrows `Component` to `T` but can't confirm it's assignable to `JSX.IntrinsicElements` or a component type without the cast. This is a known TS limitation with conditional types on generics.

---

## Phase 4 — Output

Produce a complete, copy-paste-ready implementation. Always include:

1. The **shared utility types** (`PolymorphicComponentProps`, `PolymorphicRef`, etc.) as a standalone block — these belong in a `types/polymorphic.ts` or similar; do not inline them per-component.
2. The **component implementation** — with or without `forwardRef` based on Phase 1 discovery.
3. **Usage examples** covering at least: a valid `as` swap with element-specific props, and an example that correctly TS-errors when used wrong.

---

## Non-obvious Rules to Enforce

**`React.ElementType` vs `keyof JSX.IntrinsicElements`**
`React.ElementType` = HTML element strings + React components. Use this for maximum flexibility. `keyof JSX.IntrinsicElements` only covers HTML tags — custom component support requires `React.ElementType`.

**Default element must appear in TWO places**
In the generic default (`T extends React.ElementType = 'span'`) AND in the fallback (`as ?? 'span'`). Mismatching these causes correct TS types but wrong runtime behavior, or vice versa.

**Don't spread `as` onto the DOM element**
`rest` must NOT contain `as` — it's not a valid HTML attribute and will throw a DOM warning. Always destructure it out explicitly, even if unused.

**`children` is usually fine but check slots**
`React.ComponentPropsWithoutRef<T>` includes `children` from `React.PropsWithChildren` for most elements. If `OwnProps` declares its own `children` with a different type (e.g., a render prop), add `children` to `PropsToOmit`.

**Styled-components / Emotion `as` prop**
These libraries implement their own `as` prop internally. If the polymorphic component is itself a styled component, there will be a double-`as` conflict. Rename the prop to `renderAs` and document why.

**`displayName` is lost after the cast**
After the `forwardRef` cast, `Component.displayName` won't infer from the function name. Set it explicitly: `Text.displayName = 'Text'`.