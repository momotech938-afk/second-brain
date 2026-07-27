# Platform-Aware Code

Back to [Module 18](README.md) · Previous: [SSR concepts](01-ssr-concepts.md) · Next module: [19 Testing](../19-testing/README.md)

## 1. Learning objectives

- Write code that runs safely on the server and in the browser.
- Use `afterNextRender`, `isPlatformBrowser`, `DOCUMENT` and DI tokens instead of globals.
- Handle cookies and auth in SSR.
- Debug hydration mismatches.

## 2. Why it matters even for a CSR app

Writing platform-aware code from day one keeps the door open for SSR/prerendering later, makes tests easier (no real `window`), and avoids surprises in tools that render components outside a browser.

## 3. What breaks on the server

| Global | On the server |
|---|---|
| `window`, `document` (direct) | undefined (Node) or a limited DOM shim |
| `localStorage`, `sessionStorage` | missing |
| `navigator.clipboard`, `matchMedia`, `IntersectionObserver` | missing |
| `setInterval` started at construction | keeps the server render from finishing ("app never stabilises") |
| Third-party DOM libraries (Leaflet, Tiptap) | crash on import or init |

## 4. The tools

| Need | Use |
|---|---|
| DOM work after render | `afterNextRender` / `afterEveryRender` / `afterRenderEffect` — **never run on the server** |
| The document | `inject(DOCUMENT)` |
| Which platform | `isPlatformBrowser(inject(PLATFORM_ID))` |
| Storage | a token with a browser and a server implementation |
| Request data on the server | `inject(REQUEST)`, `REQUEST_CONTEXT`, `RESPONSE_INIT` from `@angular/core` (null in the browser) |
| Lazy browser libraries | dynamic `import()` inside `afterNextRender` |

## 5. Examples

### Theme store, SSR-safe

```typescript
// core/theme/theme-store.ts (platform-aware version)
import { DOCUMENT, Service, PLATFORM_ID, computed, effect, inject, signal } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Service()
export class ThemeStore {
  private doc = inject(DOCUMENT);
  private isBrowser = isPlatformBrowser(inject(PLATFORM_ID));
  private storage = inject(KV_STORAGE);       // from Module 07: localStorage in the browser, memory elsewhere

  readonly mode = signal<'light' | 'dark' | 'system'>((this.storage.getItem('myspce.theme') as any) ?? 'system');
  private systemDark = signal(false);
  readonly isDark = computed(() => this.mode() === 'dark' || (this.mode() === 'system' && this.systemDark()));

  constructor() {
    if (this.isBrowser) {
      const mq = this.doc.defaultView!.matchMedia('(prefers-color-scheme: dark)');
      this.systemDark.set(mq.matches);
      mq.addEventListener('change', e => this.systemDark.set(e.matches));
    }
    effect(() => {
      this.doc.documentElement.classList.toggle('dark', this.isDark());   // DOCUMENT works on the server too
      this.storage.setItem('myspce.theme', this.mode());
    });
  }
}
```

(`DOCUMENT` is exported from `@angular/core` in recent versions and from `@angular/common` in older ones; both work in v22.)

### Third-party library only in the browser

```typescript
constructor() {
  afterNextRender(async () => {
    const L = await import('leaflet');
    this.map = L.map(this.el().nativeElement);
  });
}
```

### Timers that don't block SSR

```typescript
constructor() {
  afterNextRender(() => {
    const id = setInterval(() => this.now.set(new Date()), 30_000);
    inject(DestroyRef).onDestroy(() => clearInterval(id));   // ⚠ inject() here is outside an injection context!
  });
}
```

Fix the trap above by injecting first:

```typescript
private destroyRef = inject(DestroyRef);
constructor() {
  afterNextRender(() => {
    const id = setInterval(() => this.now.set(new Date()), 30_000);
    this.destroyRef.onDestroy(() => clearInterval(id));
  });
}
```

### Cookies and auth under SSR

The Myspce auth design keeps the access token **in memory** and the refresh token in an **httpOnly cookie** ([Module 13](../13-http-and-api-integration/02-interceptors-and-auth.md)). On a server render, there's no in-memory token, and forwarding the user's cookies to the API from Node requires care (`inject(REQUEST)?.headers.get('cookie')`) and must never be cached. That complexity is a key reason private pages stay `RenderMode.Client`.

## 6. Hydration mismatches

Angular hydration expects the client's first render to match the server HTML.

| Cause | Fix |
|---|---|
| Rendering `Date.now()`/random values directly | compute once on the server and transfer, or render after hydration |
| Browser-only branches in the template (`@if (isBrowser)`) | move to `afterNextRender` or `@defer` |
| Invalid HTML (e.g. `<div>` inside `<p>`, missing `<tbody>`) | the browser "fixes" it, breaking the match — write valid HTML |
| Third-party scripts mutating the DOM before hydration | load them after hydration |
| Direct DOM manipulation (`innerHTML`, `appendChild`) | use templates; or mark the component `host: { ngSkipHydration: 'true' }` as a last resort |

Errors look like `NG0500: During hydration Angular expected <div> but found <p>`. Angular DevTools shows hydrated vs skipped components.

## 7. Common mistakes

| Mistake | Fix |
|---|---|
| `if (typeof window !== 'undefined')` scattered everywhere | central tokens/services + `afterNextRender` |
| `localStorage` in field initializers | storage token |
| Importing a DOM library at the top of a component file | dynamic import in `afterNextRender` |
| `inject()` inside `afterNextRender` callbacks | inject in the constructor/field |

## 8. Practice questions

1. Make `IdleLock` (Module 09) SSR-safe.
2. Make `CopyToClipboard` (Module 05) do nothing on the server.
3. Find two places in your practice app that would crash under SSR.

## 9. Interview questions

- **Q: How do you run code only in the browser?** `afterNextRender`/`afterEveryRender`, or check `isPlatformBrowser(PLATFORM_ID)`.
- **Q: What causes a hydration mismatch?** Different DOM between server and first client render: non-deterministic values, browser-only branches, invalid HTML, external DOM changes.

## 10. Completion checklist

- [ ] `ThemeStore` and `IdleLock` are platform-aware.
- [ ] No direct `window`/`localStorage` access outside dedicated services.
