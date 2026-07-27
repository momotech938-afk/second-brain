# Lab 13 — Solution

Back to [Lab 13](../lab-13-performance.md) · [Labs index](../README.md)

## Baseline template — `docs/perf-baseline.md`

```markdown
# Performance baseline — <date>

Build: `ng build` (production), served with `npx serve -s dist/myspce/browser`
Device: Lighthouse mobile emulation, incognito, median of 3 runs

| Metric | Before | After | Target |
|---|---|---|---|
| Initial total (raw / transferred) | | | ≤ 200 kB transferred |
| Largest lazy chunk | | | ≤ 100 kB |
| Lighthouse Performance — /login | | | ≥ 90 |
| Lighthouse Performance — /dashboard | | | ≥ 90 |
| LCP / INP / CLS — /dashboard | | | 2.5 s / 200 ms / 0.1 |

Biggest items in `main` (from the analyser):
1.
2.

Changes kept:
Changes reverted (no measurable gain):
```

## Dashboard with `@defer`

```html
<!-- features/dashboard/dashboard-page.html -->
<app-greeting-header [name]="auth.user()?.name ?? ''" />
<app-quick-actions />

<app-widget-frame title="Overview" [res]="summary">
  @if (summary.hasValue()) { <app-overview-tiles [summary]="summary.value()" /> }
</app-widget-frame>

<div class="mt-6 grid gap-4 lg:grid-cols-2">
  @defer (on viewport) {
    <app-recent-activity-widget />
  } @placeholder (minimum 300ms) {
    <div class="h-72 rounded-xl bg-white shadow-sm dark:bg-slate-900" aria-hidden="true"></div>
  } @loading (after 150ms; minimum 300ms) {
    <div class="h-72 animate-pulse rounded-xl bg-slate-200 dark:bg-slate-800" aria-hidden="true"></div>
  } @error {
    <app-error-state title="Couldn't load recent activity." (retry)="reloadPage()" />
  }

  @defer (on viewport) {
    <app-todays-tasks-widget />
  } @placeholder (minimum 300ms) {
    <div class="h-72 rounded-xl bg-white shadow-sm dark:bg-slate-900" aria-hidden="true"></div>
  }
</div>

@defer (on viewport; prefetch on idle) {
  <app-week-progress-widget class="mt-6 block" />
} @placeholder {
  <div class="mt-6 h-56 rounded-xl bg-white shadow-sm dark:bg-slate-900" aria-hidden="true"></div>
}
```

Points to check:
- Each widget fetches its own data (`httpResource` inside the widget), so data loading is deferred too. The dashboard's `today`/`week-progress` resources from Lab 7 move into the widgets.
- `@error` covers chunk-load failures (offline, or a new deploy that removed old chunks). `reloadPage()` calls `location.reload()`.
- Don't reference deferred components anywhere else, including the `imports` of other eager components, or they get pulled into the eager bundle.

## Editor

```html
<!-- features/notes/note-editor-page.html (body) -->
@defer (on interaction; prefetch on idle) {
  <app-rich-editor [(content)]="content" />
} @placeholder {
  <button type="button" class="block min-h-64 w-full rounded-lg border p-4 text-left text-slate-500">
    {{ content() ? 'Click to edit…' : 'Click to start writing…' }}
  </button>
}
```

A `<button>` placeholder is keyboard-focusable and activates on Enter/Space, so `on interaction` works for keyboard users too.

Until Tiptap is chosen (A5), `RichEditor` can be a `<textarea>` with `model()`. The defer boundary is what you're practising.

## Budgets

```json
// angular.json → projects.myspce.architect.build.configurations.production
"budgets": [
  { "type": "initial", "maximumWarning": "450kB", "maximumError": "600kB" },
  { "type": "anyComponentStyle", "maximumWarning": "4kB", "maximumError": "8kB" },
  { "type": "anyScript", "maximumWarning": "350kB", "maximumError": "500kB" }
]
```

Budgets measure **raw** (uncompressed) sizes, so they are larger than the transferred targets in the lab table. Set them from *your* measured numbers plus ~10–15%. The values above are placeholders. `anyScript` applies to every single script file (including `main`), so set it from your largest file; the CLI has no separate "lazy chunks only" budget type, so check the biggest lazy chunk manually in the build output.

## Avatar with `NgOptimizedImage`

```typescript
// shared/ui/avatar.ts
import { Component, computed, input } from '@angular/core';
import { NgOptimizedImage } from '@angular/common';

@Component({
  selector: 'app-avatar',
  imports: [NgOptimizedImage],
  template: `
    @if (src(); as s) {
      <img [ngSrc]="s" [width]="size()" [height]="size()" [alt]="name()" class="rounded-full object-cover" [priority]="priority()" />
    } @else {
      <span class="grid place-items-center rounded-full bg-blue-600 font-semibold text-white"
            [style.width.px]="size()" [style.height.px]="size()" aria-hidden="true">{{ initials() }}</span>
    }
  `,
})
export class Avatar {
  src = input<string | null>(null);
  name = input.required<string>();
  size = input(32);
  priority = input(false);
  protected initials = computed(() => this.name().split(/\s+/).map(w => w[0]).join('').slice(0, 2).toUpperCase());
}
```

Use `priority` only for an image that is the LCP element (rare in Myspce, where text is usually the LCP).

## Rendering checks

- `@for (note of notes(); track note.id)` everywhere. Never `track $index` for editable lists.
- Formatting through pure pipes (`date`, `timeAgo` with `now` as an argument), not method calls in templates.
- Toggling a favourite with immutable `update` (Lab 5) replaces one object, so with OnPush only that card's inputs change.

## Independent task 1 — virtual scroll experiment

```html
<cdk-virtual-scroll-viewport itemSize="56" class="h-[70dvh]">
  <app-credential-row *cdkVirtualFor="let c of items(); trackBy: byId" [credential]="c" class="block h-14" />
</cdk-virtual-scroll-viewport>
```

```typescript
protected byId = (_: number, c: CredentialSummary) => c.id;
```

Typical finding: at 2,000 rows, the initial render and scroll are much smoother with virtual scroll. At ~100 rows (realistic for one user) the difference isn't noticeable, and server pagination or "Load more" (Lab 10) is simpler. Write down what *you* measured.

## Independent task 2 — dev-only Web Vitals

```typescript
// app.config.ts or main.ts
import { isDevMode } from '@angular/core';

if (isDevMode()) {
  import('web-vitals').then(({ onLCP, onINP, onCLS }) => {
    const log = (m: { name: string; value: number; rating: string }) => console.table({ [m.name]: { value: Math.round(m.value * 1000) / 1000, rating: m.rating } });
    onLCP(log); onINP(log); onCLS(log);
  });
}
```

`isDevMode()` is `false` in production builds and the dynamic `import()` creates a separate chunk, so production users never download it. Check the build output to confirm. To measure real users in production, send these metrics to your own analytics endpoint. That is a later decision, not part of v1.
