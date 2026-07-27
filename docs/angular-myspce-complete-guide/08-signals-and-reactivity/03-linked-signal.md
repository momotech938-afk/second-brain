# `linkedSignal()` — Writable State That Depends on Other State

Back to [Module 08](README.md) · Previous: [`computed()` and `effect()`](02-computed-and-effect.md) · Next: [Resources](04-resources.md)

## 1. Learning objectives

- Recognise "derived but also user-editable" state.
- Use `linkedSignal` in its shorthand and full (`source` + `computation`) forms.
- Preserve or reset previous values intelligently.
- Replace `effect`-based resets.

## 2. Prerequisites

[`computed()` and `effect()`](02-computed-and-effect.md).

## 3. What is it?

A **writable** signal whose value is **recomputed from a source** whenever the source changes, but which you can also `set()` yourself in between. Stable since v20.

## 4. Why does Angular need it?

`computed` is read-only; `signal` doesn't know about other state. Many UI values are both:

- **Page number:** user changes it, but it must reset to 1 when the filter changes.
- **Selected item:** user picks one, but if the list changes and the item disappears, pick a sensible default.
- **Editable draft:** starts from the saved note's title; user edits it; resets when a different note is opened.

Before `linkedSignal`, people wrote effects to reset state — which runs late and causes an extra render with stale state.

## 5. How it works internally

`linkedSignal` behaves like a `computed` for its source dependencies and like a `signal` for writes:

- When a **source** dependency changes, the next read recomputes the value (discarding manual writes).
- `set()`/`update()` override the value until the source changes again.
- The full form's `computation(source, previous)` receives the previous `{ source, value }`, so you can keep a user's choice when it's still valid.

## 6. Syntax and API reference

```typescript
import { linkedSignal } from '@angular/core';

// Shorthand: recompute from the function's reads
const page = linkedSignal(() => {
  this.category();       // dependency
  this.query();          // dependency
  return 1;              // reset to 1 whenever filters change
});

// Full form
const selectedId = linkedSignal<Note[], string | null>({
  source: () => this.visibleNotes(),
  computation: (notes, previous) => {
    const keep = previous?.value;
    return keep && notes.some(n => n.id === keep) ? keep : (notes[0]?.id ?? null);
  },
  equal: (a, b) => a === b,      // optional
});

page.set(3);                     // manual write, until the source changes
```

## 7. Smallest working example

```typescript
// theme preview in Settings → Appearance
const savedTheme = signal<'light' | 'dark'>('dark');
const previewTheme = linkedSignal(() => savedTheme());   // starts as saved; user can preview others

previewTheme.set('light');   // preview → 'light'
savedTheme.set('dark');      // no change → previewTheme stays 'light'
savedTheme.set('light');     // source changed → previewTheme resets to 'light'
```

## 8. Step-by-step

1. `previewTheme` starts equal to `savedTheme`.
2. Manual `set` overrides it.
3. Setting `savedTheme` to the **same** value doesn't count as a change (equality), so the preview stays.
4. A real change of the source resets the preview.

## 9. Myspce real-world example — pagination that resets on filter change

```typescript
// features/notes/pages/notes-page.ts
import { Component, computed, inject, linkedSignal, signal } from '@angular/core';
import { NotesStore } from '../data-access/notes-store';

@Component({
  selector: 'app-notes-page',
  templateUrl: './notes-page.html',
})
export class NotesPage {
  protected store = inject(NotesStore);
  protected readonly pageSize = 12;

  protected page = linkedSignal(() => {
    this.store.category();
    this.store.query();
    return 1;
  });

  protected pageCount = computed(() => Math.max(1, Math.ceil(this.store.visibleNotes().length / this.pageSize)));
  protected pageItems = computed(() => {
    const start = (this.page() - 1) * this.pageSize;
    return this.store.visibleNotes().slice(start, start + this.pageSize);
  });

  next() { this.page.update(p => Math.min(p + 1, this.pageCount())); }
  prev() { this.page.update(p => Math.max(p - 1, 1)); }
}
```

