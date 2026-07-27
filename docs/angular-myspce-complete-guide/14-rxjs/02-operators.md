# Operators

Back to [Module 14](README.md) · Previous: [Observables](01-observables.md) · Next: [Subjects and multicasting](03-subjects-and-multicasting.md)

## 1. Learning objectives

- Use the core transformation, filtering and utility operators.
- Choose the right flattening operator every time.
- Handle errors without killing long-lived streams.
- Combine streams with `combineLatest`, `forkJoin`, `merge`, `concat`.
- Build search autocomplete and dependent requests without race conditions.

## 2. Prerequisites

[Observables](01-observables.md).

## 3. What is it?

Operators are functions that take an Observable and return a new one: `source$.pipe(op1, op2, op3)`. They never mutate the source.

## 4. Why does Angular need it?

Debounced search, cancel-on-new-input, "ignore double clicks", "retry twice", "wait for all three dashboard calls" — each is one operator.

## 5. How it works internally

`pipe` composes operators left to right. Each operator subscribes to the previous Observable when it is subscribed to, and unsubscribes from it on teardown. Flattening operators subscribe to **inner** Observables created per outer value.

## 6. Syntax and API reference

### Transform, filter, utilities

| Operator | Does | Example |
|---|---|---|
| `map(fn)` | transform each value | `map(r => r.data)` |
| `filter(pred)` | drop values | `filter(q => q.length >= 2)` |
| `tap(fn)` | side effect, value unchanged | logging |
| `scan(fn, seed)` | running accumulation | count clicks, accumulate pages |
| `take(n)` / `first()` | first n values then complete | |
| `distinctUntilChanged()` | skip repeats | same query typed twice |
| `debounceTime(ms)` | wait for silence | typing |
| `throttleTime(ms)` | at most one per window | scroll/resize |
| `startWith(v)` | emit v first | initial state |
| `finalize(fn)` | on complete/error/unsubscribe | stop spinner |
| `catchError(fn)` | replace an error with another Observable | fallback |
| `retry({ count, delay })` | resubscribe on error | transient failures |
| `timeout(ms)` | error if too slow | |
| `takeUntilDestroyed()` | complete on destroy (Angular) | cleanup |

### The four flattening operators

| Operator | When a new outer value arrives while an inner is running… | Use for | Myspce |
|---|---|---|---|
| `switchMap` | **cancel** the old inner, start the new one | latest wins | search, route param → load note, autosave latest |
| `mergeMap` | run **in parallel** | independent work | upload 5 files at once |
| `concatMap` | **queue**, run one after another in order | order matters | save habit logs in order, sequential migrations |
| `exhaustMap` | **ignore** new values until the current inner finishes | prevent duplicates | "Save" button double-click, login submit |

### Combination

| Function | Emits | Completes | Use |
|---|---|---|---|
| `forkJoin([a$, b$])` | **once**, the last value of each, when **all complete** | after emitting | parallel one-shot requests (dashboard) |
| `combineLatest([a$, b$])` | latest of each, whenever **any** emits (after all have emitted once) | when all complete | derived from multiple live streams |
| `merge(a$, b$)` | every value from either | when all complete | multiple event sources |
| `concat(a$, b$)` | all of a$, then all of b$ | when last completes | sequencing |
| `withLatestFrom(b$)` | on source emission, pairs with b$'s latest | with source | "on save, include current settings" |

With signals, many `combineLatest` uses become a `computed()`.

## 7. Smallest working example — debounced search

