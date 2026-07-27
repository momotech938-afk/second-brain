# Service, Store and HTTP Tests

Back to [Module 19](README.md) · Previous: [Component tests](02-component-tests.md) · Next: [Forms and routing tests](04-forms-routing-tests.md)

## 1. Learning objectives

- Test services with fake dependencies.
- Test stores (signals, resources, optimistic updates).
- Test API services and interceptors with `HttpTestingController`.
- Test async code: Promises, Observables, timers.

## 2. Services with fake dependencies

```typescript
// core/storage/local-store.spec.ts
describe('LocalStore', () => {
  beforeEach(() => localStorage.clear());

  it('returns the fallback when nothing is stored', () => {
    expect(TestBed.inject(LocalStore).get('notes.view', 'grid')).toBe('grid');
  });

  it('round-trips JSON values with a prefix', () => {
    const s = TestBed.inject(LocalStore);
    s.set('notes.view', 'list');
    expect(localStorage.getItem('myspce.notes.view')).toBe('"list"');
    expect(s.get('notes.view', 'grid')).toBe('list');
  });
});
```

Override dependencies with providers:

```typescript
TestBed.configureTestingModule({
  providers: [
    { provide: NotesApi, useValue: { list: vi.fn(() => of(page)), update: vi.fn() } satisfies Partial<NotesApi> },
    { provide: NOW, useValue: () => new Date('2026-10-28T21:42:00+05:30') },
  ],
});
```

## 3. Stores with resources

Resources start loading after Angular's scheduler runs. Flush with `TestBed.tick()` and wait with `whenStable()`:

```typescript
it('exposes grouped tasks after loading', async () => {
  TestBed.configureTestingModule({
    providers: [
      { provide: TasksApi, useValue: { list: () => of({ data: [task()], meta: { total: 1, page: 1, limit: 200 } }) } },
      { provide: ToastService, useValue: { error: vi.fn(), success: vi.fn() } },
    ],
  });
  const store = TestBed.inject(TasksStore);
  TestBed.tick();
  await TestBed.inject(ApplicationRef).whenStable();
  expect(store.grouped().today).toHaveLength(1);
});
```

For HTTP-backed resources, combine with `HttpTestingController`: tick → `expectOne` → `flush` → `whenStable` → assert.

## 4. `HttpTestingController` in depth

```typescript
describe('NotesApi', () => {
  let api: NotesApi;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: provideTestDefaults() });
    api = TestBed.inject(NotesApi);
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());

  it('PATCHes the pin endpoint', () => {
    api.pin('n1', true).subscribe();
    const req = http.expectOne({ method: 'PATCH', url: '/api/v1/notes/n1/pin' });
    expect(req.request.body).toEqual({ pinned: true });
    req.flush({ ...MOCK_NOTES[0], pinned: true });
  });

  it('propagates a 404 as HttpErrorResponse', () => {
    let status = 0;
    api.get('missing').subscribe({ error: (e: HttpErrorResponse) => (status = e.status) });
    http.expectOne('/api/v1/notes/missing').flush({ statusCode: 404, message: 'Not found' }, { status: 404, statusText: 'Not Found' });
    expect(status).toBe(404);
  });
});
```

Matching options: exact URL string (including query string for `expectOne('/x?a=1')`), `{ method, url }` (URL without params), or a predicate `r => r.url === … && r.params.get('page') === '2'`.

## 5. Interceptors

