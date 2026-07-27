# Module 17 — Performance Optimization

Make Myspce fast on a mid-range phone: render less, load less, measure before changing anything.

Back to [README](../README.md) · Previous: [Module 16](../16-advanced-components/README.md) · Next: [Module 18](../18-ssr-and-hydration/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Rendering performance](01-rendering-performance.md) | how Angular renders, OnPush + signals, immutable state, `track`, expensive templates, pure pipes and memoisation, virtual scrolling, images |
| 2 | [Loading and bundles](02-loading-and-bundles.md) | lazy routes, `@defer` (triggers, prefetch, placeholder/loading/error), bundle analysis, tree shaking, dependency reduction, caching and network optimisation |
| 3 | [Measuring](03-measuring.md) | Core Web Vitals, Lighthouse, Angular DevTools profiler, Chrome Performance panel, production build analysis, the Myspce performance checklist |

**Lab:** [Lab 13 — Optimise loading and rendering](../21-myspce-practice-labs/lab-13-performance.md)

**Time:** ~3 h reading + 2 h practice.

## Self-assessment checklist

- [ ] I can explain why OnPush + signals + immutable data minimise rendering work.
- [ ] I can find an expensive template expression and move it into `computed()` or a pure pipe.
- [ ] I can virtualise a long list with CDK `ScrollingModule`.
- [ ] I can use `NgOptimizedImage` with correct `width`/`height`/`priority`.
- [ ] I can use `@defer` with the right trigger and blocks.
- [ ] I can analyse the production bundle and remove a heavy dependency.
- [ ] I can measure LCP, INP and CLS and explain what each means.
- [ ] I can profile change detection with Angular DevTools.
- [ ] I know the rule: **measure first, then optimise**.
