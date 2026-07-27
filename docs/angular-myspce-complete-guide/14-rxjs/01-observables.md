# Observables

Back to [Module 14](README.md) · Next: [Operators](02-operators.md)

## 1. Learning objectives

- Explain Observable, Observer and Subscription precisely.
- Distinguish cold and hot Observables.
- Create Observables with the main creation functions.
- Manage subscription lifecycle and error/complete semantics.

## 2. Prerequisites

[Async JavaScript](../00-prerequisites/04-async-and-error-handling.md).

## 3. What is it?

**Simple:** an Observable is a recipe for a stream of values over time. Nothing happens until someone subscribes.

**Precise:** a function that, when subscribed with an **observer** (`next`, `error`, `complete` callbacks), starts producing values and returns a **Subscription** whose `unsubscribe()` tears the producer down. After `error` or `complete`, no more values are delivered.

## 4. Why does Angular need it?

Some things are naturally streams: keystrokes, route events, WebSocket messages, timers, retries, HTTP requests you may want to cancel. Promises can't represent many values or cancellation; signals don't model time. RxJS does both.

## 5. How it works internally

```typescript
const ticks$ = new Observable<number>(subscriber => {
  let n = 0;
  const id = setInterval(() => subscriber.next(n++), 1000);   // producer
  return () => clearInterval(id);                             // teardown on unsubscribe/complete/error
});

const sub = ticks$.subscribe({
  next: v => console.log(v),
  error: e => console.error(e),
  complete: () => console.log('done'),
});
sub.unsubscribe();   // runs the teardown
```

Contract: `next*` then optionally one `error` **or** one `complete`. Errors stop the stream unless handled.

### Cold vs hot

| | Cold | Hot |
|---|---|---|
| Producer created | per subscriber | once, shared |
| Late subscribers | get everything from the start | miss earlier values |
| Examples | `http.get()`, `of()`, `interval()`, `timer()` | `fromEvent(document, 'click')`, `Subject`, router events |
| Angular gotcha | two subscriptions = two HTTP requests | values before subscribing are lost |

`fromEvent` is technically cold (each subscriber adds a listener) but behaves "hot" because the event source exists independently.

## 6. Syntax and API reference — creation functions

| Function | Emits | Myspce use |
|---|---|---|
| `of(a, b, c)` | the values, then completes | mock data in tests |
| `from(arrayOrPromiseOrIterable)` | each item / the resolved value | wrap a Promise API |
| `interval(ms)` | 0, 1, 2… every ms | polling, clocks |
| `timer(delay, period?)` | after delay (then every period) | debounce helpers, backoff |
| `fromEvent(target, 'event')` | DOM events | keyboard shortcuts, online/offline |
| `defer(() => obs)` | creates the Observable at subscribe time | lazy Promise calls, fresh timestamps |
| `throwError(() => err)` | errors immediately | tests, rethrow in `catchError` |
| `EMPTY` | completes immediately | "skip" in `switchMap` |
| `NEVER` | nothing, never completes | placeholders |

```typescript
import { defer, from, fromEvent, interval, of, throwError, timer, EMPTY } from 'rxjs';

const online$ = fromEvent(window, 'online');
const fresh$ = defer(() => from(crypto.subtle.digest('SHA-256', data)));   // runs at subscribe, not at creation
```

## 7. Smallest working example

```typescript
import { interval, take } from 'rxjs';

interval(500).pipe(take(3)).subscribe({
  next: v => console.log('tick', v),
  complete: () => console.log('complete'),
});
// tick 0, tick 1, tick 2, complete
```

## 8. Step-by-step

1. `interval(500)` would emit forever.
2. `take(3)` completes after three values, which also tears down the interval.
3. Completing frees resources; no manual unsubscribe needed here.

## 9. Myspce real-world example — online/offline banner