Register the interceptor under test in `provideHttpClient(withInterceptors([...]))` before `provideHttpClientTesting()` and assert on the requests the testing backend receives (see the auth and retry interceptor tests in [Module 13](../13-http-and-api-integration/02-interceptors-and-auth.md#16-testing)).

Testing the single-flight refresh:

```typescript
it('three concurrent 401s trigger one refresh', () => {
  TestBed.configureTestingModule({ providers: [provideHttpClient(withInterceptors([authInterceptor])), provideHttpClientTesting(), provideRouter([]), { provide: API_URL, useValue: '/api/v1' }] });
  const http = TestBed.inject(HttpClient), ctrl = TestBed.inject(HttpTestingController);

  ['a', 'b', 'c'].forEach(p => http.get(`/api/v1/${p}`).subscribe());
  ['a', 'b', 'c'].forEach(p => ctrl.expectOne(`/api/v1/${p}`).flush(null, { status: 401, statusText: 'Unauthorized' }));

  const refreshes = ctrl.match('/api/v1/auth/refresh');
  expect(refreshes).toHaveLength(1);
  refreshes[0]!.flush({ accessToken: 'new', user: { id: 'u1', name: 'P', email: 'p@x.dev', role: 'super_admin', avatarUrl: null, bio: null } });

  ['a', 'b', 'c'].forEach(p => expect(ctrl.expectOne(`/api/v1/${p}`).request.headers.get('Authorization')).toBe('Bearer new'));
});
```

## 6. Async patterns

| Code under test | Test with |
|---|---|
| `async` method | `await` it; `await expect(p).rejects.toThrow()` |
| Observable that completes | subscribe and assert, or `await firstValueFrom(obs$)` / `lastValueFrom` |
| Debounce/timers | `vi.useFakeTimers()` + `await vi.advanceTimersByTimeAsync(ms)` |
| Time-based RxJS logic | `TestScheduler.run()` marbles ([Module 14](../14-rxjs/04-interop-and-testing.md)) |
| Effects | `TestBed.tick()` |
| Eventually-true conditions | `await vi.waitFor(() => expect(...))` |

Always restore real timers (`vi.useRealTimers()` in `afterEach`) to avoid leaking into other tests.

## 7. Myspce example — credential reveal flow (store + HTTP)

```typescript
it('reveal requests re-auth on 403 REAUTH_REQUIRED and retries', async () => {
  const ask = vi.fn().mockResolvedValue(true);
  TestBed.configureTestingModule({
    providers: [
      provideHttpClient(withInterceptors([reauthInterceptor])), provideHttpClientTesting(),
      { provide: API_URL, useValue: '/api/v1' }, { provide: ReauthDialog, useValue: { ask } },
    ],
  });
  const api = TestBed.inject(CredentialsApi), ctrl = TestBed.inject(HttpTestingController);

  const result = firstValueFrom(api.reveal('c1'));
  ctrl.expectOne('/api/v1/credentials/c1/reveal').flush({ message: 'REAUTH_REQUIRED' }, { status: 403, statusText: 'Forbidden' });
  await vi.waitFor(() => expect(ask).toHaveBeenCalled());
  ctrl.expectOne('/api/v1/credentials/c1/reveal').flush({ secret: 'S3cret!' });
  await expect(result).resolves.toEqual({ secret: 'S3cret!' });
});
```

## 8. Common mistakes

| Mistake | Fix |
|---|---|
| No `http.verify()` | unexpected requests go unnoticed |
| `expectOne` with a URL including params when the request uses `params` | match `{ url }` or a predicate |
| Asserting before `flush` | the subscriber hasn't received anything yet |
| Real timers in debounce tests | fake timers |
| Shared mutable fixtures modified by tests | clone (`structuredClone`) or build per test |

## 9. Practice

1. Test `NotesStore.toggleFavorite` rollback with `HttpTestingController` returning 500.
2. Test `loadingInterceptor` busy state across success and error.
3. Test `IdleLock` locks after the timeout and resets on activity (fake timers).

## 10. Interview questions

- **Q: What does `http.verify()` check?** That no requests are outstanding that the test didn't expect or flush.
- **Q: How do you test an interceptor?** Register it with `withInterceptors` alongside `provideHttpClientTesting()`, make requests via `HttpClient`, and assert/flush with `HttpTestingController`.

## 11. Completion checklist

- [ ] Every API service has request-shape tests.
- [ ] Auth interceptor tests include the concurrent refresh case.
- [ ] Store tests cover optimistic rollback.
