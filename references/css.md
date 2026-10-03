# View transitions in plain CSS

The framework-free path. No React, no build step, no dependencies — just a CSS property
and one DOM API. If you are not on React, this page is the whole answer.

React users: read [react.md](react.md) instead. The mechanism below is the same; only the
coordination differs.

## Contents

- [The one idea](#the-one-idea)
- [Naming two elements](#naming-two-elements)
- [Running the transition](#running-the-transition)
- [The pseudo-element tree](#the-pseudo-element-tree)
- [The unique-name rule](#the-unique-name-rule)
- [Modal from its button](#modal-from-its-button)
- [Cross-document navigation](#cross-document-navigation)
- [Production requirements](#production-requirements)
- [What not to do](#what-not-to-do)

## The one idea

A view transition is a **screenshot morph**, not a property animation.

The browser takes a picture of the element before the change and a picture of it after,
then interpolates position, size, border-radius, and the pixels between them. You never
measure anything. That is why it is one property rather than a layout library.

This is also why the usual animation rules do not apply. A morph animates `width` and
`height` — the properties that force layout — because it is moving a bitmap, not a box
in the document. `transform: translate()` and `scale()` tricks cannot produce this effect
at all, because the element's real geometry changes.

## Naming two elements

Put the same `view-transition-name` on both sides of the change. That is the entire
mechanism.

```css
.card img  { view-transition-name: hero; }
.page header { view-transition-name: hero; }
```

When the class on the element changes from `card` to `page`, the browser sees `hero`
before and `hero` after, and morphs one into the other — width, position, and corner
radius included.

Working example: [`../assets/demo.html#shared-element`](../assets/demo.html#shared-element).

## Running the transition

Renaming an element is not enough on its own; something has to tell the browser that a
change is happening and to snapshot around it.

```js
const supported = typeof document.startViewTransition === 'function';
const run = (fn) => (supported ? document.startViewTransition(fn) : fn());
```

Two things about that one-liner:

- **`fn()` alone is the correct fallback.** Without support, the page still works — it
  just changes instantly. This is why you feature-detect rather than assume.
- **Never call `startViewTransition` unguarded.** It is `undefined` in a browser without
  support, so the call throws and takes your click handler down with it.

Note that `startViewTransition` returns an object and the *browser* calls your callback —
it does not run synchronously. If you assert on the DOM immediately after calling it, you
will read the pre-transition state. Await `ready` if you need the animation phase:

```js
const t = document.startViewTransition(update);
await t.ready;    // pseudo-elements exist, animation about to start
await t.finished; // animation done, new state interactive
```

## The pseudo-element tree

While the transition runs, the browser paints a live snapshot into a generated element
tree you cannot reach with normal selectors. Reach it through `::view-transition-*`
pseudo-elements instead:

| Pseudo-element | What it is |
| --- | --- |
| `::view-transition` | The root of the tree. Targets it to control the overlay, e.g. `pointer-events`. |
| `::view-transition-group(name)` | The element's box. Animates position, size, and radius. |
| `::view-transition-image-pair(name)` | Wraps old and new so they can be stacked and cross-faded. |
| `::view-transition-old(name)` | The "before" snapshot. |
| `::view-transition-new(name)` | The "after" snapshot. |

A name goes inside the parentheses: `::view-transition-group(hero)`.

The unnamed `::view-transition-old(root)` / `::view-transition-new(root)` pair is
special — `root` is the entire page. Everything you did not name animates through it as a
single cross-fade, which is why a header you wanted to keep still will slide along with
the content unless you stop it (see below).

### Keeping an element still

```css
::view-transition-group(site-header) { animation: none; z-index: 100; }
::view-transition-old(site-header)  { display: none; }
::view-transition-new(site-header)  { animation: none; }
```

`display: none` on the old snapshot prevents a flash where both headers are briefly
visible.

### A reveal from a point

The theme-wipe technique is a `clip-path` animation on the root's new snapshot, with the
click position fed in as two custom properties:

```css
::view-transition-old(root), ::view-transition-new(root) {
  animation: none; mix-blend-mode: normal;
}
::view-transition-new(root) { animation: reveal 520ms cubic-bezier(0.77, 0, 0.175, 1); }

@keyframes reveal {
  from { clip-path: circle(0 at var(--x, 50%) var(--y, 50%)); }
  to   { clip-path: circle(150% at var(--x, 50%) var(--y, 50%)); }
}
```

```js
btn.addEventListener('click', (e) => {
  document.documentElement.style.setProperty('--x', `${e.clientX}px`);
  document.documentElement.style.setProperty('--y', `${e.clientY}px`);
  run(() => { document.documentElement.dataset.theme = next; });
});
```

The `0` radius means the new state is invisible at the start, so growing the circle
reveals it outward from the click. `clip-path` is the sanctioned exception to the usual
"transform and opacity only" rule.

Working example: [`../assets/demo.html#theme-wipe`](../assets/demo.html#theme-wipe).

## The unique-name rule

**Only one element can hold a given `view-transition-name` at a time.** This is the single
most common reason a transition "does not work", and it fails quietly.

Give two visible elements the same name and the browser skips the transition. In plain
CSS there is no error, no console message — the elements just refuse to animate, and the
page still behaves correctly, so the cause is easy to miss.

For a list, derive the name from the item's identity so each row owns one:

```html
<ul>
  <li style="--trip: trip-1">…</li>
  <li style="--trip: trip-2">…</li>
</ul>
```

```css
li { view-transition-name: var(--trip); }
```

Working example — including a button that deliberately breaks it, so you can see the
failure: [`../assets/demo.html#list-reorder`](../assets/demo.html#list-reorder).

If you maintain names in JavaScript, keep them in one module and import from there. A
name collision found at runtime is a much worse bug than a duplicate constant.

## Modal from its button

A dialog that appears "from nowhere" is missing a handoff. The button must **give up**
its name inside the same transition, or both elements hold the name and the dialog has
nothing to grow from.

```html
<button class="share">Share</button>
<dialog id="sheet">…</dialog>
```

```css
.share { view-transition-name: share; }
```

```js
sheet.addEventListener('close', () => run(() => {
  sheet.style.viewTransitionName = 'none';
  btn.style.viewTransitionName = 'share';   // reclaim it on the way back
}));

btn.addEventListener('click', () => run(() => {
  btn.style.viewTransitionName = 'none';    // relinquish…
  sheet.style.viewTransitionName = 'share'; // …hand off
  sheet.showModal();
}));
```

Order matters: relinquish first, then assign, then open. Get it backwards and the dialog
materialises with no visual connection to the button.

Working example: [`../assets/demo.html#modal-handoff`](../assets/demo.html#modal-handoff).

## Cross-document navigation

For a multi-page site, opt in with one at-rule per document:

```css
@view-transition { navigation: auto; }
```

**It must appear in both the page you are leaving and the page you are arriving at.** It
is a per-document opt-in, not a global setting — a rule on one side alone does nothing,
which is the usual reason this appears broken.

This is multi-page only. It does nothing in a single-page app, where you need
`startViewTransition` instead.

Working example: [`../assets/trips.html`](../assets/trips.html) and
[`../assets/lisbon.html`](../assets/lisbon.html).

## Production requirements

Both of these ship with every transition. Neither is optional, and neither is applied for
you.

### Let clicks through the overlay

The `::view-transition` overlay captures pointer events for the whole duration of the
animation. Without this, a user clicking twice quickly has the second click swallowed.

```css
::view-transition { pointer-events: none; }
```

### Honour reduced motion

Nothing applies this automatically — not the browser, not React. Directional slides and
circular wipes are the highest-risk effects here, because they move large areas across
the viewport; morphs and cross-fades carry much less risk.

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(*),
  ::view-transition-new(*),
  ::view-transition-group(*) {
    animation-duration: 0s !important;
    animation-delay: 0s !important;
  }
}
```

Reduced motion means gentler, not zero. If a morph is the only thing communicating that
you moved somewhere, a plain cross-fade still carries that signal; wiping the entire
viewport does not need to.

## What not to do

| Never | Instead |
| --- | --- |
| Two elements sharing one `view-transition-name` | One name per identity, from a shared constant |
| `startViewTransition` called unguarded | `if (typeof document.startViewTransition === 'function')` |
| Animating `width`/`height` per frame to fake a morph | The snapshot morph — one property |
| `transform: scale()` on both sides to fake continuity | `view-transition-name` |
| `@view-transition` in a SPA | `startViewTransition` |
| `navigation: auto` on only one page | Both documents need it |
| Naming a header and leaving it default | Kill its animation explicitly |
| Shipping without `prefers-reduced-motion` | Gentler variant, not zero |
| Omitting `pointer-events: none` | The overlay eats rapid clicks |