```typescript
// core/network/network-status.ts
import { Service } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { fromEvent, map, merge, startWith } from 'rxjs';

@Service()
export class NetworkStatus {
  readonly online = toSignal(
    merge(fromEvent(window, 'online'), fromEvent(window, 'offline')).pipe(
      map(() => navigator.onLine),
      startWith(navigator.onLine),
    ),
    { requireSync: true },
  );
}
```

```html
<!-- core/layout/shell.html -->
@if (!network.online()) {
  <div role="status" class="bg-amber-500 px-4 py-1 text-center text-sm text-black">You're offline. Changes won't be saved.</div>
}
```

`startWith` provides the current value synchronously, so `requireSync: true` is valid and the signal's type has no `undefined`.

## 10. Advanced example — HTTP is cold (and why that matters)

```typescript
const notes$ = this.http.get<Paginated<Note>>('/api/v1/notes');

notes$.subscribe();   // request #1
notes$.subscribe();   // request #2 — a new request, not a cached response
```

```html
<!-- ❌ two async pipes = two requests -->
<p>{{ (notes$ | async)?.meta?.total }}</p>
<app-notes-grid [notes]="(notes$ | async)?.data ?? []" />
```

Fix: convert once (`toSignal`/resource) or share (`shareReplay`, [note 3](03-subjects-and-multicasting.md)).

## 11. Visual explanation (marble diagram)

```text
interval(500):   --0--1--2--3--4--…
take(3):         --0--1--2|
                          └ "|" = complete, "#" = error, "-" = time passes
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Expecting an Observable to run without `subscribe` | it's lazy |
| Nested `subscribe` inside `subscribe` | flattening operators (`switchMap`…) |
| Long-lived subscriptions without teardown | `takeUntilDestroyed()`, `toSignal`, `async` |
| Assuming errors are recoverable after they happen | handle with `catchError` inside the right inner stream |
| `new Observable` for everything | creation functions cover most cases |

## 13. Debugging guide

- `tap({ next: v => console.log('[notes]', v), error: e => console.error(e), complete: () => console.log('[notes] done') })`.
- Values stop forever after one failure → an unhandled error completed the stream.
- Duplicate requests → multiple subscriptions to a cold Observable.

## 14. Best practices

- Name Observables with a `$` suffix (`online$`).
- Prefer completion-based Observables (HTTP, `take`, `first`) where possible.
- Convert to signals at the edge for templates.

## 15. Angular vs React

React has no built-in stream type; you'd use event listeners in `useEffect`, or libraries. Angular uses RxJS in the router, HttpClient and forms, so you'll meet it in any real Angular codebase.

## 16. Testing

```typescript
it('emits 3 ticks and completes', async () => {
  const values: number[] = [];
  await new Promise<void>(resolve =>
    interval(1).pipe(take(3)).subscribe({ next: v => values.push(v), complete: resolve }),
  );
  expect(values).toEqual([0, 1, 2]);
});
```

## 17. Practice questions

1. Write an Observable that emits the current time every minute and stops when the tab is hidden.
2. Why does `defer` matter for `from(fetch(...))`?
3. Draw marbles for `timer(1000, 500).pipe(take(4))`.

## 18. Interview questions

- **Q: Observable vs Promise?** Observables are lazy, can emit many values and can be cancelled; Promises are eager, resolve once and can't be cancelled.
- **Q: Cold vs hot?** Cold creates a producer per subscriber; hot shares one producer among subscribers.
- **Q: What happens after `error`?** The stream ends; no more `next` values.

## 19. Revision notes

Lazy · `next*` then `error|complete` · teardown on unsubscribe · cold (HTTP) vs hot (events/subjects) · `of/from/interval/timer/fromEvent/defer/throwError/EMPTY` · `$` suffix.

## 20. Completion checklist

- [ ] `NetworkStatus` built with `toSignal`.
- [ ] I can draw marbles for three creation functions.
- [ ] I can explain why two async pipes cause two requests.
