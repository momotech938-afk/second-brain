# Lab 13 — Performance: Measure, Defer, Budget

Back to [Labs](README.md) · Solution: [lab-13-solution.md](solutions/lab-13-solution.md) · Previous: [Lab 12](lab-12-tests.md) · Next: [Lab 14](lab-14-production-build.md)

## 1. Learning objective

Measure first, then fix: get a baseline (bundle sizes, Lighthouse, Angular DevTools profiler), then apply the changes that matter for Myspce. That means `@defer` for below-the-fold dashboard widgets and the rich editor, lean initial bundles, stable list rendering, optimised images, and budgets that stop regressions.

## 2. Prerequisites

All of [Module 17](../17-performance/README.md), [Lab 7](lab-07-ui-states.md), [Lab 10](lab-10-search-filter-sort-paginate.md).

## 3. Problem statement

Myspce is "mobile responsive mainly" (confirmed). On a mid-range phone over 4G, the dashboard should be usable fast and typing in Notes search must never stutter. You can't optimise what you haven't measured.

## 4. Expected behaviour (targets, measured on a production build)

| Metric | Target |
|---|---|
| Initial JS (transferred, gzip/brotli) | ≤ 200 kB |
| Largest lazy chunk (excluding the rich editor) | ≤ 100 kB |
| Lighthouse mobile Performance (login and dashboard, mock API) | ≥ 90 |
| LCP / INP / CLS (Lighthouse or Web Vitals) | ≤ 2.5 s / ≤ 200 ms / ≤ 0.1 |
| Toggling one favourite | re-renders one card (DevTools profiler) |
| Typing in search | no frame > 50 ms in the Performance panel |

These are **your** targets for learning. Real thresholds come from measuring real devices.

## 5. Relevant concepts

`@defer` triggers (`on viewport`, `on idle`, `on interaction`, `prefetch on`), lazy routes, `angular.json` budgets, `ng build --stats-json` + an esbuild analyser, `NgOptimizedImage`, `track`, OnPush (the v22 default), `computed` memoisation, Core Web Vitals.

## 6. Step-by-step implementation

1. **Baseline.** Run `ng build`, note the "Initial total" and the largest lazy chunks in the CLI output. Serve `dist/…/browser` with `npx serve -s`. Run Lighthouse (mobile, incognito). Save the numbers in `docs/perf-baseline.md`.
2. **Bundle analysis.** `ng build --stats-json`, open it in an esbuild analyser. Look for libraries in `main` that the first screen doesn't need.
3. **Dashboard.** Keep the greeting, quick actions and overview tiles eager. Wrap Recent Activity + Today's Tasks in `@defer (on viewport)` and the Week Progress chart in `@defer (on viewport; prefetch on idle)`, each with a same-size `@placeholder` (no layout shift).
4. **Editor.** Load the rich editor with `@defer (on interaction; prefetch on idle)`. Tiptap is assumption A5; use a plain `<textarea>` placeholder component if you haven't added it.
5. **Lists.** Confirm `track note.id`. Confirm `computed` derivations aren't recreated per render. Check in the profiler that toggling a favourite touches one card.
6. **Images.** Use `NgOptimizedImage` for the avatar and any note/memory images: `ngSrc`, `width`/`height`, `priority` only for the LCP image.
7. **Fonts and icons.** Import only the icons you use. Preconnect to the font origin, or self-host fonts.
8. **Budgets.** Set `initial` warning/error and `anyComponentStyle` budgets in `angular.json` slightly above your *improved* numbers.
9. **Re-measure.** Compare against the baseline. Keep only the changes that helped.

## 7. Complete working code

Dashboard `@defer` blocks, placeholder components, the budgets block, the `NgOptimizedImage` avatar, and a before/after template are in the [solution](solutions/lab-13-solution.md).

## 8. Important lines explained

- `@defer (on viewport)` — the widget's code and data load only when it scrolls into view. The placeholder must be a **single root element**.
- `prefetch on idle` — download the chunk early in idle time, render on the trigger, so the user feels no delay.
- `@placeholder (minimum 300ms)` — avoids a flash when the chunk is already cached.
- `"budgets": [{ "type": "initial", "maximumWarning": "…", "maximumError": "…" }]` — the build fails before a regression ships.
- `<img ngSrc="…" width="32" height="32">` — explicit size prevents CLS; `NgOptimizedImage` warns about common mistakes in dev mode.

## 9. Expected output

`ng build` prints separate lazy chunks for the deferred widgets (names vary). Lighthouse "Reduce unused JavaScript" no longer lists dashboard widgets or the editor.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Deferred component still in the main chunk | it's also imported/used **outside** `@defer` somewhere (or referenced in TS code); deferral only works when the component is used exclusively inside the block |
| Layout jumps when a widget loads | placeholder height differs; give it the same min-height |
| `on viewport` never fires | placeholder has multiple roots or zero size |
| Lighthouse score swings wildly | run 3× and take the median; use incognito; close DevTools' other panels |
| Optimising based on `ng serve` | always measure a production build |
| Budgets always red | set them from measured numbers, then tighten gradually |

## 11. Test cases

```typescript
import { DeferBlockBehavior, DeferBlockState } from '@angular/core/testing';

it('renders the placeholder, then the widget when the defer block completes', async () => {
  TestBed.configureTestingModule({ deferBlockBehavior: DeferBlockBehavior.Manual, providers: provideTestDefaults() });
  const fixture = TestBed.createComponent(DashboardPage);
  await fixture.whenStable();
  const [activity] = await fixture.getDeferBlocks();
  expect(fixture.nativeElement.querySelector('app-recent-activity-widget')).toBeNull();

  await activity!.render(DeferBlockState.Complete);
  expect(fixture.nativeElement.querySelector('app-recent-activity-widget')).not.toBeNull();
});
```

Plus a CI guard: the production build fails when `initial` exceeds the error budget (that *is* the test).

## 12. Independent practice task

1. Credentials list: render 2,000 mock credentials with `cdk-virtual-scroll-viewport` (fixed 56 px rows). Profile before and after. Write down whether it was worth it at Myspce's realistic data size (likely tens to hundreds).
2. Add a **Web Vitals** logger for development only: dynamically `import('web-vitals')` and `console.table` LCP, INP and CLS. Make sure it's excluded from production bundles.

## 13. Solution

See [solutions/lab-13-solution.md](solutions/lab-13-solution.md).

## 14. Completion checklist

- [ ] Baseline recorded; after-numbers recorded.
- [ ] Dashboard widgets and editor deferred with stable placeholders.
- [ ] Budgets set and enforced.
- [ ] Avatar/images via `NgOptimizedImage`.
- [ ] Virtual-scroll experiment + dev-only Web Vitals logger.
- [ ] Tagged `lab-13`.
