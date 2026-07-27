# Lab 5 — Solution

Back to [Lab 5](../lab-05-signal-ui-state.md) · [Labs index](../README.md)

## `NotesStore`

```typescript
// features/notes/data-access/notes-store.ts
import { Service, computed, signal } from '@angular/core';
import { Category, Note } from '../../../core/models';
import { MOCK_NOTES } from '../../../../testing/mock-data';

@Service()
export class NotesStore {
  private readonly _notes = signal<Note[]>(MOCK_NOTES);
  readonly notes = this._notes.asReadonly();

  readonly category = signal('all');
  readonly query = signal('');

  readonly categories = computed(() => {
    const map = new Map<string, Category>();
    for (const n of this._notes()) if (n.category) map.set(n.category.slug, n.category);
    return [...map.values()].toSorted((a, b) => a.name.localeCompare(b.name));
  });

  readonly visibleNotes = computed(() => {
    const cat = this.category();
    const q = this.query().trim().toLowerCase();
    return this._notes()
      .filter(n => cat === 'all' || n.category?.slug === cat)
      .filter(n => !q || `${n.title} ${n.preview}`.toLowerCase().includes(q))
      .toSorted((a, b) => Number(b.pinned) - Number(a.pinned) || b.updatedAt.localeCompare(a.updatedAt));
  });

  readonly categoryOptions = computed(() => {
    const counts = new Map<string, number>();
    for (const n of this._notes()) if (n.category) counts.set(n.category.slug, (counts.get(n.category.slug) ?? 0) + 1);
    return [
      { value: 'all', label: 'All', count: this._notes().length },
      ...this.categories().map(c => ({ value: c.slug, label: c.name, count: counts.get(c.slug) ?? 0 })),
    ];
  });

  toggleFavorite(id: string) {
    this._notes.update(list => list.map(n => (n.id === id ? { ...n, isFavorite: !n.isFavorite } : n)));
  }
  togglePin(id: string) {
    this._notes.update(list => list.map(n => (n.id === id ? { ...n, pinned: !n.pinned } : n)));
  }
}
```

(The relative import of mock data from `src/testing` is temporary; Lab 6 replaces it with the API.)

## `NoteCard`

```typescript
// features/notes/components/note-card.ts
import { Component, input, output } from '@angular/core';
import { DatePipe } from '@angular/common';
import { Note } from '../../../core/models';

@Component({
  selector: 'app-note-card',
  imports: [DatePipe],
  template: `
    <article class="flex h-full flex-col rounded-xl p-4 shadow-sm" [style.background]="note().color ?? null"
             [class.bg-white]="!note().color" [class.dark:bg-slate-900]="!note().color">
      <header class="mb-2 flex items-start justify-between gap-2">
        <h3 class="font-semibold text-slate-900">{{ note().title }}</h3>
        <div class="flex gap-1">
          <button type="button" (click)="pinToggled.emit(note().id)" [attr.aria-pressed]="note().pinned"
                  [attr.aria-label]="(note().pinned ? 'Unpin ' : 'Pin ') + note().title">📌</button>
          <button type="button" (click)="favoriteToggled.emit(note().id)" [attr.aria-pressed]="note().isFavorite"
                  [attr.aria-label]="'Favourite ' + note().title">{{ note().isFavorite ? '★' : '☆' }}</button>
        </div>
      </header>
      <p class="line-clamp-3 flex-1 text-sm text-slate-700">{{ note().preview }}</p>
      <footer class="mt-3 flex items-center justify-between text-xs text-slate-600">
        <time [attr.datetime]="note().updatedAt">{{ note().updatedAt | date: 'd MMM y' }}</time>
        @if (note().category; as c) { <span class="rounded-full bg-white/70 px-2 py-0.5">{{ c.name }}</span> }
      </footer>
    </article>
  `,
})
export class NoteCard {
  note = input.required<Note>();
  favoriteToggled = output<string>();
  pinToggled = output<string>();
}
```

