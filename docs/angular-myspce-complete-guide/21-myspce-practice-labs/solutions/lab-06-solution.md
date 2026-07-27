# Lab 6 — Solution

Back to [Lab 6](../lab-06-typed-api-services.md) · [Labs index](../README.md)

## Models

```typescript
// core/models/api.ts
export interface PageMeta { page: number; limit: number; total: number; }
export interface Paginated<T> { data: T[]; meta: PageMeta; }
export interface ApiErrorBody { statusCode: number; message: string | string[]; path: string; requestId: string; timestamp: string; }
```

```typescript
// core/models/index.ts  (proposed shapes — copied from MYSPCE_PROJECT_CONTEXT.md §4, not invented here)
export type Ulid = string;
export type CategoryModule = 'note' | 'credential' | 'task' | 'memory' | 'file' | 'event';
export interface Category { id: Ulid; module: CategoryModule; name: string; slug: string; color: string; }
export interface Note {
  id: Ulid; title: string; preview: string; color: string | null; pinned: boolean; isFavorite: boolean;
  category: Category | null; tags: string[]; createdAt: string; updatedAt: string;
}
```

The list endpoint returns `Note` (with `preview`). The editor needs the full body. The context does not define that field yet, so the lab adds **one clearly marked assumption** in the frontend only:

```typescript
// core/models/note-detail.ts — ASSUMPTION: GET /notes/:id adds a `content` field (Tiptap JSON per A5, string here for simplicity)
import { Note, Ulid } from './index';

export interface NoteDetail extends Note { content: string; }

export interface CreateNoteDto { title: string; content: string; categoryId?: Ulid | null; color?: string | null; tags?: string[]; }
export type UpdateNoteDto = Partial<CreateNoteDto> & { isFavorite?: boolean };
export interface NotesQuery { category?: string; q?: string; page: number; limit: number; }
```

