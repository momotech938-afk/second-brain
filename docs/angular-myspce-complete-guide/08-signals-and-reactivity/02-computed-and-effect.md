# `computed()` and `effect()`

Back to [Module 08](README.md) · Previous: [Signal basics](01-signal-basics.md) · Next: [`linkedSignal()`](03-linked-signal.md)

## 1. Learning objectives

- Derive state with `computed()` and understand dynamic dependency tracking.
- Use `untracked()` correctly.
- Know exactly what `effect()` is for, when it runs, and how to clean up.
- Avoid the classic effect anti-patterns.

## 2. Prerequisites

[Signal basics](01-signal-basics.md).

## 3. What is it?

- **`computed(fn)`** — a read-only signal whose value is `fn()`. It remembers which signals `fn` read and recomputes only when one of them changes. Lazy and cached.
- **`effect(fn)`** — runs `fn` whenever a signal it reads changes. It produces **side effects** (logging, storage, imperative DOM/library calls). It returns nothing you can read.

## 4. Why does Angular need them?

`computed` keeps derived data (filtered lists, counts, labels) always consistent without manual syncing. `effect` bridges the reactive world to non-reactive APIs (localStorage, `document.title`, a chart library, analytics).

## 5. How they work internally

### `computed`

1. First read runs `fn` and records dependencies **that were actually read**.
2. When a dependency changes, the computed is marked dirty (no work yet).
3. Next read: if dirty, re-run `fn`, re-record dependencies, and compare the new value with `equal` (default `Object.is`). If equal, its own consumers are not notified.

Dependencies are **dynamic**:

```typescript
const label = computed(() => (this.showCount() ? `${this.title()} (${this.count()})` : this.title()));
// while showCount() is false, changes to count() don't trigger recomputation
```

### `effect`

- Runs at least once, then again after any read signal changes.
- Runs **asynchronously**, scheduled by Angular (not synchronously inside `set`). Multiple `set` calls in a row cause **one** effect run.
- **Component effects** (created in a component/directive) run during change detection, before that component's view is checked; **root effects** (created in a root service) run as part of the app's change-detection cycle too, before component effects.
- Destroyed automatically with the injector that created it (component destroyed → its effects stop).
- Must be created in an injection context, or pass `{ injector }`.
- Writing signals inside effects is allowed (since v19) but usually a smell.

## 6. Syntax and API reference

```typescript
import { computed, effect, untracked, EffectRef, Injector } from '@angular/core';

const total = computed(() => this.items().length);
const sorted = computed(() => this.items().toSorted(byDate), { equal: sameIds, debugName: 'sorted' });

const ref: EffectRef = effect(onCleanup => {
  const id = setInterval(() => this.tick.set(new Date()), 60_000);
  onCleanup(() => clearInterval(id));   // runs before the next run and on destroy
});
ref.destroy();                          // stop manually

effect(() => console.log(this.query()), { injector: this.injector });   // outside a constructor
effect(() => …, { manualCleanup: true });                               // survives its injector — rare

untracked(() => this.user());           // read without creating a dependency
```

## 7. Smallest working example

```typescript
// title-sync.ts
@Component({ selector: 'app-root', template: `<input [value]="title()" (input)="title.set($any($event.target).value)" />` })
export class App {
  title = signal('MySpace');
  private upper = computed(() => this.title().trim().toUpperCase() || 'MYSPACE');

  constructor() {
    effect(() => (document.title = this.upper()));   // side effect: sync to the outside world
  }
}
```

## 8. Step-by-step

1. `upper` derives from `title` (pure, cached).
2. The effect reads `upper`; whenever it changes, `document.title` is updated.
3. Typing fast → many `set` calls → the effect runs once per change-detection cycle, not per keystroke.

## 9. Myspce real-world example — Notes filtering + dashboard clock

```typescript
// features/notes/data-access/notes-store.ts (continued)
readonly visibleNotes = computed(() => {
  const cat = this.category();
  const q = this.query().trim().toLowerCase();
  return this._notes()
    .filter(n => cat === 'all' || n.category?.slug === cat)
    .filter(n => !q || n.title.toLowerCase().includes(q) || n.preview.toLowerCase().includes(q))
    .toSorted((a, b) => Number(b.pinned) - Number(a.pinned) || b.updatedAt.localeCompare(a.updatedAt));
});

readonly categories = signal<Category[]>([]);   // filled from CategoriesApi (GET /categories?module=note)

readonly categoryOptions = computed(() => {
  const counts = new Map<string, number>();
  for (const n of this._notes()) {
    const slug = n.category?.slug ?? 'uncategorised';
    counts.set(slug, (counts.get(slug) ?? 0) + 1);
  }
  return [
    { value: 'all', label: 'All', count: this._notes().length },
    ...this.categories().map(c => ({ value: c.slug, label: c.name, count: counts.get(c.slug) ?? 0 })),
  ];
});
```

```typescript
// features/dashboard/pages/dashboard-page.ts — clock for the greeting header
export class DashboardPage {
  protected now = signal(new Date());

  constructor() {
    effect(onCleanup => {
      const id = setInterval(() => this.now.set(new Date()), 30_000);
      onCleanup(() => clearInterval(id));
    });
  }
}
```

The clock effect has no signal dependencies, so it runs once and its cleanup runs when the dashboard is destroyed. (A plain `setInterval` in the constructor plus `inject(DestroyRef).onDestroy(...)` is equally fine and arguably clearer.)

