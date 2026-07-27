# Signals vs RxJS, Patterns, Debugging and Testing

Back to [Module 08](README.md) · Previous: [Resources](04-resources.md) · Next module: [09 Lifecycle and change detection](../09-lifecycle-and-change-detection/README.md)

## 1. Learning objectives

- Choose signals, RxJS or resources for a given problem.
- Convert between them with `toSignal`, `toObservable`, `outputFromObservable`.
- Implement search, filter, sort and pagination with signals.
- Recognise reactive design mistakes and debug them.
- Test signal-based code.

## 2. Prerequisites

Notes 1–4 of this module. Basic Observables ([Module 14](../14-rxjs/01-observables.md)).

## 3. Signals vs Observables

| | Signal | Observable |
|---|---|---|
| Holds a current value | always | not necessarily |
| Read | synchronously, `s()` | subscribe |
| Time / events | no concept of time | streams over time |
| Cancellation | n/a | unsubscribe |
| Operators | `computed`, `linkedSignal` | 100+ (`debounceTime`, `switchMap`, `retry`…) |
| Template | read directly | `async` pipe or `toSignal` |
| Best for | **state** | **events and async flows** |

**Rule:** state → signals. Events over time (typing, WebSocket messages, polling, retries, combining multiple async sources with timing rules) → RxJS. Async data loading tied to state → resources (they bridge both).

## 4. Interop

```typescript
import { toSignal, toObservable, outputFromObservable, takeUntilDestroyed } from '@angular/core/rxjs-interop';

// Observable → signal (subscribes immediately; unsubscribes with the injection context)
online = toSignal(merge(fromEvent(window, 'online'), fromEvent(window, 'offline')).pipe(map(() => navigator.onLine)),
                  { initialValue: navigator.onLine });

// signal → Observable (emits asynchronously after changes, via an effect)
query$ = toObservable(this.query);

// BehaviorSubject-like sources: requireSync avoids `undefined` in the type
user = toSignal(this.auth.user$, { requireSync: true });
```

| Pitfall | Explanation |
|---|---|
| `toSignal` outside an injection context | `NG0203`; pass `{ injector }` or call in a field |
| Forgetting `initialValue` | type becomes `T \| undefined` |
| `toSignal` on an HTTP call that should run on demand | it subscribes immediately and once — use a resource or call the API in a method |
| `toObservable` timing | emits on the next effect run, not synchronously; multiple quick sets → one emission |
| Errors | an erroring Observable makes the signal throw on read; handle with `catchError` |

## 5. Pattern — search + filter + sort + client pagination

```typescript
// features/credentials/pages/credentials-page.ts
type SortKey = 'name' | '-lastViewedAt';

export class CredentialsPage {
  private api = inject(API_URL);
  protected credentials = httpResource<Paginated<CredentialSummary>>(() => `${this.api}/credentials?limit=500`);

  protected query = signal('');
  protected category = signal('all');
  protected sort = signal<SortKey>('name');
  protected readonly pageSize = 20;
  protected page = linkedSignal(() => { this.query(); this.category(); this.sort(); return 1; });

  private all = computed(() => (this.credentials.hasValue() ? this.credentials.value().data : []));

  protected filtered = computed(() => {
    const q = this.query().trim().toLowerCase();
    const cat = this.category();
    const list = this.all().filter(c =>
      (cat === 'all' || c.category?.slug === cat) &&
      (!q || c.name.toLowerCase().includes(q) || (c.url ?? '').toLowerCase().includes(q) || c.username.toLowerCase().includes(q)),
    );
    return this.sort() === 'name'
      ? list.toSorted((a, b) => a.name.localeCompare(b.name))
      : list.toSorted((a, b) => (b.lastViewedAt ?? '').localeCompare(a.lastViewedAt ?? ''));
  });

  protected pageCount = computed(() => Math.max(1, Math.ceil(this.filtered().length / this.pageSize)));
  protected pageItems = computed(() => this.filtered().slice((this.page() - 1) * this.pageSize, this.page() * this.pageSize));
}
```

Client-side filtering is right for a single user's ~30–500 credentials. Server-side filtering (params in the resource) is right for large or growing datasets like notes and files.

## 6. Pattern — loading/error/success derived once

