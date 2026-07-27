# Providers and Injection Tokens

Back to [Module 07](README.md) · Previous: [Services and `inject()`](01-services-and-inject.md) · Next: [Injector hierarchy](03-injector-hierarchy.md)

## 1. Learning objectives

- Configure providers with `useClass`, `useValue`, `useFactory`, `useExisting`.
- Create typed `InjectionToken`s for values, config and functions.
- Use multi providers and `provide*()` helper functions.
- Handle optional and hierarchical lookups.

## 2. Prerequisites

[Services and `inject()`](01-services-and-inject.md).

## 3. What is it?

A **provider** is a recipe: "when someone asks for token **T**, give them **this**". A **token** is the key: a class, or an `InjectionToken` for non-class values.

## 4. Why does Angular need it?

To swap implementations without touching consumers: a mock API in tests, a different storage on the server, a config value per environment, feature flags from the admin panel.

## 5. How it works internally

Each injector holds a map `token → provider record`. On `inject(T)`, the injector looks up `T`; if found, it runs the recipe once and caches the result. `providedIn: 'root'`/`@Service()` registers the recipe lazily on the token itself (tree-shakable): if nobody injects it, it's dropped from the bundle.

## 6. Syntax and API reference

```typescript
providers: [
  NotesApi,                                              // shorthand for { provide: NotesApi, useClass: NotesApi }
  { provide: NotesApi, useClass: MockNotesApi },         // swap implementation
  { provide: API_URL, useValue: '/api/v1' },             // a constant
  { provide: STORAGE, useFactory: () => (typeof localStorage !== 'undefined' ? localStorage : memoryStorage()) },
  { provide: NOTES_REPO, useExisting: NotesApi },        // alias: same instance under another token
  { provide: VALIDATORS, useValue: urlValidator, multi: true },   // collect many values under one token
]
```

| Recipe | Gives | Typical use |
|---|---|---|
| `useClass` | new instance of a class | mocks, alternative implementations |
| `useValue` | the value as-is | config, constants, test stubs |
| `useFactory` | result of a function (can `inject()` inside) | computed config, platform-specific choices |
| `useExisting` | another token's instance | aliases, exposing a class under an abstract token |
| `multi: true` | array of all provided values | plugins, validators, interceptors (legacy) |

### `InjectionToken`

```typescript
// core/http/api-url.token.ts
import { InjectionToken } from '@angular/core';
export const API_URL = new InjectionToken<string>('API_URL');

// Token with a default (tree-shakable, no provider needed unless overriding)
export const PAGE_SIZE = new InjectionToken<number>('PAGE_SIZE', { providedIn: 'root', factory: () => 20 });

// Token whose factory injects other things
export const NOW = new InjectionToken<() => Date>('NOW', { factory: () => () => new Date() });
```

`NOW` is a great testing trick: components call `inject(NOW)()` instead of `new Date()`, and tests provide a fixed clock.

### `provide*()` helper functions

Libraries (and you) expose configuration as functions returning providers:

```typescript
// core/feature-flags/provide-feature-flags.ts
import { EnvironmentProviders, makeEnvironmentProviders } from '@angular/core';

export const FEATURE_FLAGS = new InjectionToken<Record<string, boolean>>('FEATURE_FLAGS');

export function provideFeatureFlags(defaults: Record<string, boolean>): EnvironmentProviders {
  return makeEnvironmentProviders([{ provide: FEATURE_FLAGS, useValue: defaults }]);
}

// app.config.ts
providers: [provideFeatureFlags({ memories: false, files: false, calendar: false, habits: false })],
```

`EnvironmentProviders` can only be used in environment injectors (app config, route `providers`), not in component `providers` — a useful guard for app-wide setup.

### Abstract class as a token

```typescript
export abstract class SecretClipboard {
  abstract copy(text: string, clearAfterMs?: number): Promise<void>;
}

@Service({ autoProvided: false })
export class BrowserSecretClipboard extends SecretClipboard {
  async copy(text: string, clearAfterMs = 30_000) { /* navigator.clipboard … */ }
}

// app.config.ts
{ provide: SecretClipboard, useClass: BrowserSecretClipboard }
// consumers: inject(SecretClipboard) — they never import the concrete class
```

## 7. Smallest working example

```typescript
export const GREETING = new InjectionToken<string>('GREETING', { factory: () => 'Hello' });

@Component({ selector: 'app-hi', template: `{{ greeting }}, {{ name() }}` })
export class Hi {
  protected greeting = inject(GREETING);
  name = input('Pranjal');
}

// override for a subtree
@Component({ providers: [{ provide: GREETING, useValue: 'Namaste' }], imports: [Hi], template: `<app-hi />` })
export class IndianGreeting {}
```

## 8. Step-by-step

