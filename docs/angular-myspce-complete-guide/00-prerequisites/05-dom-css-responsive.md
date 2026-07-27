# DOM, Events, CSS and Responsive Design

Back to [Module 00](README.md) · Previous: [Async and error handling](04-async-and-error-handling.md) · Next: [npm, Git and DevTools](06-npm-git-devtools.md)

## 1. Learning objectives

- Explain the DOM, event bubbling and default actions.
- Write semantic, accessible HTML for Myspce screens.
- Lay out the app with Flexbox and Grid.
- Build mobile-first responsive layouts with Tailwind CSS v4 and dark mode.

## 2. Prerequisites

Basic HTML and CSS.

## 3. What is it?

The **DOM** is the browser's live tree of elements. Angular creates and updates it for you from templates; you rarely touch it directly. CSS styles it; responsive design makes it work from phone to desktop.

## 4. Why does Angular need it?

- Event bindings like `(click)` are DOM events; knowing bubbling explains bugs like "clicking delete also opens the card".
- Accessibility (labels, roles, focus) is HTML knowledge, not Angular knowledge.
- Myspce is **mobile-responsive first** (confirmed decision).

## 5. How it works — events

```text
click on <button> inside <article>
  capture: window → document → … → article → button
  target:  button
  bubble:  button → article → … → document → window
```

An `(click)` listener on the card and one on the delete button **both** fire unless you stop propagation.

```html
<article (click)="open(note)">
  <h3>{{ note.title }}</h3>
  <button type="button" (click)="remove(note.id); $event.stopPropagation()">Delete</button>
</article>
```

Default actions: a `<form>` submit reloads the page, a link navigates. `$event.preventDefault()` stops that (Angular forms do it for you when you use `(ngSubmit)` with `FormsModule`/`ReactiveFormsModule`).

## 6. Syntax reference — semantic HTML for Myspce

| UI piece | Element |
|---|---|
| Sidebar | `<nav aria-label="Main">` with `<a>` links |
| Top bar | `<header>` |
| Page content | `<main>` (one per page) |
| Note card | `<article>` with a heading |
| Settings tabs | `role="tablist"` / `role="tab"` / `role="tabpanel"` (or Angular Aria) |
| Toggles | `<button role="switch" aria-checked>` or `<input type="checkbox">` |
| Icons-only buttons | `<button aria-label="Copy password">` |
| Modal | `<dialog>` or CDK Dialog with focus trap |

## 7. Smallest working example — responsive card grid (Tailwind v4)

```html
<!-- Notes grid: 1 column on phones, 2 on tablets, 4 on desktop -->
<section class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
  <article class="rounded-xl bg-amber-50 p-4 shadow-sm dark:bg-slate-800">
    <h3 class="font-semibold text-slate-900 dark:text-slate-100">Angular Notes</h3>
    <p class="mt-1 text-sm text-slate-600 dark:text-slate-400">Components, Services, Routing…</p>
  </article>
</section>
```

## 8. Step-by-step

1. `grid-cols-1` is the **mobile** default (mobile-first).
2. `sm:` applies at ≥ 640 px, `lg:` at ≥ 1024 px.
3. `dark:` applies when dark mode is active (configured below).

## 9. Myspce example — the app shell layout

```html
<!-- core/layout/shell.html -->
<div class="flex min-h-dvh bg-slate-50 text-slate-900 dark:bg-slate-950 dark:text-slate-100">
  <!-- Sidebar: hidden on mobile, slides in via a toggle -->
  <aside class="fixed inset-y-0 left-0 z-30 w-64 -translate-x-full border-r border-slate-200 bg-white transition-transform
                lg:static lg:translate-x-0 dark:border-slate-800 dark:bg-slate-900"
         [class.translate-x-0]="sidebarOpen()">
    <nav aria-label="Main" class="p-4"><!-- nav items --></nav>
  </aside>

  <div class="flex min-w-0 flex-1 flex-col">
    <header class="sticky top-0 z-20 flex h-16 items-center gap-3 border-b border-slate-200 bg-white/80 px-4 backdrop-blur
                   dark:border-slate-800 dark:bg-slate-900/80">
      <button type="button" class="lg:hidden" aria-label="Open menu" (click)="sidebarOpen.set(true)">☰</button>
      <!-- search, theme toggle, bell, avatar -->
    </header>
    <main class="flex-1 p-4 lg:p-8">
      <router-outlet />
    </main>
  </div>
</div>
```

