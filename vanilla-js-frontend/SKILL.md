---
name: vanilla-js-frontend
description: Enforces strict vanilla web technology standards (HTML, CSS, JS) and explicitly prohibits frontend frameworks and CSS libraries. Use this skill whenever the user asks to build, extend, or debug a UI, page, component, layout, or client-side interaction — even if they don't say "vanilla," "HTML," or "JavaScript" explicitly (e.g. "add a modal", "make this responsive", "wire up this form", "the dropdown isn't working"). Also applies when scaffolding a new frontend project or reviewing existing frontend code for structure, accessibility, or performance issues.
---

# Frontend Architecture Standards

You are an expert UI developer. When assisting with building user interfaces, strictly adhere to the following technological boundaries.

## 1. Architectural Philosophy

- Apply the principle: "Always is cheap and sometimes is expensive." Avoid over-engineering for edge cases that rarely occur — don't build a generic component system for one component.
- Prioritize the platform. Browsers now natively cover most of what frameworks used to be needed for (templating via template literals, reactivity via minimal pub/sub, encapsulation via Web Components) — reach for those before reaching for a library.
- Keep the client-side lightweight: no build step required to run the app locally. If a project already has a bundler (esbuild/Vite in library-free mode) for bundling ES modules, that's acceptable — it is not a loophole for adding a framework.

## 2. Tech Stack

- **Environment:** Native Browser APIs.
- **Core:** Plain vanilla JavaScript (ES6+), standard HTML5, and pure CSS (Grid/Flexbox, custom properties).
- **STRICT PROHIBITION:** Do NOT use frontend frameworks or libraries such as React, Vue, Angular, Svelte, or Solid, and do NOT use CSS frameworks such as Tailwind, Bootstrap, or Bulma. Do not add them "just for one utility" — no exceptions unless the user explicitly overrides this in the conversation.
- Native `<template>`, `customElements` (Web Components), and `<dialog>` are part of the platform, not a framework — use them where they reduce boilerplate (e.g. a repeated card component, a modal).
- Rely on native DOM APIs (`document.querySelector`, `addEventListener`, `fetch`) rather than any DOM-manipulation helper library (no jQuery).

## 3. Project Structure

Default to this layout for a new frontend project; match an existing project's layout if one already exists:

```
index.html
css/
  tokens.css        # design tokens as CSS custom properties (colors, spacing, type)
  base.css          # resets, element defaults
  layout.css         # shared layout primitives (containers, grid helpers)
  components/         # one file per component, named after it
js/
  main.js            # entry point — wires up the page
  components/         # one module per component (factory fn or custom element)
  state/              # small state/store modules, if the page needs shared state
  utils/              # DOM helpers, formatters, fetch wrappers
assets/               # images, icons, fonts
```

Use ES modules (`<script type="module">`, `import`/`export`) to split code — never one giant `main.js`.

## 4. Design Tokens

- If a `design.md` or `tokens.css` exists in the project, treat its CSS custom properties as the single source of truth for color, spacing, radius, and type — never hardcode a hex value or pixel size that already has a token.
- If no token file exists yet and the user is building out a real product (not a one-off snippet), propose one (`css/tokens.css`) rather than scattering literals across component files.

## 5. Component Pattern

Without a framework, a "component" is one of two things — pick based on reuse needs:

- **Factory function** (default, for most components): a function that builds and returns a DOM subtree, takes props as arguments, and returns element(s) plus any handles the caller needs.
  ```js
  // js/components/card.js
  export function createCard({ title, body, onClick }) {
    const el = document.createElement("article");
    el.className = "card";
    el.innerHTML = `<h3>${title}</h3><p>${body}</p>`;
    if (onClick) el.addEventListener("click", onClick);
    return el;
  }
  ```
- **Web Component** (`customElements.define`): use when a component needs true encapsulation (Shadow DOM for style isolation) or will be reused across multiple pages/projects as a drop-in tag.

Always escape/avoid injecting raw user input via `innerHTML`; use `textContent` or sanitize first.

## 6. Styling Conventions

- Use CSS custom properties (from `tokens.css`) for anything that repeats — color, spacing, radius, font size. A literal value is a signal the token file is incomplete, not a reason to inline it.
- Name classes with a consistent convention (BEM — `.card`, `.card__title`, `.card--featured`) so specificity stays flat and predictable. Avoid ID selectors and `!important`.
- Mobile-first: write base styles for small screens, layer `min-width` media queries for larger ones.
- Co-locate a component's styles in `css/components/<name>.css`, imported once via `<link>` or a build-time CSS import — not inline `style=""` attributes except for values computed at runtime (e.g. a dynamic width).

## 7. State & Data

- For local component state, plain closures/variables inside the factory function are enough — don't add a store for state one component owns.
- For state shared across components, use a small explicit store: a plain object plus a subscribe/publish function (10-20 lines), not a hand-rolled framework. Re-render only the DOM nodes that depend on changed state, not the whole page.
- Fetch data with `fetch` + `async/await`; every fetch call handles both the error case (network failure, non-2xx status) and a loading state in the UI — never a bare `fetch().then()` with no `.catch`.

## 8. Accessibility

- Use semantic HTML elements (`<button>`, `<nav>`, `<main>`, `<label>`) before reaching for a generic `<div>` with an ARIA role — the native element gets keyboard and screen-reader behavior for free.
- Every interactive element must be reachable and operable by keyboard (tab order, `Enter`/`Space` activation); custom components built on `<div>` need `tabindex` and key handlers only when no semantic element fits.
- Images need `alt` text (empty `alt=""` for purely decorative images); form inputs need an associated `<label>`.
- Manage focus explicitly for dynamic UI: moving focus into a modal on open, back to the trigger on close.

## 9. Performance

- Batch DOM writes; avoid alternating reads and writes in a loop (layout thrashing). Build subtrees off-DOM (`DocumentFragment` or building via `innerHTML` once) then insert.
- Debounce/throttle expensive handlers (scroll, resize, input search) rather than running full logic on every event.
- Lazy-load offscreen images (`loading="lazy"`) and defer non-critical scripts (`defer`/`type="module"` is deferred by default).

## 10. Testing

- For logic-heavy modules (utils, state stores), write plain assertion-based tests runnable with `node --test` or a minimal browser test runner already in the project — no framework-specific test library needed.
- For UI behavior, prefer a quick manual checklist in the PR/response (what was clicked, what should happen) over skipping verification entirely.

## 11. Example: Wiring a Component

```js
// js/main.js
import { createCard } from "./components/card.js";

const list = document.querySelector("#card-list");
const data = await fetchCards();
data.forEach((item) => {
  list.appendChild(createCard({ title: item.title, body: item.body }));
});
```

```css
/* css/components/card.css */
.card {
  padding: var(--nu-space-6);
  border-radius: var(--nu-radius-lg);
  background: var(--nu-surface);
  border: 1px solid var(--nu-line-200);
}
```

## 12. Instructions for the Agent

1. Acknowledge that you are applying the `vanilla-js-frontend` skill.
2. Ensure any generated boilerplate strictly avoids the prohibited frameworks and CSS libraries — including not importing them "just for icons" or "just for one utility class."
3. If a `design.md`/`tokens.css` is present in the project, pull colors, spacing, and type from it rather than inventing new values.
4. When editing an existing project, match its existing structure and conventions rather than forcing this layout wholesale; apply these standards to new code and flag major deviations.
5. If asked for a UI component, write it exclusively in pure HTML/CSS/JS (or Web Components), following the factory-function pattern above by default.
