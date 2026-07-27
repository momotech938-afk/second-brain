# `@if`, `@for` and `@switch`

Back to [Module 03](README.md) · Next: [Expressions and loading states](02-expressions-and-loading-states.md)

## 1. Learning objectives

- Conditionally render with `@if`, `@else if`, `@else`, and alias values with `as`.
- Render lists with `@for`, choose good `track` expressions, and use contextual variables.
- Handle empty lists with `@empty`.
- Switch between views with `@switch`, including compile-time exhaustiveness.
- Read and migrate legacy `*ngIf`, `*ngFor`, `ngSwitch`.

## 2. Prerequisites

[Data binding](../02-components-and-templates/02-data-binding.md).

## 3. What is it?

Built-in template blocks (since v17, the default) that add, remove or repeat parts of the DOM. They are part of the template language, so there's nothing to import.

## 4. Why does Angular need it?

Every Myspce screen is conditional: logged in or not, list empty or not, Timeline / Gallery / Map tabs, done vs pending tasks. Built-in blocks are faster and type-check better than the old structural directives.

## 5. How it works internally

- `@if` creates or **destroys** its content's DOM and component instances when the condition flips. Destroyed children run `ngOnDestroy`.
- `@for` keeps a map from each item's **track key** to its DOM. When the array changes, Angular compares keys: unchanged keys keep their DOM (and their component state), new keys get new DOM, removed keys are destroyed, moved keys are moved. That's why `track` matters.
- `@switch` renders exactly one `@case` (strict `===` comparison).

## 6. Syntax and API reference

### `@if`

```html
@if (user(); as u) {
  <span>{{ u.name }}</span>
} @else if (loading()) {
  <app-spinner />
} @else {
  <a routerLink="/login">Sign in</a>
}
```

`as` stores the condition's value so you don't call the signal repeatedly, and narrows its type (non-null inside the block).

### `@for`

```html
@for (note of notes(); track note.id; let i = $index, last = $last) {
  <app-note-card [note]="note" [class.mb-0]="last" />
} @empty {
  <app-empty-state title="No notes yet" />
}
```

| Variable | Value |
|---|---|
| `$index` | position (0-based) |
| `$first` / `$last` | boolean |
| `$even` / `$odd` | boolean |
| `$count` | length of the collection |

**`track` options:**

| Track by | When |
|---|---|
| `track item.id` | items have a stable unique id (Myspce: always) |
| `track $index` | static lists that never reorder (e.g., fixed skeleton placeholders) |
| `track item` | primitives (strings like tag names) |

`@for` accepts any iterable (arrays, `Map.values()`, `Set`).

### `@switch`

```html
@switch (view()) {
  @case ('timeline') { <app-memory-timeline [memories]="memories()" /> }
  @case ('gallery')  { <app-memory-gallery [memories]="memories()" /> }
  @case ('map')      { <app-memory-map [memories]="memories()" /> }
  @default never;
}
```

`@default never;` (v21.2+) makes the compiler check that every value of `view()`'s union type has a `@case`. In v22 you can also write `@default never(expr);` to check a nested discriminant, e.g. `@switch (widget.type) { … @default never(widget); }`.

### Legacy structural directives (read, don't write)

```html
<div *ngIf="user$ | async as u; else loading">{{ u.name }}</div>
<ng-template #loading>Loading…</ng-template>

<li *ngFor="let n of notes; trackBy: trackById; let i = index">{{ n.title }}</li>

<div [ngSwitch]="view">
  <app-timeline *ngSwitchCase="'timeline'" />
  <app-gallery *ngSwitchDefault />
</div>
```

Migrate automatically: `ng generate @angular/core:control-flow`.

## 7. Smallest working example

