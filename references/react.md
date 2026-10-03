# View transitions in React

`<ViewTransition>` is **stable as of React 19.3** (released 2026-09-09), along with
`addTransitionType`. Both are plain named exports — no `unstable_` prefix:

```jsx
import { ViewTransition, addTransitionType, startTransition, useState } from 'react';
```

Check the version in the project before writing this. On React 19.2 and earlier, use the
plain CSS path in [css.md](css.md) instead — the API is not available.

## Contents

- [How React differs from plain CSS](#how-react-differs-from-plain-css)
- [The four triggers](#the-four-triggers)
- [Shared elements](#shared-elements)
- [Suspense reveals](#suspense-reveals)
- [Directional navigation](#directional-navigation)
- [List reordering](#list-reordering)
- [Next.js](#nextjs)
- [What not to do](#what-not-to-do)

## How React differs from plain CSS

Everything in [css.md](css.md) about the mechanism still holds: a view transition is a
screenshot morph, one name per identity, only named elements animate. Three things change:

1. **React calls `startViewTransition` for you.** You never call it. Calling it yourself
   while React is managing a transition interrupts React's.
2. **You must wrap the update in a Transition.** A bare `setState` does not animate.
3. **You style with View Transition Classes, not `::view-transition-name()`.** React
   assigns the names itself.

## The four triggers

React picks the animation from how the tree changed:

| Trigger | Cause |
| --- | --- |
| `enter` | The boundary was added |
| `exit` | The boundary was removed |
| `update` | Its contents or size changed |
| `share` | A named boundary was removed in one place and added in another |

Style each independently with the `enter` / `exit` / `update` / `share` props, and use
`default` to turn everything off:

```jsx
<ViewTransition update="auto" default="none">
  …
</ViewTransition>
```

`default="none"` is the important one. Without it, **every named `<ViewTransition>` on the
page animates whenever any transition runs** — so a theme toggle makes every image on the
screen cross-fade.

Values can be `"auto"`, `"none"`, a class name, or an object keyed by transition type.

## Shared elements

Give both sides the same `name` and React morphs one into the other:

```jsx
// grid
<ViewTransition name={`photo-${photo.id}`} share="morph" default="none">
  <Image src={photo.src} alt={photo.title} />
</ViewTransition>

// detail — same name
<ViewTransition name={`photo-${photo.id}`} share="morph" default="none">
  <Image src={photo.src} alt={photo.title} />
</ViewTransition>
```

Three rules that make this fail:

- **Names must be unique app-wide at any instant.** Two mounted elements with one name is
  a development-mode error in React, where plain CSS fails silently — same bug, different
  symptom.
- **`default="none"` requires an explicit `share`.** With `default="none"` and no `share`,
  the pair silently stops morphing. This is the quietest failure in the whole API.
- **The pair must mount in the same commit.** If the destination suspends into a fallback
  first, no pair forms.

Compare [css.md — shared elements](css.md#naming-two-elements), and see it running in
[`../assets/demo.html#shared-element`](../assets/demo.html#shared-element).

## Suspense reveals

This is the one technique where animating *more* is the bug. React's own guidance:

- Fallbacks appear **without** animation.
- The fallback → content update **does** animate.
- Cached content appears instantly, with no animation.

```jsx
<ViewTransition update="auto" default="none">
  <Suspense fallback={<Skeleton />}>
    <Content />
  </Suspense>
</ViewTransition>
```

Without `default="none"` the skeleton animates in, which is exactly what you don't want —
it makes an already-cached UI feel slow.

There are two placements and they mean different things:

```
<ViewTransition>              → treated as one "update": same name, so it cross-fades
  <Suspense fallback={<A />}>
    <B />
  </Suspense>
</ViewTransition>
```

```
<Suspense fallback={<ViewTransition><A /></ViewTransition>}>   → separate enter and exit
  <ViewTransition><B /></ViewTransition>
</Suspense>
```

Use the first unless you specifically want a hand-off in which the placeholder yields to
the real thing.

Working example, including all three behaviours above:
[`../assets/react-suspense.html`](../assets/react-suspense.html).

## Directional navigation

Tag the *cause* of a transition, then style by tag. This is how a carousel going forward
and backward animates in opposite directions even though both set the same state.

```jsx
function nextSlide() {
  startTransition(() => {
    addTransitionType('next');
    setCurrentSlide((c) => (c + 1) % count);
  });
}

<ViewTransition
  enter={{ next: 'from-right', previous: 'from-left' }}
  exit={{ next: 'to-left', previous: 'to-right' }}
>
  <Page />
</ViewTransition>
```

```css
::view-transition-new(.from-right) { --offset: 100%; animation-name: slide-in; }
::view-transition-new(.from-left)  { --offset: -100%; animation-name: slide-in; }
::view-transition-old(.to-left)    { --offset: 100%; animation-name: slide-out; }
@keyframes slide-in  { from { transform: translateX(var(--offset)); opacity: 0; } }
@keyframes slide-out { to   { transform: translateX(var(--offset)); opacity: 0; } }
```

Directional slides are the highest-risk effect for motion sensitivity — they move large
areas across the viewport — so they are the first thing to disable under
`prefers-reduced-motion`.

## List reordering

Wrap each row in its own `<ViewTransition>` and let the key carry identity:

```jsx
{items.map((item) => (
  <ViewTransition key={item.id}>
    <Row item={item} />
  </ViewTransition>
))}
```

**Do not add a wrapper element** around the row inside the list. With a wrapper, React
animates the parent instead and you get one cross-fade rather than rows travelling. This
is the same "only named elements animate" rule as CSS, in a different disguise.

Keep `key` stable and correct. Do not reach for `name` to force a reorder: `name` forms a
shared-element pair, and it will not fire if one side is outside the viewport.

## Next.js

**No configuration is required.** The App Router already runs on a React build that
includes `<ViewTransition>`, and route navigations are Transitions, so animations activate
during navigation automatically.

The `experimental.viewTransition: true` flag you may find in older docs was the Next 15
gate. It is obsolete — do not add it.

Tag links with `transitionTypes` so the animation reflects direction:

```tsx
<Link href={`/photo/${id}`} transitionTypes={['nav-forward']}>Open</Link>
<Link href="/" transitionTypes={['nav-back']}>← Gallery</Link>
```

```tsx
<ViewTransition
  enter={{ 'nav-forward': 'nav-forward', 'nav-back': 'nav-back', default: 'none' }}
  exit={{ 'nav-forward': 'nav-forward', 'nav-back': 'nav-back', default: 'none' }}
  default="none"
>
  <Page />
</ViewTransition>
```

Put the wrapper in each `page.tsx`, **not** in a layout — layouts persist across
navigation, so enter and exit never fire there.

Browser-initiated back/forward carries no transition type, so no directional slide plays.
The shared-element morph still applies if both pages share a name.

## What not to do

| Never | Instead |
| --- | --- |
| Calling `startViewTransition` yourself | React calls it; you interrupt it |
| `setState` outside a Transition, expecting animation | `startTransition`, `<Suspense>`, or `useDeferredValue` |
| Two mounted boundaries sharing one `name` | One name per identity, from a shared constant |
| `default="none"` with no `share` | The pair silently stops morphing |
| A wrapper element around each list row | Let the row own its boundary |
| `<ViewTransition>` not first in its subtree | Move it above the DOM node |
| `name` used to force a list reorder | Use a stable `key` |
| `experimental.viewTransition` in `next.config` | Not needed on Next 16 |
| Assuming reduced motion is automatic | It never is — ship the media query |
| Omitting `pointer-events: none` | The overlay eats rapid clicks |