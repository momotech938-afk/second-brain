# Rendering Performance

Back to [Module 17](README.md) · Next: [Loading and bundles](02-loading-and-bundles.md)

## 1. Learning objectives

- Explain where rendering time goes in Angular.
- Apply OnPush + signals + immutable data correctly.
- Remove expensive work from templates.
- Render long lists efficiently (track, virtual scrolling, pagination).
- Optimise images.

## 2. Prerequisites

[Change detection](../09-lifecycle-and-change-detection/02-change-detection.md), [Signals](../08-signals-and-reactivity/README.md).

## 3. Where time goes

| Cost | Caused by | Fix |
|---|---|---|
| Change detection checks | many views checked often | OnPush (v22 default) + signals |
| Template expressions | method calls, filtering/sorting in templates | `computed()`, pure pipes |
| DOM creation | recreating rows | stable `track`, avoid recreating arrays of new objects |
| Layout/paint | huge DOM, layout thrashing | virtual scroll, pagination, `afterNextRender` phases |
| Images | large unoptimised images, no dimensions (CLS) | `NgOptimizedImage`, `srcset`, lazy loading |
| JavaScript size | parse/compile time | [next note](02-loading-and-bundles.md) |

## 4. How Angular renders (recap)

A signal change marks consumer views dirty → Angular schedules a tick → it refreshes dirty OnPush views (skipping clean subtrees) → each binding compares old/new and patches the DOM only on change. Your job: keep the number of dirty views small and each binding cheap.

## 5. Patterns

### Derive once, read many times

```typescript
// ❌ runs on every check of the view, for every row
// <span>{{ formatCategory(note) }}</span>
// <app-notes-grid [notes]="notes().filter(n => n.pinned)" />   ← new array each check → child input changes every time

// ✅
pinned = computed(() => this.notes().filter(n => n.pinned));   // new array only when notes change
```

### Keep references stable

`computed` only produces a new value when dependencies change, so `[notes]="pinned()"` passes the **same** array until notes change — the child stays clean.

Custom `equal` can avoid notifying when content is equal even if the reference changed:

```typescript
ids = computed(() => this.notes().map(n => n.id), { equal: (a, b) => a.length === b.length && a.every((x, i) => x === b[i]) });
```

### `track` by identity

`@for (n of notes(); track n.id)` reuses DOM for unchanged notes across filtering/sorting. `track $index` on a reorderable list re-renders every row's content.

### Pure pipes for formatting

`{{ file.sizeBytes | fileSize }}` is memoised per binding; a method call isn't.

## 6. Virtual scrolling

For the Files list or Recent Activity with thousands of items, render only what's visible:

```typescript
// features/files/components/file-list.ts
import { Component, input } from '@angular/core';
import { ScrollingModule } from '@angular/cdk/scrolling';
import { FileSizePipe } from '../../../shared/pipes/file-size';
import { DatePipe } from '@angular/common';

@Component({
  selector: 'app-file-list',
  imports: [ScrollingModule, FileSizePipe, DatePipe],
  template: `
    <cdk-virtual-scroll-viewport itemSize="56" class="h-[70dvh]">
      <div *cdkVirtualFor="let f of files(); trackBy: byId" class="flex h-14 items-center gap-3 border-b px-3">
        <span class="flex-1 truncate">{{ f.originalName }}</span>
        <span class="text-xs text-slate-500">{{ f.sizeBytes | fileSize }}</span>
        <time class="text-xs text-slate-500">{{ f.createdAt | date }}</time>
      </div>
    </cdk-virtual-scroll-viewport>
  `,
})
export class FileList {
  files = input.required<FileEntity[]>();
  protected byId = (_: number, f: FileEntity) => f.id;
}
```

`npm i @angular/cdk`. Rows must have a fixed height (`itemSize`). For typical Myspce data sizes (dozens to hundreds), **server pagination** is simpler and usually enough; reach for virtual scroll when a single screen truly needs thousands of rows.

## 7. Images with `NgOptimizedImage`

Memories photos and avatars are the heaviest assets.

```typescript
import { NgOptimizedImage } from '@angular/common';
```

```html
<!-- above the fold, the LCP candidate -->
<img ngSrc="/assets/hero-mountains.webp" width="1200" height="400" priority alt="" class="h-40 w-full object-cover" />

<!-- gallery thumbnails: lazy by default, responsive sizes -->
<img [ngSrc]="m.thumbnailUrl" width="320" height="240" sizes="(min-width: 1024px) 25vw, 50vw" [alt]="m.title" />

<!-- fill mode when the container controls size -->
<div class="relative aspect-video"><img [ngSrc]="photo.url" fill [alt]="photo.alt" class="object-cover" /></div>
```

What it does: enforces `width`/`height` (prevents layout shift), lazy-loads non-priority images, sets `fetchpriority="high"` for `priority`, generates `srcset` when an image loader is configured, and warns in dev about oversized images. Serve WebP/AVIF thumbnails from S3 (the build plan's thumbnail generator) rather than full-size photos.

## 8. Myspce example — a fast Notes grid

```typescript
export default class NotesPage {
  protected store = inject(NotesStore);
  protected pageItems = computed(() => this.store.visibleNotes().slice(0, 48));   // cap DOM size; paginate/infinite-load the rest
}
```

```html
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
  @for (note of pageItems(); track note.id) {
    <app-note-card [note]="note" (favoriteToggled)="store.toggleFavorite($event)" />
  }
</div>
```

- `NoteCard` is OnPush (default), so toggling one favourite re-renders **one** card (its `note` input is the only new object).
- `visibleNotes` is a `computed`; typing in search recomputes once per change, not per card.

## 9. Common mistakes

| Mistake | Fix |
|---|---|
| `[items]="list().filter(...)"` in templates | `computed` |
| Recreating all objects on every update (`list.map(n => ({ ...n }))`) | only replace what changed |
| `ChangeDetectionStrategy.Eager` left on heavy components after upgrade | migrate to OnPush |
| Rendering 1,000 cards | paginate or virtualise |
| Images without dimensions | `NgOptimizedImage` or explicit `width`/`height` |
| Premature micro-optimisation | measure first ([note 3](03-measuring.md)) |

## 10. Testing

Performance is measured, not unit-tested — but you can assert behaviour that protects performance:

```typescript
it('keeps the same visibleNotes reference when unrelated state changes', () => {
  const store = TestBed.inject(NotesStore);
  store.setAll(MOCK_NOTES);
  const first = store.visibleNotes();
  store.categories.set([]);                  // categories feed categoryOptions, not visibleNotes
  expect(store.visibleNotes()).toBe(first);  // no recomputation → children receive the same array
});
```

## 11. Practice questions

1. Find three template expressions in your practice app that should be `computed`.
2. Add `NgOptimizedImage` to the Memories gallery and compare CLS before/after.
3. When is virtual scrolling worse than pagination?

## 12. Interview questions

- **Q: Why are method calls in templates a performance smell?** They run on every check of the view and can't be memoised by Angular.
- **Q: How does `track` affect performance?** It lets Angular reuse DOM for items with the same key instead of destroying and recreating them.
- **Q: What does `NgOptimizedImage` enforce?** Explicit dimensions, lazy loading by default, priority hints for LCP images, and responsive `srcset` support.

## 13. Completion checklist

- [ ] No filtering/sorting in templates.
- [ ] Gallery uses `NgOptimizedImage`.
- [ ] One long list is virtualised or paginated.