### Tailwind v4 setup and class-based dark mode

```css
/* src/styles.css */
@import "tailwindcss";

/* Dark mode follows a .dark class on <html>, controlled by ThemeStore (Module 08). */
@custom-variant dark (&:where(.dark, .dark *));

/* Design tokens the admin Theme Builder can override at runtime */
@theme {
  --color-primary: oklch(0.55 0.2 260);
  --radius-card: 0.75rem;
  --font-sans: "Inter", system-ui, sans-serif;
}
```

```json
// .postcssrc.json
{ "plugins": { "@tailwindcss/postcss": {} } }
```

```bash
npm install tailwindcss @tailwindcss/postcss postcss
```

`@theme` variables become utilities (`bg-primary`, `rounded-card`) **and** CSS variables (`var(--color-primary)`), so runtime theming works by changing the variable.

## 10. Advanced example — container queries for widgets

Dashboard widgets can be small or large depending on admin config. Container queries make them adapt to their **own** width, not the screen:

```html
<section class="@container rounded-card bg-white p-4 dark:bg-slate-900">
  <div class="grid grid-cols-2 gap-3 @md:grid-cols-4">
    <!-- stat tiles -->
  </div>
</section>
```

## 11. Visual explanation

```text
< 640px          640–1023px              ≥ 1024px
┌──────────┐     ┌──────────────────┐    ┌──────┬─────────────────────┐
│ ☰ topbar │     │ ☰ topbar         │    │ side │ topbar              │
├──────────┤     ├────────┬─────────┤    │ bar  ├──────┬──────┬───────┤
│ card     │     │ card   │ card    │    │      │ card │ card │ card  │
│ card     │     │ card   │ card    │    │      │ card │ card │ card  │
└──────────┘     └────────┴─────────┘    └──────┴──────┴──────┴───────┘
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `<div (click)>` used as a button | Use `<button type="button">` — keyboard and screen readers work for free |
| Inputs without labels | `<label for="email">` or `aria-label` |
| Designing desktop first, then patching mobile | Write base classes for mobile, add `sm:`/`lg:` upward |
| `h-screen` on mobile (address bar jumps) | `min-h-dvh` |
| Colour-only status (red/green tags) | Add text or an icon too |

## 13. Debugging guide

- DevTools → **Elements** → toggle device toolbar (Ctrl+Shift+M) to test widths.
- Event firing twice? Check bubbling; look at the **Event Listeners** tab on the element.
- Tailwind class not working? Check the class is complete in source (no string concatenation like `'bg-' + color`), because Tailwind scans source text.

## 14. Best practices

- Semantic HTML first, ARIA only when no native element fits.
- Mobile-first, then enhance.
- Keep tokens (colours, radius, fonts) in `@theme`; never hard-code hex values in components.

## 15. Angular vs React

Same DOM and CSS. Angular adds **view encapsulation** (component styles are scoped by default) instead of CSS Modules/CSS-in-JS. Event syntax differs: `(click)="f()"` vs `onClick={f}`.

## 16. Testing

Use Playwright's viewport options to test layouts at phone and desktop widths ([Module 19](../19-testing/05-e2e-and-ci.md)), and axe-core for accessibility checks.

## 17. Practice questions

1. Why does a click on "delete" open the note, and how do you stop it?
2. Build the credential row from the design (icon, name, URL, username, masked password, copy) responsive at 375 px.
3. Make the Settings toggles accessible with a keyboard.

## 18. Interview questions

- **Q: Event bubbling?** A: After the target handles an event, it propagates up through ancestors, triggering their listeners.
- **Q: Mobile-first CSS?** A: Base styles for small screens; media queries (or `sm:`/`lg:`) add rules for larger screens.

## 19. Revision notes

Bubbling ↑ · `stopPropagation` / `preventDefault` · semantic elements · Grid for 2D, Flex for 1D · Tailwind: base = mobile, `sm`/`md`/`lg`/`xl` upward, `dark:` variant.

## 20. Completion checklist

- [ ] Shell layout works at 375 px, 768 px and 1440 px.
- [ ] Dark mode toggles via a `.dark` class.
- [ ] Every icon button has an accessible name.
