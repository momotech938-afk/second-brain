# Subjects and Multicasting

Back to [Module 14](README.md) · Previous: [Operators](02-operators.md) · Next: [Interop and testing](04-interop-and-testing.md)

## 1. Learning objectives

- Use `Subject`, `BehaviorSubject` and `ReplaySubject` and know their differences.
- Share one expensive source among many subscribers with `shareReplay`.
- Decide when a subject should be a signal instead.

## 2. Prerequisites

[Operators](02-operators.md).

## 3. What is it?

- A **Subject** is both an Observable (you can subscribe) and an Observer (you can call `next`). It's **hot**: all subscribers share the same emissions.
- **Multicasting** means one producer, many consumers.

## 4. Why does Angular need it?

To turn imperative events into streams (`save$.next()`), to share one HTTP response among several consumers, and in older code, to hold state (`BehaviorSubject`) — a role signals now play better.

## 5. How it works

| Type | Initial value | Late subscriber receives | Typical use |
|---|---|---|---|
| `Subject<T>` | none | only future values | events: "save clicked", "refresh requested" |
| `BehaviorSubject<T>(init)` | required | the **current** value, then future | legacy state containers |
| `ReplaySubject<T>(n)` | none | the last **n** values, then future | caches, late subscribers needing history |
| `AsyncSubject<T>` | none | the last value on complete | rare |

## 6. Syntax and API reference

```typescript
const refresh$ = new Subject<void>();
refresh$.next();
refresh$.complete();

const user$ = new BehaviorSubject<User | null>(null);
user$.value;           // synchronous current value
user$.next(u);

// Expose read-only
readonly changes$ = this.subject.asObservable();

// Share a cold source
const summary$ = this.api.summary().pipe(shareReplay({ bufferSize: 1, refCount: true }));
```

`shareReplay({ bufferSize: 1, refCount: true })`:

- first subscriber triggers one request; later subscribers get the cached last value;
- `refCount: true` unsubscribes from the source when the last subscriber leaves (avoids leaks for infinite sources);
- `refCount: false` keeps the cache forever (good for app-wide config that never changes, dangerous for streams).

`share()` multicasts without replaying (late subscribers miss past values).

## 7. Smallest working example — a refresh trigger

```typescript
// features/notes/data-access/notes-feed.ts
@Service()
export class NotesFeed {
  private api = inject(NotesApi);
  private refresh$ = new Subject<void>();

  readonly notes$ = this.refresh$.pipe(
    startWith(undefined),
    switchMap(() => this.api.list()),
    shareReplay({ bufferSize: 1, refCount: true }),
  );

  refresh() { this.refresh$.next(); }
}
```

## 8. Step-by-step

1. `startWith` triggers the first load on subscription.
2. Each `refresh()` re-fetches (cancelling a pending one).
3. `shareReplay` means two components using `notes$` share one request and one cached result.

## 9. Myspce real-world example — `BehaviorSubject` store → signal store

You'll see this in older Angular code (and in many company apps):

```typescript
// legacy
@Injectable({ providedIn: 'root' })
export class TasksStoreLegacy {
  private readonly _tasks = new BehaviorSubject<Task[]>([]);
  readonly tasks$ = this._tasks.asObservable();
  readonly today$ = this.tasks$.pipe(map(ts => ts.filter(t => isToday(t.dueAt))));
  toggle(id: string) {
    this._tasks.next(this._tasks.value.map(t => (t.id === id ? { ...t, status: t.status === 'done' ? 'todo' : 'done' } : t)));
  }
}
```

```typescript
// modern
@Service()
export class TasksStore {
  private readonly _tasks = signal<Task[]>([]);
  readonly tasks = this._tasks.asReadonly();
  readonly today = computed(() => this._tasks().filter(t => isToday(t.dueAt)));
  toggle(id: string) {
    this._tasks.update(ts => ts.map(t => (t.id === id ? { ...t, status: t.status === 'done' ? 'todo' : 'done' } : t)));
  }
}
```

