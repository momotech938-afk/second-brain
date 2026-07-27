# Measuring Performance

Back to [Module 17](README.md) · Previous: [Loading and bundles](02-loading-and-bundles.md) · Next module: [18 SSR and hydration](../18-ssr-and-hydration/README.md)

## 1. Learning objectives

- Explain Core Web Vitals and their targets.
- Run Lighthouse correctly (production build, mobile, cold cache).
- Profile change detection with Angular DevTools and the browser Performance panel.
- Apply the Myspce performance checklist.

## 2. The rule

**Measure → find the biggest bottleneck → fix it → measure again.** Optimising without numbers wastes time and adds complexity.

## 3. Core Web Vitals

| Metric | Measures | Good | Myspce risk |
|---|---|---|---|
| **LCP** — Largest Contentful Paint | when the main content appears | ≤ 2.5 s | dashboard greeting image, slow `/bootstrap` |
| **INP** — Interaction to Next Paint | responsiveness to clicks/keys | ≤ 200 ms | heavy filtering on every keystroke, large lists |
| **CLS** — Cumulative Layout Shift | visual stability | ≤ 0.1 | images without size, skeletons of the wrong size, late-loading banners |

Supporting: TTFB (server response), FCP (first paint), TBT (main-thread blocking in lab tests).

## 4. Lighthouse the right way

1. `ng build` (production) and serve `dist/myspce/browser` with a static SPA server (`npx serve -s dist/myspce/browser`) or, better, your Docker/nginx image from [Module 20](../20-security-and-deployment/04-deployment-and-ci.md).
2. Chrome DevTools → Lighthouse → **Mobile**, "Navigation", clear storage.
3. For logged-in pages, log in first, then run Lighthouse in "Timespan" or "Snapshot" mode, or script it with Playwright + Lighthouse CI.
4. Run 3 times; use the median.

Never judge performance on `ng serve` (unminified, dev checks, HMR).

## 5. Profiling rendering

### Angular DevTools → Profiler

1. Start recording, perform the slow interaction (type in search, toggle a task), stop.
2. Each bar is a change-detection cycle; click it to see which components were checked and how long each took.
3. Look for: components checked that shouldn't be (Eager strategy, mutated inputs), and components with high self-time (expensive templates).

### Chrome Performance panel

- Record the interaction with CPU throttling 4×.
- Look for **long tasks** (> 50 ms, red corner), forced reflows ("Layout" after "Recalculate Style" in the same task), and large scripting blocks.
- Bottom-up view → your functions sorted by self time.

### Quick in-code timing (dev only)

```typescript
const t0 = performance.now();
const result = expensiveFilter(list);
console.debug(`filter took ${(performance.now() - t0).toFixed(1)} ms for ${list.length} items`);
```

## 6. Production build analysis

```bash
ng build --stats-json
npx esbuild-visualizer --metadata dist/myspce/stats.json --open
```

Record the initial bundle size in your README; set budgets slightly above it ([Module 01](../01-angular-fundamentals/04-configuration-and-builds.md)).

## 7. Myspce performance checklist

**Rendering**
- [ ] All components OnPush (v22 default); no `Eager` left after upgrades.
- [ ] No filtering/sorting/method calls in templates; derived data in `computed`.
- [ ] Every `@for` tracks by `id`.
- [ ] Lists > 100 items paginated or virtualised.

**Loading**
- [ ] Every feature route lazy; admin behind `canMatch`.
- [ ] Below-the-fold dashboard widgets deferred.
- [ ] Heavy libraries (editor, map, charts, PDF) lazy-imported.
- [ ] Initial JS within budget; no CommonJS warnings ignored.

**Network**
- [ ] One `/bootstrap` + one `/dashboard/summary` on first load.
- [ ] No duplicate requests (check Network tab on each page).
- [ ] Images: thumbnails for galleries, `NgOptimizedImage`, explicit sizes.
- [ ] nginx: long cache for hashed assets, `no-cache` for `index.html`, gzip/brotli on.

**Vitals targets (mobile, production build)**
- [ ] LCP ≤ 2.5 s on Dashboard and Notes.
- [ ] INP ≤ 200 ms when typing in Notes search.
- [ ] CLS ≤ 0.1 on every page.

## 8. Common mistakes

| Mistake | Fix |
|---|---|
| Measuring `ng serve` | production build |
| Desktop-only testing | Mobile emulation + CPU throttling |
| One run | median of three |
| Optimising code that isn't on the hot path | profile first |

## 9. Practice questions

1. Record a baseline Lighthouse report for Dashboard and Notes; write the numbers in `PERF.md`.
2. Use the Angular DevTools profiler to show that toggling one favourite checks only one `NoteCard`.
3. Introduce a deliberate CLS (image without size), measure it, fix it.

## 10. Interview questions

- **Q: What are the Core Web Vitals?** LCP (loading), INP (responsiveness), CLS (visual stability).
- **Q: How do you find which components run change detection too often?** Angular DevTools profiler.

## 11. Completion checklist

- [ ] Baseline metrics recorded.
- [ ] Checklist passes for v1 pages.
