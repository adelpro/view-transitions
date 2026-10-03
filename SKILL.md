---
name: view-transitions
description: Build view transitions and shared-element animations on the web. Use when adding a page transition, a shared element morph, a card growing into a detail page, a circular theme wipe, a modal that grows from the button that opened it, list rows that travel to new positions, or cross-document navigation animation, in plain CSS, React with the ViewTransition component, or Next.js. Also use when someone mentions view-transition-name, startViewTransition, ViewTransition, navigation auto, or a transition that does not animate.
version: 1.0.0
author: Adel Ben Yahia (adelpro)
license: MIT
platforms: [linux, macos, windows]
metadata:
  author: Adel Ben Yahia (adelpro)
  version: 1.0.0
  hermes:
    tags: [skills, animation, view-transitions, react, nextjs, css]
    related_skills: [animate, animate-expo]
---
A construction skill for the web. It turns a request for a transition between two states
into a view transition that would survive review, and refuses the requests the platform
cannot answer.

## Operating posture

You are a senior design engineer. Make the call, state the reasoning in one line, write the
code. Never present the options as a menu.

The honest answer to "add a transition here" is often **no transition**. Route changes
already carry a visual signal; adding motion on top of it is decoration, not communication.
A transition earns its place when it answers one question the user is actually asking:
*where did that come from*, *where did that go*, or *did anything change*.

## The mechanism

A view transition is a **screenshot morph**, not a property animation. The browser
snapshots the element before and after the change, then interpolates position, size,
border-radius, and the pixels. You never measure anything.

Three consequences follow, and they are the whole skill:

1. **One name per identity, app-wide.** Two elements holding the same
   `view-transition-name` does not degrade gracefully — the transition is skipped. In plain
   CSS it is completely silent; in React it is a development-mode error.
2. **Only named elements animate.** Everything else cross-fades as one `root` snapshot. A
   header you wanted to keep still will travel with the content unless you stop it.
3. **The overlay eats pointer events** for the animation's duration.

Note what this means for the usual rules: a morph animates `width` and `height`, because it
moves a bitmap rather than re-laying out a box. `transform: scale()` cannot produce this
effect at all. If a sibling skill's property-animation rules seem to be violated here, that
is why — different mechanism, different rules.

## Pick the path

Three paths. Choose by stack before writing anything.

| Condition | Mechanism | Read |
| --- | --- | --- |
| React >= 19.3 | `<ViewTransition>` — React drives `startViewTransition` | [references/react.md](references/react.md) |
| Anything else with JS | `document.startViewTransition`, feature-detected | [references/css.md](references/css.md) |
| Multi-page site, no JS | `@view-transition { navigation: auto }` | [references/css.md](references/css.md) |

`<ViewTransition>` is stable in React 19.3 and **not available before it**. Next.js App
Router needs **no configuration** — the `experimental.viewTransition` flag in older docs
was the Next 15 gate and is obsolete.

## What are you building

Route from intent, because "card to detail page" is what people ask for.

| Building this | Technique | Demo |
| --- | --- | --- |
| Card becoming a detail page | shared element | [#shared-element](assets/demo.html#shared-element) |
| Sorting or filtering a list | per-item names | [#list-reorder](assets/demo.html#list-reorder) |
| Theme toggle | circular reveal | [#theme-wipe](assets/demo.html#theme-wipe) |
| Modal opened from a button | name handoff | [#modal-handoff](assets/demo.html#modal-handoff) |
| Multi-page site | cross-document | [trips.html](assets/trips.html) |
| Skeleton becoming content | Suspense update | [react-suspense.html](assets/react-suspense.html) |

## The three that fail silently

Three techniques look trivial and have a specific trap each. These are worth the extra care.

**A modal from its button.** The button must *relinquish* the name inside the same
transition — set it to `none`, give the dialog `share`, then open it. Skip the rename and
the modal appears from nowhere. Reclaim the name on close.

**Every row its own name.** Derive it from item identity (`view-transition-name: var(--trip)`).
Two rows sharing one name freezes both, with no error anywhere.

**A skeleton becoming content.** Fallbacks appear *without* animation; only the
fallback → content update animates. Needs `update="auto"` *and* `default="none"` on React.
This is the one case where animating more is the bug — it makes cached UI feel slow.

## Production requirements

These ship with every transition. None is applied for you.

```css
::view-transition { pointer-events: none; }

@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(*), ::view-transition-new(*), ::view-transition-group(*) {
    animation-duration: 0s !important;
    animation-delay: 0s !important;
  }
}
```

**Reduced motion is never automatic** — not by the browser, not by React. Directional
slides and circular wipes move large areas across the viewport and are the highest-risk
effects; morphs and cross-fades carry much less risk. Reduced motion means gentler, not
zero: if the morph is the only thing saying you moved somewhere, a cross-fade still carries
that signal.

**Feature-detect.** `document.startViewTransition` is `undefined` without support.
Calling the update directly *is* the correct fallback — the page works, it just does not
animate.

## Duration

250–400ms for morphs and directional slides. This is deliberately wider than the 300ms
ceiling that applies to property animations on frequently-used UI, because a page morph is
a deliberate one-off navigation rather than a hover or a toggle. For everything that is not
a morph, the sub-300ms rule still applies.

## Never ship

| Never | Instead |
| --- | --- |
| Two elements sharing one `view-transition-name` | One name per identity, from a shared constant |
| Unguarded `document.startViewTransition` | `if (typeof … === 'function')` |
| Manual `startViewTransition` beside `<ViewTransition>` | Let React drive it — you interrupt it |
| `default="none"` with no `share` | The pair silently stops morphing |
| Wrapper element inside a list row | Let the row own its boundary |
| `setState` outside a Transition, expecting animation | `startTransition`, `<Suspense>`, `useDeferredValue` |
| Per-frame `width`/`height` to fake a morph | The snapshot morph |
| `transform: scale()` to fake continuity | `view-transition-name` |
| `navigation: auto` in a SPA, or on one page only | `startViewTransition`; both documents need it |
| Naming a header and leaving it default | Kill its animation explicitly |
| `experimental.viewTransition` in `next.config` | Not needed on Next 16 |
| Shipping without `prefers-reduced-motion` | Gentler variant, not zero |
| Omitting `pointer-events: none` | The overlay eats rapid clicks |

## Output

Write the code. Then, in a few lines:

- **The path** you took and why — React, plain CSS, or cross-document.
- **The names** you assigned, and proof they are unique.
- **What to feel-check** — morph quality and wipe direction cannot be judged from code. Play
  it at 2–5x duration, check it with reduced motion on, and interrupt it mid-flight.

## Out of scope

React Native, Expo, and Android. `animate-expo` owns native transitions; the platform
transitions there are not shared-element animations and this skill does not cover them.