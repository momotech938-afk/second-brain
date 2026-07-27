# Module 18 — Server-Side Rendering and Hydration

How Angular renders on the server, hydrates in the browser, and when that's worth it. Myspce v1 is a **client-rendered SPA** (assumption A1 — the content is private and behind login), but you should understand SSR because you'll meet it at work (intercert.com-style public sites) and may add a public landing page later.

Back to [README](../README.md) · Previous: [Module 17](../17-performance/README.md) · Next: [Module 19](../19-testing/README.md)

> **v22:** incremental hydration is now the **default** hydration strategy (`withIncrementalHydration()` is deprecated; `withNoIncrementalHydration()` opts out). `provideServerRendering()` accepts an options object (e.g. `maxResponseBodySize`). `resource`/`rxResource` can be transfer-cached with an `id`.

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [SSR concepts and hydration](01-ssr-concepts.md) | CSR vs SSR vs prerendering, architecture, server routes and render modes, hydration, incremental hydration, event replay, HTTP transfer cache, SEO, deployment trade-offs, when Myspce should use it |
| 2 | [Platform-aware code](02-platform-aware-code.md) | browser-only APIs, `isPlatformBrowser`, `afterNextRender`, `DOCUMENT`, `REQUEST`, cookies/auth under SSR, hydration mismatches and debugging |

**Time:** ~3 h reading + 1 h practice.

## Self-assessment checklist

- [ ] I can explain CSR, SSR and SSG/prerendering and pick one for a given page.
- [ ] I can configure server routes with `RenderMode.Server`, `Client` and `Prerender`.
- [ ] I can explain hydration, incremental hydration and event replay.
- [ ] I know why direct `window`/`document`/`localStorage` access breaks SSR and how to fix it.
- [ ] I can avoid double-fetching with the HTTP transfer cache.
- [ ] I can debug hydration mismatch errors.
- [ ] I can justify Myspce's CSR choice and say which pages could be prerendered.