```typescript
// tag-list.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-tag-list',
  template: `
    <ul class="flex flex-wrap gap-2">
      @for (tag of tags(); track tag) {
        <li class="rounded-full bg-slate-100 px-2 text-xs dark:bg-slate-800">#{{ tag }}</li>
      } @empty {
        <li class="text-xs text-slate-400">No tags</li>
      }
    </ul>
  `,
})
export class TagList {
  tags = input<string[]>([]);
}
```

## 8. Step-by-step

1. `tags()` is read once per check; `@for` iterates the array.
2. `track tag` uses the string itself as the key (tags are unique per note).
3. `@empty` renders when the array has zero items.

## 9. Myspce real-world example — Tasks grouped by day

The Tasks screen shows **Today**, **Tomorrow**, **This Week** groups, each with checkbox rows, and a filter row (All / Today / Upcoming / Completed).

```typescript
// features/tasks/pages/tasks-page.ts
import { Component, computed, inject, signal } from '@angular/core';
import { TasksStore } from '../data-access/tasks-store';
import { TaskItem } from '../components/task-item';

type TaskFilter = 'all' | 'today' | 'upcoming' | 'completed';

@Component({
  selector: 'app-tasks-page',
  imports: [TaskItem],
  templateUrl: './tasks-page.html',
})
export class TasksPage {
  protected store = inject(TasksStore);
  protected filter = signal<TaskFilter>('all');
  protected readonly filters: { value: TaskFilter; label: string }[] = [
    { value: 'all', label: 'All' },
    { value: 'today', label: 'Today' },
    { value: 'upcoming', label: 'Upcoming' },
    { value: 'completed', label: 'Completed' },
  ];

  protected groups = computed(() => {
    const g = this.store.grouped();   // { today, tomorrow, week, later, completed }
    switch (this.filter()) {
      case 'today': return [{ title: 'Today', tasks: g.today }];
      case 'upcoming': return [{ title: 'Tomorrow', tasks: g.tomorrow }, { title: 'This Week', tasks: g.week }, { title: 'Later', tasks: g.later }];
      case 'completed': return [{ title: 'Completed', tasks: g.completed }];
      case 'all': return [{ title: 'Today', tasks: g.today }, { title: 'Tomorrow', tasks: g.tomorrow }, { title: 'This Week', tasks: g.week }];
    }
  });
}
```

```html
<!-- features/tasks/pages/tasks-page.html -->
<div role="tablist" aria-label="Task filter" class="mb-4 flex gap-2">
  @for (f of filters; track f.value) {
    <button type="button" role="tab" [attr.aria-selected]="filter() === f.value"
            class="rounded-lg px-3 py-1.5 text-sm"
            [class]="filter() === f.value ? 'bg-blue-600 text-white' : 'bg-slate-100 dark:bg-slate-800'"
            (click)="filter.set(f.value)">
      {{ f.label }}
    </button>
  }
</div>

@for (group of groups(); track group.title) {
  @if (group.tasks.length) {
    <section class="mb-6" [attr.aria-labelledby]="'grp-' + $index">
      <h2 [id]="'grp-' + $index" class="mb-2 font-semibold">{{ group.title }} <span class="text-slate-400">({{ group.tasks.length }})</span></h2>
      <ul class="divide-y divide-slate-100 dark:divide-slate-800">
        @for (task of group.tasks; track task.id) {
          <app-task-item [task]="task" (toggled)="store.toggle($event)" />
        }
      </ul>
    </section>
  }
} @empty {
  <p>No tasks.</p>
}
```

Because tasks are tracked by `id`, ticking one checkbox moves that row between groups without recreating the others.

## 10. Advanced example — exhaustive switch over a discriminated union

Dashboard widgets come from admin config as a union:

```typescript
type Widget =
  | { type: 'stats'; id: string }
  | { type: 'recent_activity'; id: string; limit: number }
  | { type: 'todays_tasks'; id: string }
  | { type: 'week_progress'; id: string };
```

