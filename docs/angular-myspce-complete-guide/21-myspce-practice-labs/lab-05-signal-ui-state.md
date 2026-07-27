# Lab 5 — Signal-Based UI State

Back to [Labs](README.md) · Solution: [lab-05-solution.md](solutions/lab-05-solution.md) · Previous: [Lab 4](lab-04-routing-and-nested-layouts.md) · Next: [Lab 6](lab-06-typed-api-services.md)

## 1. Learning objective

Drive the Notes screen entirely with signals: a `NotesStore` holding mock notes, category chips with counts, search, favourite/pin toggles, and pagination that resets on filter changes.

## 2. Prerequisites

[Module 08](../08-signals-and-reactivity/README.md), [Lab 4](lab-04-routing-and-nested-layouts.md).

## 3. Problem statement

The Notes design shows category chips (All, Work, Personal, Learning, Ideas, Travel, Finance), a search box, and colourful cards. Filtering must feel instant and stay consistent across components.

## 4. Expected behaviour

- Chips show counts; clicking one filters the grid.
- Typing in search filters by title and preview (case-insensitive).
- Pinned notes appear first; favourite star toggles instantly.
- 8 notes per page; changing chip or search resets to page 1.
- Empty search result shows "No notes match …".

## 5. Relevant concepts

`signal`, `computed`, `linkedSignal`, immutable `update`, `model()` two-way binding, `track`.

## 6. Step-by-step implementation

1. Create mock data `src/testing/mock-data.ts` with ~20 notes across the seven categories (use the `Note` model from [project context](../MYSPCE_PROJECT_CONTEXT.md#4-data-entities-used-in-examples)).
2. Create `NotesStore` (`@Service()`):
   - private `_notes = signal<Note[]>(MOCK_NOTES)` (real API in Lab 6)
   - `category = signal('all')`, `query = signal('')`
   - `categories` (derived from notes: unique categories)
   - `visibleNotes` computed: filter + search + sort (pinned first, then `updatedAt` desc)
   - `categoryOptions` computed with counts
   - `toggleFavorite(id)`, `togglePin(id)` immutable updates
3. Build `ChipFilter` (`model()`), `SearchBox` (`model()`), `NoteCard` (inputs + outputs) if you haven't.
4. `NotesPage`:
   - inject the store
   - `page = linkedSignal(() => { store.category(); store.query(); return 1; })`
   - `pageCount`, `pageItems` computed
   - template: header with search, chips, grid with `@for … track note.id`, pager, empty state
5. Verify in Angular DevTools that toggling a favourite only updates one card.

## 7. Complete working code

`NotesStore`, `NotesPage` and `NoteCard` are in the [solution](solutions/lab-05-solution.md).

## 8. Important lines explained

- `this._notes.update(list => list.map(n => n.id === id ? { ...n, isFavorite: !n.isFavorite } : n))` — only the changed note gets a new reference, so only its card re-renders.
- `[(value)]="store.category"` — binds the store's writable signal to the chip filter's `model()`.
- `linkedSignal(() => { …; return 1; })` — "reset to 1 when these change", without an effect.
- `.toSorted(...)` — sorting without mutating state.

## 9. Expected output

Click "Work": grid shows only Work notes, page 1, chip "Work" highlighted with `aria-checked="true"`. Type "aws": only matching notes.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Grid doesn't update after favourite | you mutated `note.isFavorite`; use `update` with new objects |
| Page doesn't reset | `linkedSignal` function doesn't read `category`/`query` |
| All cards re-render on one toggle | `track` isn't `note.id`, or every note object was recreated |
| `[(value)]="store.category()"` error | pass the signal, not its value |

## 11. Test cases

```typescript
describe('NotesStore', () => {
  it('filters by category and query together', () => {
    const s = TestBed.inject(NotesStore);
    s.category.set('work');
    s.query.set('angular');
    expect(s.visibleNotes().every(n => n.category?.slug === 'work' && /angular/i.test(n.title + n.preview))).toBe(true);
  });

  it('puts pinned notes first', () => {
    const s = TestBed.inject(NotesStore);
    const first = s.visibleNotes()[0]!;
    s.togglePin(s.visibleNotes().at(-1)!.id);
    expect(s.visibleNotes()[0]!.id).not.toBe(first.id);
  });
});
```

## 12. Independent practice task

Apply the same approach to **Credentials** (mock 10 entries matching the design: GitHub, AWS Console, Google, Hostinger, LinkedIn, Instagram, Office WiFi, MySQL, Netflix): chips (All, Work, Personal, Finance, Social, Other), search across name/URL/username, a sort select (Name A–Z / Recently viewed), and a **selected credential** that stays selected while it's in the filtered list, otherwise falls back to the first item (`linkedSignal` full form). Show the selected item's details in a side panel on desktop.

## 13. Solution

See [solutions/lab-05-solution.md](solutions/lab-05-solution.md).

## 14. Completion checklist

- [ ] Notes filtering, search, pin/favourite, pagination work.
- [ ] Only one card re-renders on toggle (checked in DevTools).
- [ ] Credentials selection with keep-if-valid logic.
- [ ] Tests pass; tagged `lab-05`.