```typescript
protected view = computed(() => {
  const r = this.credentials;
  if (r.isLoading() && !r.hasValue()) return { kind: 'loading' } as const;
  if (r.error()) return { kind: 'error', message: toMessage(r.error()) } as const;
  if (!this.filtered().length) return { kind: 'empty', filtered: !!this.query() } as const;
  return { kind: 'ready', items: this.pageItems() } as const;
});
```

```html
@let v = view();
@switch (v.kind) {
  @case ('loading') { <app-skeleton-rows [count]="6" /> }
  @case ('error') { <app-error-state [message]="v.message" (retry)="credentials.reload()" /> }
  @case ('empty') { <app-empty-state [title]="v.filtered ? 'No matches' : 'Your vault is empty'" /> }
  @case ('ready') { @for (c of v.items; track c.id) { <app-credential-row [credential]="c" /> } }
  @default never;
}
```

The discriminated union narrows `v` inside each case.

## 7. Common reactive design mistakes

| Mistake | Better |
|---|---|
| Syncing signals with effects | `computed` / `linkedSignal` |
| Fetching in effects | resources |
| Wrapping every Observable in `toSignal` "to be modern" | keep RxJS for streams; convert at the edge (template or store) |
| Signals inside signals (`signal(signal(0))`) | flatten state |
| One giant `state = signal({...everything})` | separate signals; `computed` for combinations |
| Calling `set` inside `computed` | not allowed (`NG0600`) |
| Mutable objects in signals | immutable updates |

## 8. Debugging

- **Angular DevTools → signal graph:** see producers, consumers, and which `computed` re-ran.
- `debugName` on signals/computeds/effects makes the graph readable.
- Temporary `effect(() => console.log('query', this.query()))` to trace changes (remove after).
- A value "lags one step behind" → you used `toObservable` or an effect where a `computed` belongs.

## 9. Testing signal code

```typescript
describe('CredentialsPage filtering', () => {
  let page: CredentialsPage;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting(), { provide: API_URL, useValue: '/api/v1' }],
    });
    page = TestBed.createComponent(CredentialsPage).componentInstance;
    TestBed.tick();
    TestBed.inject(HttpTestingController).expectOne('/api/v1/credentials?limit=500').flush({
      data: [
        { id: '1', name: 'GitHub', url: 'https://github.com', username: 'pranjalgoyal', icon: null, category: null, lastViewedAt: null },
        { id: '2', name: 'AWS Console', url: 'https://aws.amazon.com', username: 'pranjal', icon: null, category: null, lastViewedAt: null },
      ],
      meta: { total: 2, page: 1, limit: 500 },
    });
  });

  it('filters by query across name, url and username', async () => {
    await TestBed.inject(ApplicationRef).whenStable();
    (page as any).query.set('aws');
    expect((page as any).filtered().map((c: CredentialSummary) => c.name)).toEqual(['AWS Console']);
  });

  it('resets page when the query changes', () => {
    (page as any).page.set(2);
    (page as any).query.set('git');
    expect((page as any).page()).toBe(1);
  });
});
```

Tips:

- Signals and computeds can be asserted synchronously.
- Effects and resources need `TestBed.tick()` (or `await fixture.whenStable()`).
- `as any` to reach `protected` members is acceptable in tests; alternatively test through the DOM.

## 10. Practice questions

1. Which tool for: theme mode, global search typing, WebSocket notifications, current user, dashboard summary, a countdown timer?
2. Convert a `BehaviorSubject`-based store to signals.
3. Implement server-side pagination for Notes by moving `page` into `httpResource` params.

## 11. Interview questions

- **Q: When would you still use RxJS in a signals-first app?** For event streams with timing or cancellation semantics: debouncing, polling, retries, websockets, combining async sources.
- **Q: What does `toSignal` do and what's a pitfall?** Subscribes to an Observable and exposes its latest value as a signal; it subscribes immediately and lives with the injection context, so it's wrong for on-demand requests.
- **Q: Signals vs React state?** Signals live anywhere (not only components), updates don't re-run components, dependencies are tracked automatically.

## 12. Revision notes

State → signals · streams → RxJS · async reads → resources · interop at the edges · derive once with `computed` · discriminated union for view state · `TestBed.tick()` for effects/resources.

## 13. Completion checklist

- [ ] Credentials search/filter/sort/paginate with signals works and is tested.
- [ ] I can justify signals vs RxJS for each Myspce feature.
- [ ] I've debugged a signal issue with DevTools.
