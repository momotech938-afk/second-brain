# RxJS Interop, Comparisons, Testing and Leaks

Back to [Module 14](README.md) · Previous: [Subjects and multicasting](03-subjects-and-multicasting.md) · Next module: [15 State management](../15-state-management/README.md)

## 1. Learning objectives

- Move between Observables, signals and Promises correctly.
- Choose RxJS vs Promise vs signal for a task.
- Test RxJS code with marble diagrams (`TestScheduler`) and fake timers.
- Spot and fix subscription leaks.

## 2. Interop APIs (`@angular/core/rxjs-interop`)

| From → To | API | Notes |
|---|---|---|
| Observable → signal | `toSignal(obs$, { initialValue })` | subscribes immediately; unsubscribes with the injection context; `requireSync: true` for sync sources |
| signal → Observable | `toObservable(sig)` | emits after changes via an effect (async, coalesced) |
| Observable → output | `outputFromObservable(obs$)` | component output backed by a stream |
| output → Observable | `outputToObservable(cmp.someOutput)` | subscribe to a component's output as a stream |
| Observable-backed async data | `rxResource({ params, stream })` | resource semantics |
| Observable → Promise | `firstValueFrom(obs$)` / `lastValueFrom(obs$)` | first value / value at completion |
| Promise → Observable | `from(promise)` or `defer(() => promise())` | `defer` for laziness |
| Teardown | `takeUntilDestroyed(destroyRef?)` | |

## 3. RxJS vs Promise vs signal

| Need | Choose |
|---|---|
| Hold UI/app state | **signal** |
| Derived state | **computed** |
| Load data that depends on state | **resource / httpResource** |
| One-shot async action (save, delete) | **Promise** (`firstValueFrom`) or Observable — either is fine |
| Stream of events with timing (debounce, throttle, polling) | **RxJS** |
| Cancel previous work when new input arrives | **RxJS `switchMap`** (or resources) |
| Retry with backoff, timeouts | **RxJS** |
| Combine several async sources with rules | **RxJS** |

## 4. Marble testing with `TestScheduler`

Marble syntax: `-` = 1 frame of virtual time, `a` = value, `|` = complete, `#` = error, `^` = subscription point, `()` = synchronous group.

```typescript
// shared/utils/search-pipe.ts
import { Observable, debounceTime, distinctUntilChanged, map, switchMap } from 'rxjs';

export const searchPipe = (search: (q: string) => Observable<string[]>, wait = 300) =>
  (q$: Observable<string>) => q$.pipe(map(q => q.trim()), debounceTime(wait), distinctUntilChanged(), switchMap(search));
```

```typescript
// shared/utils/search-pipe.spec.ts
import { TestScheduler } from 'rxjs/testing';

describe('searchPipe', () => {
  let scheduler: TestScheduler;
  beforeEach(() => (scheduler = new TestScheduler((actual, expected) => expect(actual).toEqual(expected))));

  it('debounces, de-duplicates and switches', () => {
    scheduler.run(({ cold, hot, expectObservable }) => {
      const input  = hot('a-b 300ms -c', { a: 'an', b: 'ang', c: 'ang ' });
      const search = (q: string) => cold('--r|', { r: [`result:${q}`] });
      //                          b arrives at frame 2 → debounce ends at 302 → search emits 2 frames later
      expectObservable(input.pipe(searchPipe(search, 300))).toBe('304ms r', { r: ['result:ang'] });
    });
  });
});
```

Read it as: only `'ang'` survives the debounce; `'ang '` trims to the same value and is dropped by `distinctUntilChanged`.

Marble tests take practice; for component-level tests, **fake timers** are often simpler:

```typescript
vi.useFakeTimers();
// trigger input…
await vi.advanceTimersByTimeAsync(300);
// assert request…
vi.useRealTimers();
```

## 5. Finding and fixing leaks

| Smell | Fix |
|---|---|
| `.subscribe()` in a component with no teardown on an infinite source | `takeUntilDestroyed()` or `toSignal` |
| `takeUntilDestroyed` placed before `switchMap`/`shareReplay` | put it **last** |
| Subscribing in `ngOnInit` to `route.params` and never unsubscribing | `takeUntilDestroyed(this.destroyRef)` or input binding |
| Root service subscribing per component call | subscribe once, expose a signal |
| `shareReplay(1)` on an infinite source | `{ bufferSize: 1, refCount: true }` |

Detect with heap snapshots ([Module 09](../09-lifecycle-and-change-detection/03-cleanup-and-memory-leaks.md)) or by counting active subscriptions in development:

```typescript
const counted = <T>(label: string) => (src: Observable<T>) => new Observable<T>(sub => {
  console.count(`${label} +1`);
  const s = src.subscribe(sub);
  return () => { console.count(`${label} -1`); s.unsubscribe(); };
});
```

## 6. Myspce example — where RxJS stays in a signals-first app

| Feature | RxJS used for |
|---|---|
| Global search | `debounceTime` + `switchMap` (or v22 `debounced()` + resource) |
| Idle auto-lock | `merge(fromEvent…)` + `switchMap(timer)` |
| Online/offline | `fromEvent` + `toSignal` |
| Save button | `exhaustMap` |
| Autosave | `debounceTime` + `switchMap`/`concatMap` |
| Retry/backoff | `retry({ delay })` in interceptors |
| Uploads | `mergeMap(…, 3)` for concurrency |
| Router events | `filter(e => e instanceof NavigationEnd)` |

Everything else (stores, derived lists, view state) is signals.

## 7. Practice questions

1. Write a marble test for `exhaustMap` ignoring clicks while a save is running.
2. Convert `toObservable(this.page)` + `switchMap(api.list)` + `toSignal` into an `rxResource`. What did you gain?
3. Audit one component for subscription leaks and fix them.

## 8. Interview questions

- **Q: What does `toSignal` need?** An injection context (or `injector` option) and an `initialValue` unless the source emits synchronously (`requireSync`).
- **Q: How do you unit-test time-based RxJS logic?** Marble tests with `TestScheduler.run()` (virtual time) or fake timers.
- **Q: Why put `takeUntilDestroyed` last?** Operators after it (like `switchMap` or `shareReplay`) could keep inner subscriptions alive after the source completes.

## 9. Completion checklist

- [ ] I wrote one marble test and one fake-timer test.
- [ ] I can map every Myspce feature to signals/RxJS/resources.
- [ ] My code has no unmanaged subscriptions.