1. Without any provider, `GREETING` uses its factory → "Hello".
2. `IndianGreeting` provides a different value; its child `Hi` resolves the nearest provider → "Namaste".

## 9. Myspce real-world example — environment-aware config and a clock

```typescript
// app.config.ts (excerpt)
import { environment } from '../environments/environment';
providers: [
  { provide: API_URL, useValue: environment.apiUrl },
  provideFeatureFlags({ memories: false, files: false, calendar: false, habits: false }),
]

// features/dashboard/pages/dashboard-page.ts
export class DashboardPage {
  private now = inject(NOW);
  protected today = signal(this.now());
}

// dashboard-page.spec.ts
TestBed.configureTestingModule({
  providers: [{ provide: NOW, useValue: () => new Date('2026-10-28T21:42:00+05:30') }],
});
// → greeting is deterministically "Good evening"
```

## 10. Advanced example — factory provider choosing a storage backend

```typescript
export interface KeyValueStorage { getItem(k: string): string | null; setItem(k: string, v: string): void; removeItem(k: string): void; }

export const KV_STORAGE = new InjectionToken<KeyValueStorage>('KV_STORAGE', {
  providedIn: 'root',
  factory: () => {
    const doc = inject(DOCUMENT);
    const win = doc.defaultView;
    try {
      win?.localStorage.setItem('__probe', '1');
      win?.localStorage.removeItem('__probe');
      return win!.localStorage;
    } catch {
      const mem = new Map<string, string>();
      return { getItem: k => mem.get(k) ?? null, setItem: (k, v) => void mem.set(k, v), removeItem: k => void mem.delete(k) };
    }
  },
});
```

Consumers inject `KV_STORAGE` and never care whether storage is real, blocked, or (in SSR) absent.

## 11. Visual explanation

```text
token          recipe                      result
NotesApi   ──► useClass: MockNotesApi  ──► MockNotesApi instance
API_URL    ──► useValue: '/api/v1'     ──► '/api/v1'
KV_STORAGE ──► useFactory: () => …     ──► localStorage | in-memory map
NOTES_REPO ──► useExisting: NotesApi   ──► (same instance as NotesApi)
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Using a string as a token (`'API_URL'`) | `InjectionToken<string>` — typed and collision-free |
| `useClass` when you meant `useExisting` | `useClass` creates a **second** instance; `useExisting` aliases |
| Providing a root service again in a component `providers` by accident | creates a separate instance; state "mysteriously" isn't shared |
| Using `inject()` in a `useValue` | `useValue` isn't run; use `useFactory` |
| Putting `EnvironmentProviders` in component `providers` | compile/runtime error; use app or route providers |

## 13. Debugging guide

- `NG0201 No provider for InjectionToken API_URL` → you forgot to provide it (or it has no default factory).
- Two instances of a "singleton" → search for the class name in every `providers: [...]`.
- Type mismatch between token and provided value → `InjectionToken<T>` is typed; fix the value.

## 14. Best practices

- Tokens for config, functions and interfaces; classes for services.
- Give tokens sensible default factories when possible.
- Expose library-like setup as `provideX()` functions.
- Inject time and randomness through tokens for testability.

## 15. Angular vs React

`InjectionToken` + `useValue` ≈ creating a Context with a default value and wrapping a Provider. The difference: Angular resolves through a hierarchy automatically and supports factories with their own dependencies.

## 16. Testing

```typescript
TestBed.configureTestingModule({
  providers: [
    { provide: NotesApi, useValue: { list: () => of(page), toggleFavorite: vi.fn() } satisfies Partial<NotesApi> },
    { provide: API_URL, useValue: '/test' },
  ],
});
```

Stubs typed with `Partial<…>` keep them honest when the real API changes.

## 17. Practice questions

1. Create `NOW` and use it in `TimeAgoPipe` instead of a `now` argument. What's the trade-off?
2. Write `provideMyspceCore()` that bundles `API_URL`, feature flags and `SecretClipboard`.
3. When would `multi: true` be useful in Myspce?

## 18. Interview questions

- **Q: `useClass` vs `useExisting`?** `useClass` creates a new instance of the class; `useExisting` returns the instance already registered under another token.
- **Q: Why use `InjectionToken` instead of a string?** Type safety and uniqueness.
- **Q: What does a token's `factory` option do?** Provides a default value (tree-shakable) when no explicit provider exists.

## 19. Revision notes

`useClass` · `useValue` · `useFactory` (can inject) · `useExisting` (alias) · `multi` · `InjectionToken<T>` with optional `factory` · `makeEnvironmentProviders` + `provideX()`.

## 20. Completion checklist

- [ ] `API_URL`, `NOW`, `FEATURE_FLAGS`, `KV_STORAGE` tokens exist in my practice app.
- [ ] I overrode a token in a test.
- [ ] I can explain each recipe with an example.
