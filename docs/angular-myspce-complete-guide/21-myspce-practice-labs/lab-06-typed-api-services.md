# Lab 6 — Typed API Services and a Mock Backend

Back to [Labs](README.md) · Solution: [lab-06-solution.md](solutions/lab-06-solution.md) · Previous: [Lab 5](lab-05-signal-ui-state.md) · Next: [Lab 7](lab-07-ui-states.md)

## 1. Learning objective

Replace in-memory mock arrays with a real HTTP layer: typed models, an `API_URL` token, a `NotesApi` service, and a `mockBackendInterceptor` that pretends to be the planned NestJS API, so swapping in the real backend later is one provider change.

## 2. Prerequisites

[Module 07 note 4](../07-services-and-dependency-injection/04-myspce-api-layer.md), [Module 13 notes 1–2](../13-http-and-api-integration/README.md), [Lab 5](lab-05-signal-ui-state.md).

## 3. Problem statement

The backend doesn't exist yet, but the frontend must speak its **planned** contract (`/api/v1`, `{ data, meta }` lists, ULID ids — see [project context](../MYSPCE_PROJECT_CONTEXT.md)). The mock must be removable without touching feature code.

> The endpoints below are the *planned* contract from the project context, not a finished API. If the backend changes, update `core/models` and the mock together.

## 4. Expected behaviour

- `GET /api/v1/notes?category=&q=&page=&limit=` returns `{ data: Note[], meta: { total, page, limit } }` after ~300 ms.
- `POST /notes`, `GET/PATCH/DELETE /notes/:id` and `PATCH /notes/:id/pin` work against an in-memory array.
- A `?fail=1` query or a note titled "boom" returns a 500 (to test Lab 7).
- The interceptor is only registered when `environment.useMockApi` is `true`.

## 5. Relevant concepts

`HttpClient` generics, `HttpParams`, `InjectionToken`, functional interceptors, `HttpResponse`/`HttpErrorResponse`, `delay`, environment files.

## 6. Step-by-step implementation

1. In `core/models/`, copy `Paginated<T>`, `Category`, `Note` from the project context. Add `NoteDetail` (with `content`) as a **marked assumption**, plus `CreateNoteDto`/`UpdateNoteDto`.
2. Add `API_URL = new InjectionToken<string>('API_URL')` and provide `environment.apiUrl` (`/api/v1`).
3. Write `NotesApi` (`@Service()`) with `list(query)`, `get(id)`, `create(dto)`, `update(id, dto)`, `remove(id)`. Build `HttpParams` only from defined values.
4. Write `mockBackendInterceptor`:
   - only handles URLs starting with the API URL
   - routes by method + path regex
   - returns `of(new HttpResponse(...)).pipe(delay(300))` or `throwError(() => new HttpErrorResponse(...)).pipe(...)`
   - seeds from `MOCK_NOTES`
5. Register conditionally: `provideHttpClient(withInterceptors([...(environment.useMockApi ? [mockBackendInterceptor] : [])]))`.
6. Switch `NotesStore` to load via `NotesApi` using `rxResource` keyed on `{ category, query, page }`.
7. Watch the Network tab: no `/api/v1` requests leave the browser while the mock is on.

## 7. Complete working code

Models, `NotesApi`, `mockBackendInterceptor`, providers, and the updated `NotesStore` are in the [solution](solutions/lab-06-solution.md).

## 8. Important lines explained

- `this.http.get<Paginated<Note>>(url, { params })` — the generic is a promise to the compiler, not a runtime check; the model files are the single source of truth.
- `if (!req.url.startsWith(apiUrl)) return next(req);` — the mock never touches other requests (fonts, i18n files).
- `rxResource({ params: () => ({...}), stream: ({ params }) => api.list(params) })` — refetches whenever a param signal changes and cancels the previous request.
- `...(env.useMockApi ? [mock] : [])` — removing the mock is a config flag, not a code edit.

## 9. Expected output

Notes page loads after a brief delay; the browser Network tab shows **no** `/api/v1/notes` request, because the interceptor answers before anything leaves the app. Add a temporary `console.debug('[mock]', req.method, req.urlWithParams)` in the interceptor to see `GET /api/v1/notes?page=1&limit=8`.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Mock never runs | interceptor not in `withInterceptors`, or URL prefix mismatch (`/api/v1` vs `api/v1`) |
| `params` contain `undefined` strings | build `HttpParams` only from defined values |
| Store keeps old list after create | `resource.reload()` or update the local value after the mutation |
| Same request fires twice | two subscriptions to a cold observable; use the resource, not manual subscribes |
| Secrets in `environment.ts` | never — only public URLs and flags go there |

## 11. Test cases

```typescript
describe('NotesApi', () => {
  it('sends filters as query params', () => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting(), { provide: API_URL, useValue: '/api/v1' }],
    });
    const api = TestBed.inject(NotesApi);
    const ctrl = TestBed.inject(HttpTestingController);

    api.list({ category: 'work', q: '', page: 2, limit: 8 }).subscribe();
    const req = ctrl.expectOne(r => r.url === '/api/v1/notes');
    expect(req.request.params.get('category')).toBe('work');
    expect(req.request.params.has('q')).toBe(false);
    expect(req.request.params.get('page')).toBe('2');
    req.flush({ data: [], meta: { page: 2, limit: 8, total: 0 } });
    ctrl.verify();
  });
});
```

## 12. Independent practice task

Create `TasksApi` (list with the planned `view` filter, create, toggle via `PATCH /tasks/:id/toggle`, delete) and extend the mock with `/api/v1/tasks` routes. Then use the generic `createCrudApi<T, Q, C, U>(resource)` helper from [Module 07 note 4](../07-services-and-dependency-injection/04-myspce-api-layer.md) to remove duplication between `NotesApi` and `TasksApi`.

## 13. Solution

See [solutions/lab-06-solution.md](solutions/lab-06-solution.md).

## 14. Completion checklist

- [ ] Models match the project context; the one assumption (`content`) is marked and logged.
- [ ] `NotesApi` + mock interceptor working; mock is config-switchable.
- [ ] `NotesStore` loads via `rxResource`.
- [ ] `TasksApi` + mock routes.
- [ ] Tests pass; tagged `lab-06`.
