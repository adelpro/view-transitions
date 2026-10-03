# view-transitions

A skill for building **view transitions and shared-element animations on the web**.

A view transition is a *screenshot morph*, not a property animation — the browser snapshots
an element before and after a change and interpolates position, size, border-radius and the
pixels between them. Naming two elements the same is the entire mechanism, which is why it
is one CSS property instead of a shared-layout library.

## Try it first

Open [`assets/demo.html`](assets/demo.html) — no build, no dependencies, works offline.
Five CSS techniques in one file, each clickable:

1. **Card becomes a page** — one name, applied to the clicked card and handed to the hero slot
2. **Rows travel to a new position** — one name per row, plus a button that deliberately
   breaks it so you can see the failure yourself
3. **Theme toggle** — a `clip-path` reveal growing from the click point
4. **Modal born from its button** — the name handoff, in the order it has to happen
5. **Across a real page load** — cross-document navigation between two pages

Plus a sixth on real React 19.3:
[`assets/react-suspense.html`](assets/react-suspense.html) — skeleton → content, including
the case where animating *more* is the bug.

## Install

```bash
npx skills add adelpro/view-transitions
```

## What it covers

| Stack | Mechanism |
| --- | --- |
| Plain CSS | `view-transition-name`, `document.startViewTransition`, `@view-transition` |
| React >= 19.3 | `<ViewTransition>`, `addTransitionType` |
| Next.js | The above — the App Router needs no configuration |

Each stack has a reference file that the skill loads on demand:
[`references/css.md`](references/css.md),
[`references/react.md`](references/react.md).

## The parts people get wrong

This skill exists mostly for these, because each fails quietly rather than loudly:

- **Duplicate `view-transition-name`.** Two elements sharing a name does not degrade — the
  transition is skipped. In plain CSS there is no error at all.
- **`prefers-reduced-motion` is never applied automatically.** Not by the browser, not by
  React. The skill ships the media query with the animation.
- **The `::view-transition` overlay eats pointer events** for the animation's duration, so
  a fast second click disappears without `pointer-events: none`.
- **`default="none"` without `share`.** The React pair silently stops morphing.
- **`document.startViewTransition` is `undefined`** without support. Calling it unguarded
  throws and takes the click handler down.

## Not covered

React Native, Expo, and Android. Native transitions are a different mechanism with a
different platform API, and the platform transitions there are not shared-element
animations.

## Installability

Discoverable by `npx skills add`, by the Codex CLI, and by Claude Code. Validated with
[`skill-validator-omni`](https://github.com/adelpro/skill-validator-omni).

## License

MIT