### Valid effect uses in Myspce

| Effect | Why it's legitimate |
|---|---|
| Persist theme mode to localStorage | outside world (storage) |
| Toggle `.dark` on `<html>` | outside the component's template |
| Update `document.title` with unread count | outside world |
| Re-center the Leaflet map when the selected memory changes | imperative third-party API |
| Send analytics when a page filter changes | external system |

## 10. Advanced example — what NOT to do with effects

```typescript
// ❌ 1. Deriving state
effect(() => this.fullName.set(`${this.first()} ${this.last()}`));
// ✅ fullName = computed(() => `${this.first()} ${this.last()}`);

// ❌ 2. Fetching data
effect(() => this.http.get(`/api/v1/notes?category=${this.category()}`).subscribe(r => this.notes.set(r.data)));
// races (old response can overwrite new), no loading/error state, no cancellation
// ✅ notes = httpResource(() => `/api/v1/notes?category=${this.category()}`);   (next notes)

// ❌ 3. Resetting state when another signal changes
effect(() => { this.category(); this.page.set(1); });
// ✅ page = linkedSignal(() => { this.category(); return 1; });

// ❌ 4. Reading a signal you don't want as a dependency
effect(() => this.log.write(`query=${this.query()} user=${this.user().name}`));
// re-runs when user changes too
// ✅ effect(() => { const q = this.query(); untracked(() => this.log.write(`query=${q} user=${this.user().name}`)); });
```

Rule of thumb: **if the effect writes a signal, you probably wanted `computed`, `linkedSignal` or a resource.**

## 11. Visual explanation

```text
signals ──► computed (pure, lazy, cached) ──► templates
   │
   └──► effect (side effects, scheduled) ──► localStorage / document / library
```

## 12. Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Side effects in `computed` (HTTP, `set`) | `NG0600` or unpredictable behaviour | keep `computed` pure |
| `effect` in a method/after `await` | `NG0203` | create in constructor/field, or pass `{ injector }` |
| Effect writing a signal it reads | infinite loop or "effect is writing to a signal it depends on" warnings | `untracked` or restructure |
| Expecting effect to run synchronously after `set` | test sees old value | `await fixture.whenStable()` / `TestBed.tick()` |
| Heavy work in `computed` read by many views | slow | it's cached — fine; but avoid creating new arrays when nothing changed (use `equal`) |

## 13. Debugging guide

- Add `debugName` and inspect the graph in Angular DevTools.
- Temporarily log inside a `computed` to see how often it recomputes.
- If an effect never runs: was it created in an injection context? Was the injector destroyed? Does it read any signal at all?

## 14. Best practices

- Prefer `computed` for everything derivable.
- Effects only for syncing to non-signal APIs; keep them small and put them in constructors.
- Use `onCleanup` for timers/subscriptions inside effects.
- Never fetch data in effects — use resources.

## 15. Angular vs React

| React | Angular |
|---|---|
| `useMemo(fn, deps)` | `computed(fn)` — deps tracked automatically |
| `useEffect(fn, deps)` for derived state | **don't**: `computed` |
| `useEffect` for fetching | **don't**: `httpResource` / `resource` |
| `useEffect` for subscriptions/DOM sync | `effect` with `onCleanup`, or `afterNextRender` |

## 16. Testing

```typescript
it('recomputes visibleNotes when category changes', () => {
  const store = TestBed.inject(NotesStore);
  store.setAll(MOCK_NOTES);
  store.category.set('work');
  expect(store.visibleNotes().every(n => n.category?.slug === 'work')).toBe(true);
});

it('persists theme mode via effect', async () => {
  const setItem = vi.spyOn(Storage.prototype, 'setItem');
  const theme = TestBed.inject(ThemeStore);
  theme.mode.set('dark');
  TestBed.tick();                       // flush effects
  expect(setItem).toHaveBeenCalledWith('myspce.theme', 'dark');
});
```

`TestBed.tick()` runs pending effects and change detection synchronously in tests (it replaced the older `TestBed.flushEffects()`).

## 17. Practice questions

1. Write `pinnedCount`, `favoriteCount` and `hasActiveFilter` computeds for `NotesStore`.
2. Convert three effects in a sample codebase to `computed` / `linkedSignal` / `httpResource`.
3. Why does an effect created in a root service run even when no component is on screen?

## 18. Interview questions

- **Beginner — `computed` vs `effect`?** `computed` returns a derived value; `effect` performs side effects and returns nothing.
- **Intermediate — When does an effect run?** Once initially, then asynchronously after any signal it read changes, scheduled by Angular's change detection; cleanup runs before the next run and on destroy.
- **Advanced — Why are effects not for state propagation?** They run asynchronously, can cause extra render cycles and inconsistent intermediate states, and hide data flow; `computed`/`linkedSignal` express it synchronously and glitch-free.

## 19. Revision notes

`computed` = pure, lazy, cached, dynamic deps · `effect` = side effects only, scheduled, auto-destroyed, `onCleanup` · `untracked` to read without tracking · effect writing signals = smell.

## 20. Completion checklist

- [ ] `visibleNotes` and `categoryOptions` computeds work.
- [ ] My only effects sync to storage/DOM/libraries.
- [ ] I tested an effect with `TestBed.tick()`.
