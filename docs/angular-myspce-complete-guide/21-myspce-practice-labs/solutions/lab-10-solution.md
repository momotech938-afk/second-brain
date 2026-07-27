# Lab 10 — Solution

Back to [Lab 10](../lab-10-search-filter-sort-paginate.md) · [Labs index](../README.md)

## Params parsing

```typescript
// features/notes/data-access/notes-params.ts
export const NOTE_SORTS = ['updated', 'title', 'created'] as const;
export type NoteSort = (typeof NOTE_SORTS)[number];

export interface NotesListParams { q: string; category: string; sort: NoteSort; page: number; }
export const NOTES_DEFAULTS: NotesListParams = { q: '', category: 'all', sort: 'updated', page: 1 };

export function parseNotesParams(raw: Partial<Record<keyof NotesListParams, string | undefined>>): NotesListParams {
  const page = Number(raw.page);
  return {
    q: (raw.q ?? '').trim().slice(0, 100),
    category: /^[a-z0-9-]{1,40}$/.test(raw.category ?? '') ? raw.category! : NOTES_DEFAULTS.category,
    sort: (NOTE_SORTS as readonly string[]).includes(raw.sort ?? '') ? (raw.sort as NoteSort) : NOTES_DEFAULTS.sort,
    page: Number.isInteger(page) && page >= 1 && page <= 10_000 ? page : NOTES_DEFAULTS.page,
  };
}

/** Converts params to a queryParams object where defaults become null (removed from the URL). */
export function toQueryParams(p: Partial<NotesListParams>): Record<string, string | number | null> {
  const out: Record<string, string | number | null> = {};
  for (const [k, v] of Object.entries(p) as [keyof NotesListParams, NotesListParams[keyof NotesListParams]][]) {
    out[k] = v === NOTES_DEFAULTS[k] || v === '' ? null : v;
  }
  return out;
}
```

## `NotesPage` (URL as the source of truth)

```typescript
// features/notes/notes-page.ts (relevant parts)
import { Component, computed, inject, input, linkedSignal } from '@angular/core';
import { rxResource, takeUntilDestroyed, toObservable } from '@angular/core/rxjs-interop';
import { ActivatedRoute, Router } from '@angular/router';
import { debounceTime, distinctUntilChanged } from 'rxjs';
import { NotesApi } from './data-access/notes-api';
import { NOTE_SORTS, NotesListParams, parseNotesParams, toQueryParams } from './data-access/notes-params';

@Component({ /* selector, imports, templateUrl */ })
export default class NotesPage {
  private router = inject(Router);
  private route = inject(ActivatedRoute);
  private api = inject(NotesApi);

  // query params via withComponentInputBinding()
  q = input<string>();
  category = input<string>();
  sort = input<string>();
  page = input<string>();

  protected readonly limit = 8;
  protected readonly sorts = NOTE_SORTS;
  protected params = computed(() =>
    parseNotesParams({ q: this.q(), category: this.category(), sort: this.sort(), page: this.page() }));

  /** What the user is typing; follows the URL when it changes externally (Back button, shared link). */
  protected draft = linkedSignal(() => this.params().q);

  protected list = rxResource({
    params: () => ({ ...this.params(), limit: this.limit }),
    stream: ({ params }) => this.api.list(params),   // NotesApi.list maps category 'all' → no param
  });
  protected pageCount = computed(() =>
    this.list.hasValue() ? Math.max(1, Math.ceil(this.list.value().meta.total / this.limit)) : 1);

  constructor() {
    toObservable(this.draft)
      .pipe(debounceTime(300), distinctUntilChanged(), takeUntilDestroyed())
      .subscribe(q => { if (q.trim() !== this.params().q) this.setParams({ q: q.trim() }, { replace: true }); });
  }

  setParams(patch: Partial<NotesListParams>, { replace = false } = {}) {
    const resetsPage = Object.keys(patch).some(k => k !== 'page');
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: toQueryParams(resetsPage ? { ...patch, page: 1 } : patch),
      queryParamsHandling: 'merge',
      replaceUrl: replace,
    });
  }
}
```

`NotesApi.list` and `NotesQuery` gain `sort?: 'updated' | 'title' | 'created'`, sent only when not the default.

Since the URL now owns `q`/`category`/`page`, remove those signals from `NotesStore` (keep it for mutations such as favourite/pin/remove), or have the store read from the page. Don't keep two sources of truth.