Same API shape; no `async` pipes, no subscriptions, glitch-free derived state.

**Keep subjects for events** (`save$`, `refresh$`), not for state.

## 10. Advanced example — WebSocket notifications (v1.1 idea)

```typescript
// core/notifications/notifications-socket.ts (proposed; the backend doesn't define a socket yet)
import { webSocket } from 'rxjs/webSocket';

@Service()
export class NotificationsSocket {
  private socket$ = webSocket<NotificationMessage>({ url: 'wss://myspace.example/ws', openObserver: { next: () => console.info('ws open') } });
  readonly messages$ = this.socket$.pipe(
    retry({ delay: (_, i) => timer(Math.min(30_000, 2 ** i * 1000)) }),   // reconnect with backoff
    share(),
  );
  readonly unreadCount = toSignal(this.messages$.pipe(scan(n => n + 1, 0)), { initialValue: 0 });
}
```

Here RxJS shines: reconnection, backoff and sharing are a few operators.

## 11. Visual explanation

```text
Subject:          ---a---b---c---
subscriber 1 (t0) ---a---b---c---
subscriber 2 (t2)        b---c---    (missed a)

BehaviorSubject(0):
subscriber 2 (t2) a──►   a b c       (gets current 'a' immediately)

shareReplay(1):   source requested once; late subscribers get the last value
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Exposing a `Subject` publicly (anyone can `next`) | expose `asObservable()` |
| `BehaviorSubject` for new state | signals |
| `shareReplay(1)` (positional) on infinite streams | `shareReplay({ bufferSize: 1, refCount: true })` |
| Forgetting `complete()` on subjects in services that are destroyed | complete in `DestroyRef.onDestroy` |
| `subject.value` reads scattered everywhere | signals or a single selector |

## 13. Debugging guide

- "Second component gets nothing" → plain `Subject` and it subscribed late; use `BehaviorSubject`/`ReplaySubject`/`shareReplay` (or a signal).
- "Request runs twice" → missing `shareReplay`.
- "Request never re-runs" → `shareReplay` with `refCount: false` cached it forever.

## 14. Best practices

- Subjects for events, signals for state.
- Always multicast expensive sources consumed in more than one place.
- Prefer `refCount: true` unless you truly want a permanent cache.

## 15. Angular vs React

`BehaviorSubject` stores ≈ Zustand/Redux stores; signals ≈ a built-in, fine-grained store primitive. Subjects as event buses ≈ custom event emitters.

## 16. Testing

```typescript
it('shares one request between subscribers', () => {
  const list = vi.fn(() => of({ data: [], meta: { total: 0, page: 1, limit: 20 } }));
  TestBed.configureTestingModule({ providers: [{ provide: NotesApi, useValue: { list } }] });
  const feed = TestBed.inject(NotesFeed);
  feed.notes$.subscribe();
  feed.notes$.subscribe();
  expect(list).toHaveBeenCalledTimes(1);
});
```

## 17. Practice questions

1. Convert a `BehaviorSubject<Theme>` service to a signal service.
2. When would `ReplaySubject(5)` be useful in Myspce?
3. Why can `shareReplay` with `refCount: false` leak?

## 18. Interview questions

- **Q: `Subject` vs `BehaviorSubject`?** `BehaviorSubject` has a current value and gives it to new subscribers; `Subject` only emits future values.
- **Q: What does `shareReplay` do?** Multicasts a source and replays the last N values to late subscribers.
- **Q: Why prefer signals over `BehaviorSubject` for state?** Synchronous reads, automatic dependency tracking, no subscriptions, direct template use with OnPush.

## 19. Revision notes

`Subject` (events) · `BehaviorSubject` (legacy state) · `ReplaySubject` (history) · `asObservable()` · `shareReplay({ bufferSize: 1, refCount: true })` · state → signals.

## 20. Completion checklist

- [ ] `NotesFeed` with refresh + sharing.
- [ ] A legacy `BehaviorSubject` store migrated to signals.