```html
@for (w of widgets(); track w.id) {
  @switch (w.type) {
    @case ('stats') { <app-stats-widget /> }
    @case ('recent_activity') { <app-activity-widget [limit]="w.limit" /> }
    @case ('todays_tasks') { <app-today-widget /> }
    @case ('week_progress') { <app-week-progress-widget /> }
    @default never(w);
  }
}
```

Inside `@case ('recent_activity')`, `w` is narrowed so `w.limit` type-checks. If the admin model gains `'quote'` and you forget a case, **the build fails**. (A registry with dynamic components is the alternative for many widgets — [Module 16](../16-advanced-components/03-dynamic-components.md).)

## 11. Visual explanation

```text
notes before: [A, B, C]       notes after: [B, C, D]   (track note.id)
DOM:          A  B  C   →     A destroyed, B kept, C kept, D created
Without good tracking (e.g. track $index on a reorderable list):
              every row's content rebinds; component state (open menus, inputs) lands on the wrong row
```

## 12. Common mistakes

| Wrong | Problem | Right |
|---|---|---|
| `track $index` on a sortable/filterable list | wrong DOM reused, lost state, animations glitch | `track note.id` |
| `@for (n of notes)` with `notes` a signal | iterating the function | `notes()` |
| Calling `filter()` inside the `@for` expression | new array every check, more work | `computed()` |
| Comparing objects in `@case` | `===` reference comparison | switch on a string/number field |
| Nested `@if` pyramids | unreadable | `@if ... @else if ...` or a `computed` state value |

## 13. Debugging guide

| Message | Fix |
|---|---|
| `@for loop must have a "track" expression` | add `track` |
| `NG0955: The provided track expression resulted in duplicated keys` | ids aren't unique — fix data or track a unique field |
| `NG0956: … collection was re-created … expensive DOM operations` | your track expression is unstable (e.g., tracking a new object each time) |
| `Property 'x' does not exist on type 'never'` inside a case | the union has no member with that case value |

## 14. Best practices

- Always `track` by a stable id.
- Derive filtered/sorted lists in `computed()`, not in templates.
- Use `as` aliases in `@if` to narrow and avoid repeated reads.
- Use `@default never` for unions you control.

## 15. Angular vs React

| React | Angular |
|---|---|
| `{cond && <X/>}` / ternary | `@if (cond) { <x/> } @else { … }` |
| `list.map(i => <X key={i.id}/>)` | `@for (i of list(); track i.id) { <x/> }` |
| empty check by hand | `@empty` block |
| `switch` in a helper or object map | `@switch` + `@case` |

## 16. Testing

```typescript
it('shows the empty state when there are no tags', async () => {
  const fixture = TestBed.createComponent(TagList);
  fixture.componentRef.setInput('tags', []);
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).toContain('No tags');
});
```

## 17. Practice questions

1. Build the Memories tabs (Timeline / Gallery / Map) with `@switch` and `@default never;`.
2. Render the Credentials list with an `@empty` state and alternating row backgrounds using `$even`.
3. Why is `track $index` acceptable for a skeleton loader of 6 grey boxes?

## 18. Interview questions

- **Beginner — Why does `@for` require `track`?** So Angular can match items to existing DOM and avoid recreating it.
- **Intermediate — What's destroyed when an `@if` becomes false?** All DOM and component/directive instances inside the block; their `ngOnDestroy` runs.
- **Advanced — How does `@default never` work?** The compiler narrows the switch expression through each `@case`; if the remaining type isn't `never`, it's a compile error.

## 19. Revision notes

| Block | Key point |
|---|---|
| `@if (x; as v)` | narrows + aliases |
| `@for (i of list(); track i.id)` | `track` required; `$index $first $last $even $odd $count` |
| `@empty` | zero-item fallback |
| `@switch` / `@case` / `@default never;` | strict equality, exhaustiveness |

## 20. Completion checklist

- [ ] Tasks page groups work with filters.
- [ ] Memories tabs use an exhaustive switch.
- [ ] I migrated one legacy `*ngFor` to `@for`.