(`[class.dark:bg-slate-900]` is a valid class binding — Angular treats the whole string after `class.` as the class name, and Tailwind sees the literal in source.)

## `NotesPage`

```typescript
// features/notes/notes-page.ts
import { Component, computed, inject, linkedSignal } from '@angular/core';
import { PageHeader } from '../../shared/ui/page-header';
import { ChipFilter } from '../../shared/ui/chip-filter';
import { SearchBox } from '../../shared/ui/search-box';
import { EmptyState } from '../../shared/ui/empty-state';
import { NoteCard } from './components/note-card';
import { NotesStore } from './data-access/notes-store';

@Component({
  selector: 'app-notes-page',
  imports: [PageHeader, ChipFilter, SearchBox, EmptyState, NoteCard],
  template: `
    <app-page-header title="Notes">
      <app-search-box actions placeholder="Search notes…" [(value)]="store.query" />
    </app-page-header>
    <app-chip-filter label="Note category" [options]="store.categoryOptions()" [(value)]="store.category" />

    @if (pageItems().length) {
      <div class="mt-4 grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        @for (note of pageItems(); track note.id) {
          <app-note-card [note]="note" (favoriteToggled)="store.toggleFavorite($event)" (pinToggled)="store.togglePin($event)" />
        }
      </div>
      @if (pageCount() > 1) {
        <nav aria-label="Pagination" class="mt-6 flex items-center justify-center gap-3 text-sm">
          <button type="button" (click)="page.set(page() - 1)" [disabled]="page() === 1">Previous</button>
          <span>Page {{ page() }} of {{ pageCount() }}</span>
          <button type="button" (click)="page.set(page() + 1)" [disabled]="page() === pageCount()">Next</button>
        </nav>
      }
    } @else if (store.query()) {
      <app-empty-state [title]="'No notes match “' + store.query() + '”'" hint="Try another word or category." />
    } @else {
      <app-empty-state title="No notes yet" hint="Capture your first idea." />
    }
  `,
})
export default class NotesPage {
  protected store = inject(NotesStore);
  protected readonly pageSize = 8;
  protected page = linkedSignal(() => { this.store.category(); this.store.query(); return 1; });
  protected pageCount = computed(() => Math.max(1, Math.ceil(this.store.visibleNotes().length / this.pageSize)));
  protected pageItems = computed(() => this.store.visibleNotes().slice((this.page() - 1) * this.pageSize, this.page() * this.pageSize));
}
```

## Independent task — credentials selection (core logic)

```typescript
// features/credentials/data-access/credentials-ui-store.ts
@Service()
export class CredentialsUiStore {
  private readonly _all = signal<CredentialSummary[]>(MOCK_CREDENTIALS);
  readonly category = signal('all');
  readonly query = signal('');
  readonly sort = signal<'name' | 'recent'>('name');

  readonly filtered = computed(() => {
    const q = this.query().trim().toLowerCase();
    const cat = this.category();
    const list = this._all().filter(c =>
      (cat === 'all' || c.category?.slug === cat) &&
      (!q || [c.name, c.url ?? '', c.username].some(s => s.toLowerCase().includes(q))));
    return this.sort() === 'name'
      ? list.toSorted((a, b) => a.name.localeCompare(b.name))
      : list.toSorted((a, b) => (b.lastViewedAt ?? '').localeCompare(a.lastViewedAt ?? ''));
  });

  readonly selectedId = linkedSignal<CredentialSummary[], string | null>({
    source: () => this.filtered(),
    computation: (list, prev) => (prev?.value && list.some(c => c.id === prev.value) ? prev.value : (list[0]?.id ?? null)),
  });
  readonly selected = computed(() => this.filtered().find(c => c.id === this.selectedId()) ?? null);
}
```

Layout hint: `grid lg:grid-cols-[1fr_22rem]` with the list on the left and `@if (store.selected(); as c) { … }` details on the right (hidden below `lg`).