```html
<!-- notes-page.html (controls) -->
<div class="flex flex-wrap items-center gap-3">
  <label class="sr-only" for="note-search">Search notes</label>
  <input id="note-search" type="search" placeholder="Search notes…" class="rounded-lg border px-3 py-2"
         [value]="draft()" (input)="draft.set($any($event.target).value)" />

  <app-chip-filter label="Note category" [options]="categoryOptions()"
                   [value]="params().category" (valueChange)="setParams({ category: $event })" />

  <label class="text-sm" for="note-sort">Sort</label>
  <select id="note-sort" class="rounded-lg border px-2 py-2 text-sm" [value]="params().sort"
          (change)="setParams({ sort: $any($event.target).value })">
    <option value="updated">Last updated</option>
    <option value="title">Title A–Z</option>
    <option value="created">Created</option>
  </select>
</div>

<!-- …states from Lab 7, then the pager: -->
<nav aria-label="Pagination" class="mt-6 flex items-center justify-center gap-3 text-sm">
  <button type="button" [disabled]="params().page <= 1" (click)="setParams({ page: params().page - 1 })">Previous</button>
  <span aria-live="polite">Page {{ params().page }} of {{ pageCount() }}</span>
  <button type="button" [disabled]="params().page >= pageCount()" (click)="setParams({ page: params().page + 1 })">Next</button>
</nav>
```

`ChipFilter` is used one-way here (`[value]` + `(valueChange)`) because the URL, not a signal, owns the value. `model()` supports that automatically.

## Mock: sort + search

```typescript
// inside route() for GET /notes
const sort = req.params.get('sort') ?? 'updated';
const sorter: Record<string, (a: NoteDetail, b: NoteDetail) => number> = {
  updated: (a, b) => b.updatedAt.localeCompare(a.updatedAt),
  created: (a, b) => b.createdAt.localeCompare(a.createdAt),
  title: (a, b) => a.title.localeCompare(b.title),
};
// …filtered.toSorted((a, b) => Number(b.pinned) - Number(a.pinned) || (sorter[sort] ?? sorter['updated']!)(a, b))

// GET /search?q=   (endpoint planned; response shape proposed)
if (req.method === 'GET' && path === '/search') {
  const q = (req.params.get('q') ?? '').toLowerCase();
  const hit = (s: string) => s.toLowerCase().includes(q);
  return ok({
    notes: notes.filter(n => hit(n.title) || hit(n.preview)).slice(0, 5).map(n => ({ id: n.id, title: n.title })),
    credentials: MOCK_CREDENTIALS.filter(c => hit(c.name) || hit(c.url ?? '')).slice(0, 5).map(c => ({ id: c.id, title: c.name })),
    tasks: MOCK_TASKS.filter(t => hit(t.title)).slice(0, 5).map(t => ({ id: t.id, title: t.title })),
  });
}
```

## `GlobalSearch`

