# Module 07 — Services and Dependency Injection

Services hold Myspce's logic and shared state; DI creates them and hands them to whoever needs them. This module covers the v22 `@Service()` decorator, `inject()`, providers, tokens and the injector hierarchy, ending with a full API-layer design.

Back to [README](../README.md) · Previous: [Module 06](../06-pipes/README.md) · Next: [Module 08](../08-signals-and-reactivity/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Services and `inject()`](01-services-and-inject.md) | separation of concerns, `@Service()` (v22), `@Injectable`, `inject()`, constructor injection, injection context, `injectAsync()` |
| 2 | [Providers and tokens](02-providers-and-tokens.md) | `useClass`, `useValue`, `useFactory`, `useExisting`, `InjectionToken`, multi providers, optional and self/skipSelf |
| 3 | [Injector hierarchy](03-injector-hierarchy.md) | root, environment (route) and element injectors, resolution order, lifetimes, circular dependencies |
| 4 | [Myspce API layer](04-myspce-api-layer.md) | API services vs stores vs facades, typed base helpers, mocking, testing |

**Time:** ~4 h reading + 2 h practice.

## Self-assessment checklist

- [ ] I can explain what belongs in a service and what belongs in a component.
- [ ] I can write a root service with `@Service()` and with `@Injectable({ providedIn: 'root' })`, and say when to use each.
- [ ] I can use `inject()` in fields and explain what an injection context is.
- [ ] I can read and convert constructor injection.
- [ ] I can provide a value with an `InjectionToken` and override it in tests.
- [ ] I can use `useClass`, `useValue`, `useFactory`, `useExisting` correctly.
- [ ] I can explain the resolution order: element injectors → environment injectors → root → platform → null.
- [ ] I can scope a service to a route or component and predict how many instances exist.
- [ ] I can diagnose `NG0200` (circular dependency) and `NG0201` (no provider).
- [ ] I designed `NotesApi`, `NotesStore` and their test doubles for Myspce.