```typescript
// core/layout/global-search.ts
import { Component, inject, signal } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';
import { catchError, debounceTime, distinctUntilChanged, map, of, switchMap, startWith } from 'rxjs';
import { API_URL } from '../http/api-url.token';

type SearchResult =
  | { kind: 'note'; id: string; title: string }
  | { kind: 'credential'; id: string; name: string }
  | { kind: 'task'; id: string; title: string };

type SearchState = { status: 'idle' } | { status: 'loading' } | { status: 'done'; results: SearchResult[] } | { status: 'error' };

@Component({
  selector: 'app-global-search',
  template: `
    <input type="search" placeholder="Search anything…" [value]="query()" (input)="query.set($any($event.target).value)"
           class="w-full rounded-lg bg-slate-100 px-3 py-2 dark:bg-slate-800" aria-label="Search" />
    @switch (state().status) {
      @case ('loading') { <p role="status" class="text-xs">Searching…</p> }
      @case ('error') { <p role="alert" class="text-xs text-rose-500">Search failed.</p> }
      @case ('done') { <!-- render grouped results --> }
      @case ('idle') {}
    }
  `,
})
export class GlobalSearch {
  private http = inject(HttpClient);
  private api = inject(API_URL);
  protected query = signal('');

  protected state = toSignal(
    toObservable(this.query).pipe(
      map(q => q.trim()),
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(q =>
        q.length < 2
          ? of<SearchState>({ status: 'idle' })
          : this.http.get<SearchResult[]>(`${this.api}/search`, { params: { q } }).pipe(
              map(results => ({ status: 'done', results }) as SearchState),
              startWith<SearchState>({ status: 'loading' }),
              catchError(() => of<SearchState>({ status: 'error' })),   // inside switchMap: the outer stream survives
            ),
      ),
    ),
    { initialValue: { status: 'idle' } as SearchState },
  );
}
```

## 8. Step-by-step

1. `toObservable(query)` turns typing into a stream.
2. `debounceTime(300)` waits until typing pauses.
3. `distinctUntilChanged()` skips "aws" → "aws " (after trim) duplicates.
4. `switchMap` cancels the previous request when a new query arrives — **no stale results**.
5. `catchError` is **inside** the inner pipe: a failed search shows an error but the search box keeps working. Put it outside and the first error would kill the whole stream.
6. `toSignal` exposes the state to the OnPush template.

## 9. Myspce real-world example — dashboard requests and safe saves

### Parallel one-shot requests

```typescript
// features/dashboard/data-access/dashboard-store.ts
load() {
  return forkJoin({
    summary: this.api.summary(),
    today: this.api.today().pipe(catchError(() => of({ tasks: [], events: [] }))),   // one failing widget doesn't block others
    week: this.api.weekProgress().pipe(catchError(() => of(null))),
  });
}
```

### Prevent double submit

```typescript
// features/notes/pages/note-editor-page.ts
private save$ = new Subject<void>();

constructor() {
  this.save$.pipe(
    exhaustMap(() => this.api.update(this.note().id, this.draft()).pipe(
      tap(() => this.toast.success('Saved')),
      catchError(err => { this.toast.error(toMessage(err)); return EMPTY; }),
    )),
    takeUntilDestroyed(),
  ).subscribe();
}

save() { this.save$.next(); }   // clicking 5 times quickly → 1 request
```

### Dependent requests

```typescript
// load a note, then its category's other notes
this.api.get(id).pipe(
  switchMap(note => this.api.list({ category: note.category?.slug, limit: 4 }).pipe(map(related => ({ note, related })))),
);
```

