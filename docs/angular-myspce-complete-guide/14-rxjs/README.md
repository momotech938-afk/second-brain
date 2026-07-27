# Module 14 — RxJS Essentials

RxJS is Angular's tool for **events and async over time**: typing, debouncing, cancelling, retrying, polling, combining requests. With signals handling state, you need a focused subset of RxJS — learned properly.

Back to [README](../README.md) · Previous: [Module 13](../13-http-and-api-integration/README.md) · Next: [Module 15](../15-state-management/README.md)

> Read this module **before** Module 13 if Observables are new to you.

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Observables](01-observables.md) | Observable, observer, subscription, cold vs hot, creation functions, lifecycle and cleanup, error propagation |
| 2 | [Operators](02-operators.md) | `map/filter/tap/scan`, the four flattening operators, `catchError/retry/finalize`, `debounceTime/distinctUntilChanged/startWith`, combination operators, search autocomplete, dependent requests, race conditions |
| 3 | [Subjects and multicasting](03-subjects-and-multicasting.md) | `Subject`, `BehaviorSubject`, `ReplaySubject`, `shareReplay`, when signals replace them |
| 4 | [Interop and testing](04-interop-and-testing.md) | signals ↔ Observables, RxJS vs Promises vs signals, marble diagrams and `TestScheduler`, common leaks |

**Time:** ~5 h reading + 3 h practice.

## Self-assessment checklist

- [ ] I can explain what happens on `subscribe()` and when an Observable completes.
- [ ] I can explain cold vs hot with an HTTP example and a DOM event example.
- [ ] I can create Observables with `of`, `from`, `interval`, `timer`, `defer`, `fromEvent`, `throwError`.
- [ ] I can choose between `switchMap`, `mergeMap`, `concatMap`, `exhaustMap` for a scenario and explain the trade-off.
- [ ] I can build debounced search with cancellation.
- [ ] I can combine requests with `forkJoin` and `combineLatest` and know their completion rules.
- [ ] I can handle errors without killing a long-lived stream.
- [ ] I know when a `BehaviorSubject` should become a signal.
- [ ] I can use `shareReplay` safely.
- [ ] I can read and write marble diagrams and test with `TestScheduler` or fake timers.
- [ ] I can find and fix subscription leaks.