```html
<app-notes-grid [notes]="pageItems()" />
<nav aria-label="Pagination" class="mt-4 flex items-center justify-center gap-3">
  <button type="button" (click)="prev()" [disabled]="page() === 1">Previous</button>
  <span>Page {{ page() }} of {{ pageCount() }}</span>
  <button type="button" (click)="next()" [disabled]="page() === pageCount()">Next</button>
</nav>
```

Change the category chip → page snaps back to 1 in the same render. No effect, no flicker.

(With server-side pagination the page goes into the request instead; see [Module 13](../13-http-and-api-integration/04-uploads-pagination-testing.md).)

## 10. Advanced example — editable draft that resets when the note changes

```typescript
// features/notes/pages/note-editor-page.ts
export class NoteEditorPage {
  note = input.required<Note>();                         // from resolver/route
  protected draftTitle = linkedSignal(() => this.note().title);
  protected dirty = computed(() => this.draftTitle() !== this.note().title);
}
```

```html
<input class="text-2xl font-semibold" [value]="draftTitle()" (input)="draftTitle.set($any($event.target).value)" />
@if (dirty()) { <span class="text-xs text-amber-500">Unsaved</span> }
```

Navigate from note A to note B (same component instance reused by the router) → `note` input changes → draft resets to B's title. This is the v22 migration pattern too: when a component had both `model('x')` and an `xChange` output, the migration turns it into `input` + `linkedSignal` + `output`.

## 11. Visual explanation

```text
source (category, query) ─change─► page = 1 (recomputed)
user clicks Next ─────────────────► page = 2 (manual write)
user clicks Next ─────────────────► page = 3
source changes ───────────────────► page = 1 (manual value discarded)
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `effect(() => { category(); page.set(1); })` | `linkedSignal` |
| Using `linkedSignal` when the value is never written manually | that's just `computed` |
| Forgetting that equal source values don't trigger reset | expected behaviour; use a counter signal if you need "reset on every action" |
| Putting heavy logic in `computation` | keep it cheap; it runs on each source change |

## 13. Debugging guide

- Value didn't reset → the source didn't actually change (same value), or the function didn't read the signal you expected (dynamic dependencies).
- Value resets too often → the source function reads more signals than intended; read only what should trigger a reset.

## 14. Best practices

- Use the shorthand for "reset to X when Y changes".
- Use the full form when you want to **keep** the previous choice if still valid.
- Combine with `computed` for everything read-only.

## 15. Angular vs React

React would use `useState` + `useEffect(() => setPage(1), [category])` (an extra render with stale state) or a `key` reset. `linkedSignal` does it in one consistent step.

## 16. Testing

```typescript
it('resets page when category changes', () => {
  const fixture = TestBed.createComponent(NotesPage);
  const page = fixture.componentInstance as unknown as { page: WritableSignal<number> };
  page.page.set(3);
  TestBed.inject(NotesStore).category.set('work');
  expect(page.page()).toBe(1);
});
```

## 17. Practice questions

1. Use the full form to keep the selected credential selected if it's still in the filtered list.
2. Implement Calendar's "selected day resets to the 1st when the month changes".
3. Explain why `linkedSignal` beats an effect for the editor draft.

## 18. Interview questions

- **Q: What's `linkedSignal`?** A writable signal that recomputes from a source when the source changes, while allowing manual writes in between.
- **Q: Shorthand vs full form?** Shorthand recomputes from a function; the full form separates `source` and `computation(source, previous)` so you can use the previous value.

## 19. Revision notes

`linkedSignal(() => …)` reset pattern · `linkedSignal({ source, computation(src, prev) })` keep-if-valid pattern · writable · replaces reset effects.

## 20. Completion checklist

- [ ] Notes pagination resets on filter change.
- [ ] Editor draft resets on note change.
- [ ] Zero reset-effects in my code.
