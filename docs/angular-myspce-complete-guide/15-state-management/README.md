# Module 15 — State Management

Where Myspce's state lives, how it changes, and how to keep it simple. Start with signals in services; reach for NgRx only when the problem demands it.

Back to [README](../README.md) · Previous: [Module 14](../14-rxjs/README.md) · Next: [Module 16](../16-advanced-components/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [State fundamentals](01-state-fundamentals.md) | local vs shared, server vs client state, normalisation, immutability, auth/user state, loading and errors, choosing an approach |
| 2 | [Signal stores](02-signal-stores.md) | the service-with-signals pattern, resources inside stores, optimistic updates, `linkedSignal` for dependent state, testing |
| 3 | [NgRx and SignalStore](03-ngrx-and-signal-store.md) | Store/actions/reducers/selectors/effects, Entity, `@ngrx/signals` SignalStore, trade-offs, when they're worth it |

**Lab:** [Lab 10 — Search, filter, sort, paginate](../21-myspce-practice-labs/lab-10-search-filter-sort-paginate.md)

**Time:** ~3 h reading + 2 h practice.

## Myspce's state decision (single user, v1)

| State | Where | Tool |
|---|---|---|
| Session (user, token, status) | `AuthStore` (root) | signals |
| Bootstrap (theme tokens, nav, flags) | `BootstrapStore` (root) | signal + `provideAppInitializer` |
| Theme mode | `ThemeStore` (root) | signal + effect to localStorage |
| Notes / tasks / credentials lists | feature stores (root) or page resources | `httpResource`/`rxResource` + signals |
| Filters, page, sort | **URL** query params | router input binding |
| Form drafts | the form | Signal Forms |
| Dialog open, menu open, hover | component | signals |
| Admin panel config being edited | `AdminStore` (admin route) | signals |

No NgRx in v1. Re-evaluate if Myspce becomes multi-user with real-time collaboration.

## Self-assessment checklist

- [ ] I can classify any piece of Myspce state as local, shared, URL, server or form state.
- [ ] I can write a signal store with private writable state, public read-only state and actions.
- [ ] I can implement optimistic updates with rollback.
- [ ] I can normalise entities by id and derive lists with `computed`.
- [ ] I can explain NgRx actions, reducers, selectors and effects.
- [ ] I can build a small `signalStore` with `withState`, `withComputed`, `withMethods`.
- [ ] I can justify why Myspce v1 doesn't need NgRx.
- [ ] I can test stores with fake APIs.
