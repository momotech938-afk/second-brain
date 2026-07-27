# Loading Performance and Bundles

Back to [Module 17](README.md) · Previous: [Rendering performance](01-rendering-performance.md) · Next: [Measuring](03-measuring.md)

## 1. Learning objectives

- Split code with lazy routes and `@defer`.
- Choose `@defer` triggers and use placeholder/loading/error blocks well.
- Analyse bundles and reduce dependencies.
- Optimise network requests and caching.

## 2. Prerequisites

[Lazy loading](../10-routing/03-nested-layouts-and-lazy-loading.md), [Configuration and builds](../01-angular-fundamentals/04-configuration-and-builds.md).

## 3. Code splitting levels

| Level | Tool | Myspce example |
|---|---|---|
| Route | `loadComponent` / `loadChildren` | each feature (notes, credentials, admin…) |
| Block inside a page | `@defer` | dashboard widgets below the fold, Memories map, rich editor |
| Service | `injectAsync()` (v22) | PDF export, backup zipper |
| Library | dynamic `import('leaflet')` | map library, Tiptap, chart library |

## 4. `@defer` — syntax and behaviour

```html
@defer (on viewport; prefetch on idle) {
  <app-recent-activity-widget />
} @placeholder (minimum 300ms) {
  <div class="h-48 animate-pulse rounded-xl bg-slate-200 dark:bg-slate-800"></div>
} @loading (after 150ms; minimum 400ms) {
  <app-spinner />
} @error {
  <p role="alert">Couldn't load this widget.</p>
}
```

| Trigger | Fires when |
|---|---|
| `on idle` (default) | browser is idle (`requestIdleCallback`); v22: `on idle(500ms)` sets a timeout |
| `on viewport` | placeholder enters the viewport (IntersectionObserver) |
| `on interaction` | user clicks/keys on the placeholder (or on an element referenced as `on interaction(ref)` elsewhere in the template) |
| `on hover` | pointer/focus on the placeholder |
| `on immediate` | right after the non-deferred content renders |
| `on timer(2s)` | after a delay |
| `when condition()` | expression becomes truthy (one-way: once loaded, stays) |
| `prefetch on …` | download the chunk early, render later |

Rules:

- Components/directives/pipes used **only** inside the `@defer` block, and imported standalone, are split into a separate chunk. If they're used elsewhere in the template, they're eagerly bundled.
- `@placeholder` content is eager; keep it tiny. It also serves as the trigger element for `viewport`/`interaction`/`hover`.
- `@loading (after X; minimum Y)` avoids flicker for fast loads.
- In SSR apps, `@defer` blocks also define **incremental hydration** boundaries (`hydrate on …`) — default in v22 ([Module 18](../18-ssr-and-hydration/01-ssr-concepts.md)).

## 5. Myspce example — dashboard and editor

```html
<!-- dashboard-page.html: what's above the fold renders immediately -->
<app-greeting-header [name]="user().name" [now]="now()" [quote]="quote()" />
<app-quick-actions />
<app-overview-tiles [summary]="summary.value()" />

@defer (on viewport) {
  <div class="grid gap-4 lg:grid-cols-2">
    <app-recent-activity-widget />
    <app-todays-tasks-widget />
  </div>
} @placeholder {
  <div class="grid h-64 gap-4 lg:grid-cols-2"><div class="rounded-xl bg-slate-800/40"></div><div class="rounded-xl bg-slate-800/40"></div></div>
}
```

```html
<!-- note-editor-page.html: Tiptap is ~100 kB+; load it when the user starts editing -->
@defer (on interaction; prefetch on idle) {
  <app-rich-editor [(content)]="content" />
} @placeholder {
  <div tabindex="0" class="min-h-64 rounded-lg border p-4 text-slate-500">Click to start writing…</div>
}
```

(With the widget registry from [Module 16](../16-advanced-components/03-dynamic-components.md), widgets are already lazy chunks; `@defer` adds "only when visible".)

## 6. Bundle analysis

```bash
ng build --stats-json
npx esbuild-visualizer --metadata dist/myspce/stats.json --open      # or upload stats.json to https://esbuild.github.io/analyze/
```

What to look for:

| Finding | Action |
|---|---|
| A big library in `main` | is it used on the first screen? if not, lazy import it |
| Two date libraries | pick one (or use `Intl`) |
| Full icon set | import only used icons (per-icon imports) |
| `moment`/`lodash` (non-ES) | `date-fns`/native; `lodash-es` per-function imports |
| A feature component in `main` | something imports it eagerly — find the import |

Tree shaking removes unused **ES module** exports. CommonJS packages defeat it (the build warns: "module … is not ESM"). Prefer ESM libraries.

Budgets in `angular.json` stop regressions ([Module 01](../01-angular-fundamentals/04-configuration-and-builds.md)). Recent CLI versions can also merge very small lazy chunks during production builds (this started as an opt-in experiment around v20; check your version's release notes before relying on it).

## 7. Network optimisation and caching

| Technique | Myspce use |
|---|---|
| One `/bootstrap` call at startup | user, theme, nav, flags in one round trip |
| One `/dashboard/summary` instead of 8 count calls | the overview tiles |
| Resource sharing in stores | multiple widgets read `TasksStore`; one request |
| `httpResource` + params from URL | no request when inputs don't change |
| `shareReplay` for rarely-changing data | categories list |
| HTTP caching headers from the API | `Cache-Control` for avatars/thumbnails; `ETag` for lists |
| Hashed filenames + long cache for JS/CSS | `outputHashing: "all"` (default) + `Cache-Control: immutable` in nginx |
| `index.html` never cached long | `Cache-Control: no-cache` so new deploys are picked up |
| Preconnect to S3/CDN | `<link rel="preconnect" href="https://bucket.s3.ap-south-1.amazonaws.com">` |

Don't cache sensitive API responses (credentials reveal, user profile) in service workers or `Cache-Control: public`.

## 8. Common mistakes

| Mistake | Fix |
|---|---|
| `@defer` around above-the-fold content | delays LCP; render it eagerly |
| Using a deferred component elsewhere in the same template | it becomes eager; restructure |
| Heavy placeholder | defeats the purpose |
| Importing a lazy page's component in a shared barrel file | pulls it into `main` |
| `on idle` for everything | pick viewport/interaction for content users may never see |

## 9. Debugging

- Build output lists lazy chunks; deferred blocks appear as chunks named after their components.
- Network tab: verify the chunk downloads only when the trigger fires.
- If a deferred component is in `main`, search the template/imports for another reference.

## 10. Practice questions

1. Defer the Memories map until its tab is opened and the user scrolls to it.
2. Find the largest dependency in your build and lazy-load or replace it.
3. Which Myspce screens should prefetch on idle after login?

## 11. Interview questions

- **Q: How does `@defer` create a lazy chunk?** Standalone dependencies used only inside the block are compiled into a separate chunk loaded when the trigger fires.
- **Q: `on viewport` vs `prefetch on idle`?** The first decides when to render; the second downloads earlier without rendering.
- **Q: Why does CommonJS hurt bundle size?** It can't be statically analysed, so tree shaking can't remove unused code.

## 12. Completion checklist

- [ ] Dashboard below-the-fold widgets are deferred.
- [ ] Editor library loads on interaction.
- [ ] Initial bundle within budget; analysed once.