This is already listed as an open question in [MYSPCE_PROJECT_CONTEXT.md §7](../../MYSPCE_PROJECT_CONTEXT.md#7-unresolved-questions), so the backend design answers it.

## Token and environment

```typescript
// core/http/api-url.token.ts
import { InjectionToken } from '@angular/core';
export const API_URL = new InjectionToken<string>('API_URL');
```

```typescript
// environments/environment.ts  (public values only — never secrets)
export const environment = { production: false, apiUrl: '/api/v1', useMockApi: true };
```

## `NotesApi`

```typescript
// features/notes/data-access/notes-api.ts
import { Service, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { API_URL } from '../../../core/http/api-url.token';
import { Paginated } from '../../../core/models/api';
import { Note } from '../../../core/models';
import { CreateNoteDto, NoteDetail, NotesQuery, UpdateNoteDto } from '../../../core/models/note-detail';

@Service()
export class NotesApi {
  private http = inject(HttpClient);
  private base = `${inject(API_URL)}/notes`;

  list(q: NotesQuery) {
    let params = new HttpParams().set('page', q.page).set('limit', q.limit);
    if (q.category && q.category !== 'all') params = params.set('category', q.category);
    if (q.q?.trim()) params = params.set('q', q.q.trim());
    return this.http.get<Paginated<Note>>(this.base, { params });
  }
  get(id: string) { return this.http.get<NoteDetail>(`${this.base}/${id}`); }
  create(dto: CreateNoteDto) { return this.http.post<NoteDetail>(this.base, dto); }
  update(id: string, dto: UpdateNoteDto) { return this.http.patch<NoteDetail>(`${this.base}/${id}`, dto); }
  togglePin(id: string) { return this.http.patch<Note>(`${this.base}/${id}/pin`, {}); }   // planned endpoint
  remove(id: string) { return this.http.delete<void>(`${this.base}/${id}`); }
}
```

## `mockBackendInterceptor`

```typescript
// core/http/mock-backend.interceptor.ts  (dev only — a fuller version of the Module 07 sketch)
import { HttpErrorResponse, HttpInterceptorFn, HttpRequest, HttpResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Observable, delay, dematerialize, materialize, of, throwError } from 'rxjs';
import { API_URL } from './api-url.token';
import { MOCK_NOTES } from '../../../testing/mock-data';
import type { NoteDetail } from '../models/note-detail';

let notes: NoteDetail[] = MOCK_NOTES.map(n => ({ ...structuredClone(n), content: n.preview }));   // MOCK_NOTES: Note[] (Module 19)
const LATENCY = 300;

const ok = <T>(body: T, status = 200) => of(new HttpResponse({ status, body })).pipe(delay(LATENCY));
const fail = (status: number, message: string, url: string): Observable<never> =>
  throwError(() => new HttpErrorResponse({
    status, url,
    error: { statusCode: status, message, path: url, requestId: crypto.randomUUID(), timestamp: new Date().toISOString() },
  }))
    .pipe(materialize(), delay(LATENCY), dematerialize());   // delay errors too

const newId = () => crypto.randomUUID().replaceAll('-', '').slice(0, 26).toUpperCase(); // ULID-like for the mock only

export const mockBackendInterceptor: HttpInterceptorFn = (req, next) => {
  const api = inject(API_URL);
  if (!req.url.startsWith(api)) return next(req);
  const path = req.url.slice(api.length);              // e.g. /notes/01J…
  return route(req, path) ?? fail(404, `No mock for ${req.method} ${path}`, req.url);
};

function route(req: HttpRequest<unknown>, path: string) {
  if (req.params.get('fail') === '1') return fail(500, 'Simulated server error', req.url);

  if (req.method === 'GET' && path === '/notes') {
    const cat = req.params.get('category');
    const q = req.params.get('q')?.toLowerCase();
    const page = Number(req.params.get('page') ?? 1);
    const limit = Number(req.params.get('limit') ?? 20);
    const filtered = notes
      .filter(n => !cat || n.category?.slug === cat)
      .filter(n => !q || `${n.title} ${n.preview}`.toLowerCase().includes(q))
      .toSorted((a, b) => Number(b.pinned) - Number(a.pinned) || b.updatedAt.localeCompare(a.updatedAt));
    const data = filtered.slice((page - 1) * limit, page * limit).map(({ content, ...summary }) => summary);
    return ok({ data, meta: { page, limit, total: filtered.length } });
  }

  const one = path.match(/^\/notes\/([\w-]+)$/);
  if (one) {
    const id = one[1]!;
    const idx = notes.findIndex(n => n.id === id);
    if (req.method === 'GET') return idx < 0 ? fail(404, 'Note not found', req.url) : ok(notes[idx]);
    if (req.method === 'PATCH') {
      if (idx < 0) return fail(404, 'Note not found', req.url);
      notes[idx] = { ...notes[idx]!, ...(req.body as object), updatedAt: new Date().toISOString() };
      return ok(notes[idx]);
    }
    if (req.method === 'DELETE') {
      notes = notes.filter(n => n.id !== id);   // the real API soft-deletes to Trash; the mock just removes
      return ok(null, 204);
    }
  }
  const pin = path.match(/^\/notes\/([\w-]+)\/pin$/);
  if (pin && req.method === 'PATCH') {
    const n = notes.find(x => x.id === pin[1]);
    if (!n) return fail(404, 'Note not found', req.url);
    n.pinned = !n.pinned;
    return ok({ ...n });
  }

  if (req.method === 'POST' && path === '/notes') {
    const body = req.body as { title: string; content: string };
    if (body.title.toLowerCase() === 'boom') return fail(500, 'Simulated server error', req.url);
    const now = new Date().toISOString();
    const note: NoteDetail = {
      id: newId(), title: body.title, content: body.content, preview: body.content.slice(0, 120),
      color: null, pinned: false, isFavorite: false, category: null, tags: [], createdAt: now, updatedAt: now,
    };
    notes = [note, ...notes];
    return ok(note, 201);
  }
  return null;
}
```

## Providers

```typescript
// app.config.ts (excerpt)
provideHttpClient(withInterceptors([
  ...(environment.useMockApi ? [mockBackendInterceptor] : []),
])),
{ provide: API_URL, useValue: environment.apiUrl },
```

Interceptor order matters later: in Lab 8 the auth/error interceptors go **before** the mock so they behave exactly as they will against the real server.

## `NotesStore` with `rxResource`

```typescript
// features/notes/data-access/notes-store.ts
import { Service, computed, inject, linkedSignal, signal } from '@angular/core';
import { rxResource } from '@angular/core/rxjs-interop';
import { firstValueFrom } from 'rxjs';
import { NotesApi } from './notes-api';

@Service()
export class NotesStore {
  private api = inject(NotesApi);
  readonly category = signal('all');
  readonly query = signal('');
  readonly limit = 8;
  readonly page = linkedSignal(() => { this.category(); this.query(); return 1; });

  readonly list = rxResource({
    params: () => ({ category: this.category(), q: this.query(), page: this.page(), limit: this.limit }),
    stream: ({ params }) => this.api.list(params),
  });

  readonly notes = computed(() => this.list.hasValue() ? this.list.value().data : []);
  readonly total = computed(() => this.list.hasValue() ? this.list.value().meta.total : 0);
  readonly pageCount = computed(() => Math.max(1, Math.ceil(this.total() / this.limit)));

  async toggleFavorite(id: string) {
    const n = this.notes().find(x => x.id === id);
    if (!n) return;
    this.patchLocal(id, { isFavorite: !n.isFavorite });                    // optimistic
    try { await firstValueFrom(this.api.update(id, { isFavorite: !n.isFavorite })); }
    catch { this.patchLocal(id, { isFavorite: n.isFavorite }); throw new Error('Could not update favourite'); }
  }

  private patchLocal(id: string, patch: Partial<{ isFavorite: boolean; pinned: boolean }>) {
    this.list.update(v => v && { ...v, data: v.data.map(n => (n.id === id ? { ...n, ...patch } : n)) });
  }
}
```

Notes:
- The search query should be debounced before it reaches the resource (Lab 10 adds that).
- Chips now come from the planned `GET /categories?module=note`. The plan doesn't include per-category counts, so show chips **without counts** (or count only the loaded page and label it so). Don't invent a counts endpoint.

## Independent task — `TasksApi` via `createCrudApi`

Reuse `createCrudApi<T, Q, C, U>` and `toParams` exactly as written in [Module 07 note 4](../../07-services-and-dependency-injection/04-myspce-api-layer.md) (`core/http/crud-api.ts`). Then:

```typescript
// features/tasks/data-access/tasks-api.ts
import { Service, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { createCrudApi } from '../../../core/http/crud-api';
import { Task } from '../../../core/models';

// planned contract: view=today|tomorrow|week|upcoming|completed
export interface TaskQuery { view?: 'today' | 'tomorrow' | 'week' | 'upcoming' | 'completed'; category?: string; page?: number; limit?: number; }
export interface CreateTaskDto { title: string; notes?: string | null; dueAt?: string | null; priority?: Task['priority']; categoryId?: string | null; }

@Service()
export class TasksApi {
  private http = inject(HttpClient);
  private crud = createCrudApi<Task, TaskQuery, CreateTaskDto>('tasks');
  readonly list = this.crud.list;
  readonly create = this.crud.create;
  readonly update = this.crud.update;
  readonly remove = this.crud.remove;
  toggle(id: string) { return this.http.patch<Task>(`${this.crud.base}/${id}/toggle`, {}); }   // planned endpoint
}
```

Mock routes for tasks follow the same pattern as notes:
- filter by `view` (compare `dueAt` with today; `completed` = `status === 'done'`)
- `PATCH /tasks/:id/toggle` flips `status` between `'todo'` and `'done'`
- `DELETE` removes

`NotesApi` can now also be rewritten on top of `createCrudApi<Note, NotesQuery, CreateNoteDto, UpdateNoteDto>('notes')`, keeping only `togglePin` and the `HttpContext` overloads as extra methods.
