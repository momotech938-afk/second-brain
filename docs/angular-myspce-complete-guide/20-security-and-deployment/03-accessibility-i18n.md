# Accessibility, Responsive Design and i18n Basics

Back to [Module 20](README.md) · Previous: [Auth and token storage](02-auth-and-tokens.md) · Next: [Deployment and CI/CD](04-deployment-and-ci.md)

## 1. Learning objectives

- Build Myspce screens that work with keyboard and screen readers.
- Manage focus on navigation and overlays.
- Respect user preferences (contrast, motion, colour scheme).
- Make layouts responsive and browser-compatible.
- Understand Angular's i18n options and locale formatting.

## 2. Accessibility essentials (WCAG 2.2 AA mindset)

| Area | Rule | Myspce example |
|---|---|---|
| Semantics | native elements first | `<button>` for actions, `<a routerLink>` for navigation, `<nav>`, `<main>`, headings in order |
| Names | every control has an accessible name | icon buttons: `aria-label="Copy password for GitHub"` |
| Forms | labels, errors linked | `<label for>`, `aria-invalid`, `aria-describedby` |
| Keyboard | everything works without a mouse | menus with arrows/Escape, cards activatable with Enter |
| Focus | visible and managed | `focus-visible:outline-2`; move focus to `<main>` or the page `<h1>` after navigation |
| Live regions | announce async changes | toasts `aria-live="polite"`, errors `role="alert"`, loading `role="status"` |
| Contrast | 4.5:1 text, 3:1 UI | check category chip colours in both themes |
| Motion | respect reduced motion | `motion-safe:` / `motion-reduce:` Tailwind variants |
| Target size | ≥ 24×24 px touch targets | icon buttons on mobile |

## 3. Focus management after navigation

Screen-reader users otherwise stay "stuck" on the clicked link after an SPA route change.

```typescript
// core/layout/shell.ts (addition)
private injector = inject(Injector);
private doc = inject(DOCUMENT);

constructor() {
  inject(Router).events.pipe(filter(e => e instanceof NavigationEnd), takeUntilDestroyed()).subscribe(() => {
    // afterNextRender needs an injector when called outside an injection context
    afterNextRender(() => this.doc.querySelector<HTMLElement>('main h1, main')?.focus(), { injector: this.injector });
  });
}
```

`<main id="main" tabindex="-1">` makes the container focusable without adding it to the tab order. Also add a "Skip to content" link as the first focusable element:

```html
<a href="#main" class="sr-only focus:not-sr-only focus:fixed focus:left-2 focus:top-2 focus:z-50 focus:rounded focus:bg-white focus:p-2">Skip to content</a>
```


## 4. Accessible patterns already in this guide

| Component | Pattern | Note |
|---|---|---|
| `ChipFilter` | `role="radiogroup"` + `role="radio"` + `aria-checked` | [Module 04](../04-component-communication/02-outputs-and-model.md) |
| `Toggle` | `role="switch"` + `aria-checked` | [Module 04](../04-component-communication/02-outputs-and-model.md) |
| `Tabs` | `tablist`/`tab`/`tabpanel`, arrow keys, roving `tabindex` | [Module 16](../16-advanced-components/01-content-projection.md) |
| `Modal` | native `<dialog>` | [Module 16](../16-advanced-components/04-reusable-ui-kit.md) |
| `Menu` | `aria-haspopup`, `aria-expanded`, arrows, Escape | [Module 16](../16-advanced-components/04-reusable-ui-kit.md) |
| Forms | labels, `aria-invalid`, error lists | [Module 12](../12-reactive-forms/04-signal-forms.md) |

For complex widgets (combobox, listbox, tree, grid), use **Angular Aria** (stable in v22) or the CDK rather than hand-rolling ARIA.

## 5. Preferences

```css
/* styles.css */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

`ThemeStore` already follows `prefers-color-scheme` in "system" mode. Router view transitions (`withViewTransitions()`) animate via `::view-transition-*` pseudo-elements, which the rule above does not reliably cover — disable them explicitly under reduced motion:

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*), ::view-transition-old(*), ::view-transition-new(*) { animation: none !important; }
}
```

## 6. Responsive design and browsers

- Mobile-first Tailwind classes; test at 360, 768, 1024, 1440 px.
- `min-h-dvh` instead of `100vh` on mobile.
- Sidebar becomes an off-canvas drawer below `lg`.
- Tables (credentials) become stacked rows on small screens.
- Angular 22 targets the **Baseline "widely available"** browser set for its release date — modern evergreen browsers. Don't add polyfills unless you have a specific need; check the support table on angular.dev.

## 7. i18n basics

Myspce is single-user and English (confirmed language setting exists in Settings, but only English is planned). Still, know the options:

| Approach | How | When |
|---|---|---|
| Locale formatting only | `LOCALE_ID: 'en-IN'`, `registerLocaleData`, pipes | ✅ Myspce now ([Module 06](../06-pipes/01-built-in-pipes.md)) |
| Angular built-in i18n | `i18n` attributes / `$localize`, `ng extract-i18n`, one build per locale | multi-language apps where build-per-locale is acceptable |
| Runtime translation library | e.g. Transloco | switch language at runtime without rebuilds |

Built-in i18n example (read-only knowledge):

```html
<h1 i18n="@@dashboardGreeting">Good evening, {{ name() }}</h1>
<span i18n>{count, plural, =0 {No tasks} =1 {1 task} other {{{count}} tasks}}</span>
```

```typescript
const msg = $localize`:@@noteSaved:Note saved`;
```

Even without translations, avoid string concatenation for sentences and keep user-facing strings in templates (easy to extract later).

## 8. Testing accessibility

- Playwright + axe on every page ([Module 19](../19-testing/05-e2e-and-ci.md)).
- Keyboard-only pass: Tab through Dashboard → Notes → open note → menu → delete confirm → toast Undo.
- Screen reader smoke test with NVDA (Windows) or VoiceOver (macOS/iOS).
- Lighthouse accessibility score as a quick indicator (not a substitute).

## 9. Common mistakes

| Mistake | Fix |
|---|---|
| `<div (click)>` | `<button type="button">` |
| Placeholder as the only label | real `<label>` |
| Colour-only meaning (category tags) | text label too |
| Removing focus outlines | style `focus-visible` instead |
| Modals that don't return focus | native `<dialog>`/CDK |

## 10. Interview questions

- **Q: How do you make SPA navigation accessible?** Move focus to the new page's main heading/container and announce the title; provide a skip link.
- **Q: When do you use ARIA?** When no native element provides the semantics; native HTML first.

## 11. Completion checklist

- [ ] Skip link and focus-on-navigation implemented.
- [ ] axe scan clean on v1 pages.
- [ ] Keyboard-only walkthrough passes.