(With resources in v22 you'd use `chain()` instead — [Module 08](../08-signals-and-reactivity/04-resources.md).)

### Upload several files in parallel, max 3 at a time

```typescript
from(files).pipe(mergeMap(f => this.uploads.upload(f, folderId), 3));   // concurrency limit
```

## 10. Advanced example — race conditions explained

```text
User types "ang" then "angular" quickly.
mergeMap:  request("ang") ……………… response("ang") arrives LAST → UI shows results for "ang" ❌
switchMap: request("ang") ✕ cancelled; request("angular") → UI shows "angular" ✅
```

Any time "the latest input decides the result", use `switchMap` (or resources, which do this for you).

## 11. Visual explanation (marbles)

```text
outer clicks: a, then b while a's inner is running, then c while b's inner is running
each inner emits x1, x2, x3 over time

switchMap:   a1 a2 b1 c1 c2 c3          (a cancelled by b, b cancelled by c)
mergeMap:    a1 a2 b1 a3 b2 c1 b3 c2 c3 (all run; outputs interleave)
concatMap:   a1 a2 a3 b1 b2 b3 c1 c2 c3 (queued, in order)
exhaustMap:  a1 a2 a3                    (b and c ignored while a runs)

debounceTime(3):
keys:         -a-b-c------d-------
out:          ---------c-------d--
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `catchError` at the outer level of a long-lived stream | put it inside the inner Observable |
| `mergeMap` for search | `switchMap` |
| `switchMap` for "save every change" | `concatMap` (or `switchMap` only if you want latest-only) |
| `forkJoin` with a never-completing Observable | it never emits — use HTTP or add `take(1)` |
| `combineLatest` where one source never emits | nothing emits — add `startWith` |
| Nested subscribes | flattening operators |

## 13. Debugging guide

- `tap` with labels at each step.
- "Nothing happens" in `forkJoin` → some input doesn't complete.
- "Old results flash" → wrong flattening operator.
- RxJS operator decision tree: https://rxjs.dev/operator-decision-tree

## 14. Best practices

- Decide the flattening strategy deliberately; write a comment when it's not obvious.
- Handle errors per inner request in long-lived streams.
- Prefer resources for "load data from signals"; keep RxJS for event logic.

## 15. Angular vs React

React has no operators; debouncing and cancellation are manual (`AbortController`, custom hooks) or via libraries. Angular code uses RxJS heavily in forms, router and HTTP.

## 16. Testing

```typescript
it('debounces and cancels stale searches', async () => {
  vi.useFakeTimers();
  TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting(), { provide: API_URL, useValue: '/api/v1' }] });
  const fixture = TestBed.createComponent(GlobalSearch);
  const cmp = fixture.componentInstance as any;
  const ctrl = TestBed.inject(HttpTestingController);

  cmp.query.set('an'); TestBed.tick(); await vi.advanceTimersByTimeAsync(100);
  cmp.query.set('angular'); TestBed.tick(); await vi.advanceTimersByTimeAsync(300);

  ctrl.expectNone(r => r.params.get('q') === 'an');           // debounced away
  ctrl.expectOne(r => r.params.get('q') === 'angular').flush([]);
  expect(cmp.state()).toEqual({ status: 'done', results: [] });
  vi.useRealTimers();
});
```

## 17. Practice questions

1. Pick the operator: (a) tasks toggled quickly, each must reach the server in order; (b) "Refresh" clicked repeatedly; (c) open note B while note A is loading; (d) upload 10 files.
2. Implement polling of `/notifications/unread-count` every 60 s that pauses when the tab is hidden.
3. Rewrite a `combineLatest` of two `BehaviorSubject`s as a `computed`.

## 18. Interview questions

- **Q: `switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`?** Cancel previous / run in parallel / queue in order / ignore new while busy.
- **Q: `forkJoin` vs `combineLatest`?** `forkJoin` emits once when all complete; `combineLatest` emits on every change once all have emitted.
- **Q: Where do you put `catchError` in a search stream?** Inside the `switchMap` inner Observable, so one failed request doesn't terminate the stream.

## 19. Revision notes

`map filter tap scan` · `debounceTime distinctUntilChanged startWith` · switch (latest) / merge (parallel) / concat (ordered) / exhaust (ignore) · `catchError` inside inner · `retry` · `finalize` · `forkJoin` (once, all complete) vs `combineLatest` (live).

## 20. Completion checklist

- [ ] Global search works with debounce and cancellation.
- [ ] Save button uses `exhaustMap`.
- [ ] I can draw marbles for all four flattening operators.