```typescript
// core/layout/global-search.ts
import { Component, ElementRef, computed, inject, linkedSignal, signal, viewChild } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { Router } from '@angular/router';
import { debounceTime, distinctUntilChanged, map } from 'rxjs';
import { API_URL } from '../http/api-url.token';

interface Hit { id: string; title: string; }
interface SearchResult { notes: Hit[]; credentials: Hit[]; tasks: Hit[]; }
interface Row extends Hit { group: 'Notes' | 'Credentials' | 'Tasks'; url: string; }

@Component({
  selector: 'app-global-search',
  host: { '(document:keydown.control.k)': 'focus($event)', '(document:keydown.meta.k)': 'focus($event)' },
  template: `
    <div class="relative w-full max-w-md">
      <label for="global-search" class="sr-only">Search everything</label>
      <input #box id="global-search" type="search" placeholder="Search… (Ctrl K)" autocomplete="off"
             role="combobox" aria-controls="global-search-list" [attr.aria-expanded]="open()"
             [attr.aria-activedescendant]="open() && rows()[active()] ? 'gs-' + active() : null"
             class="w-full rounded-lg border px-3 py-2"
             [value]="text()" (input)="text.set($any($event.target).value)"
             (keydown.arrowdown)="move(1, $event)" (keydown.arrowup)="move(-1, $event)"
             (keydown.enter)="pick(rows()[active()])" (keydown.escape)="text.set('')" />

      @if (open()) {
        <ul id="global-search-list" role="listbox" aria-label="Search results"
            class="absolute z-40 mt-1 max-h-96 w-full overflow-auto rounded-lg bg-white p-1 shadow-lg dark:bg-slate-800">
          @if (results.isLoading() && !results.hasValue()) {
            <li class="px-3 py-2 text-sm text-slate-500" role="presentation">Searching…</li>
          } @else if (!rows().length) {
            <li class="px-3 py-2 text-sm text-slate-500" role="presentation">No results for “{{ query() }}”</li>
          }
          @for (r of rows(); track r.group + r.id; let i = $index) {
            @if (i === 0 || rows()[i - 1]!.group !== r.group) {
              <li role="presentation" class="px-3 pt-2 text-xs font-semibold uppercase text-slate-400">{{ r.group }}</li>
            }
            <li [id]="'gs-' + i" role="option" [attr.aria-selected]="i === active()"
                class="cursor-pointer rounded px-3 py-2 text-sm" [class.bg-slate-100]="i === active()"
                (mousedown)="$event.preventDefault()" (click)="pick(r)">{{ r.title }}</li>
          }
        </ul>
      }
    </div>
  `,
})
export class GlobalSearch {
  private router = inject(Router);
  private api = inject(API_URL);
  private box = viewChild.required<ElementRef<HTMLInputElement>>('box');

  protected text = signal('');
  protected query = toSignal(
    toObservable(this.text).pipe(map(t => t.trim()), debounceTime(250), distinctUntilChanged()),
    { initialValue: '' },
  );

  protected results = httpResource<SearchResult>(() =>
    this.query().length >= 2 ? { url: `${this.api}/search`, params: { q: this.query() } } : undefined);

  protected rows = computed<Row[]>(() => {
    if (!this.results.hasValue()) return [];
    const r = this.results.value();
    return [
      ...r.notes.map(h => ({ ...h, group: 'Notes' as const, url: `/notes/${h.id}` })),
      ...r.credentials.map(h => ({ ...h, group: 'Credentials' as const, url: `/credentials?selected=${h.id}` })),
      ...r.tasks.map(h => ({ ...h, group: 'Tasks' as const, url: `/tasks?task=${h.id}` })),
    ];
  });
  protected open = computed(() => this.query().length >= 2);
  protected active = linkedSignal({ source: this.rows, computation: () => 0 });   // reset when results change

  protected move(delta: number, e: Event) {
    e.preventDefault();
    const n = this.rows().length;
    if (n) this.active.update(i => (i + delta + n) % n);
  }
  protected pick(r: Row | undefined) {
    if (!r) return;
    this.text.set('');
    this.router.navigateByUrl(r.url);
  }
  protected focus(e: Event) { e.preventDefault(); this.box().nativeElement.focus(); }
}
```

Notes:
- `httpResource` returns `undefined` params when the query is short → no request; switching queries cancels the old request.
- `(mousedown)="$event.preventDefault()"` keeps focus in the input while clicking an option.
- For a production combobox, consider **Angular Aria**'s combobox/listbox primitives ([Module 16 note 4](../../16-advanced-components/04-reusable-ui-kit.md)).
- The `selected`/`task` query params on Credentials/Tasks are proposals for deep-linking. Wire them up with an input and `linkedSignal`, as in Lab 5.

## Independent task — Tasks URL state + Credentials "Load more"

```typescript
// features/tasks/pages/tasks-tab.ts (excerpt)
view = input<string>();
category = input<string>();
priority = input<string | string[]>();       // repeated ?priority=… arrives as an array

protected params = computed(() => ({
  view: (['today', 'tomorrow', 'week', 'upcoming', 'completed'] as const).find(v => v === this.view()) ?? 'today',
  category: this.category() ?? '',
  priority: ([] as string[]).concat(this.priority() ?? []).filter(p => ['low', 'medium', 'high'].includes(p)),
}));
```

`createCrudApi`'s `toParams` already appends arrays as repeated params.

```typescript
// features/credentials/credentials-page.ts (excerpt) — "Load more"
protected filters = computed(() => ({ q: this.q() ?? '', category: this.category() ?? 'all' }));
protected page = linkedSignal({ source: this.filters, computation: () => 1 });

private pageRes = rxResource({
  params: () => ({ ...this.filters(), page: this.page(), limit: 20 }),
  stream: ({ params }) => this.api.list(params),
});

protected items = linkedSignal<{ filters: unknown; res: Paginated<CredentialSummary> | undefined }, CredentialSummary[]>({
  source: () => ({ filters: this.filters(), res: this.pageRes.hasValue() ? this.pageRes.value() : undefined }),
  computation: ({ filters, res }, prev) => {
    if (!res) return prev?.value ?? [];
    const sameFilters = prev?.source.filters === filters;
    return res.meta.page === 1 || !sameFilters ? res.data : [...prev!.value, ...res.data];
  },
});
protected hasMore = computed(() => this.pageRes.hasValue() && this.items().length < this.pageRes.value().meta.total);
loadMore() { this.page.update(p => p + 1); }
```

Template: list `items()`, then `@if (hasMore()) { <button (click)="loadMore()" [disabled]="pageRes.isLoading()">Load more</button> }`.
