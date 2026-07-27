# JavaScript for Angular

Back to [Module 00](README.md) · Next: [TypeScript fundamentals](02-typescript-fundamentals.md)

## 1. Learning objectives

- Use `const`/`let`, arrow functions and `this` correctly inside classes.
- Use ES modules (`import`/`export`) the way Angular files do.
- Destructure, spread and copy objects and arrays without mutating them.
- Use `map`, `filter`, `find`, `some`, `reduce` to transform lists of Myspce data.
- Use optional chaining `?.` and nullish coalescing `??` safely.

## 2. Prerequisites

Basic JavaScript syntax (variables, functions, objects, arrays).

## 3. What is it?

The subset of modern JavaScript (ES2022+) that every Angular file uses. Angular code is TypeScript, and TypeScript is JavaScript with types, so every runtime behaviour below is identical in Angular.

## 4. Why does Angular need it?

- Components and services are **classes**, so you need `this` and class fields.
- Every file is an **ES module**; nothing is global.
- Signals and OnPush change detection depend on **immutable updates**: Angular notices a new array, not a changed one.

## 5. How it works

### `this` in classes

`this` is decided by **how a function is called**, not where it is written. Arrow functions don't have their own `this`; they use the one from where they were defined.

```javascript
class NotesPage {
  title = 'Notes';

  logLater() {
    setTimeout(function () {
      console.log(this.title);   // undefined: `this` is not the class here
    }, 100);

    setTimeout(() => {
      console.log(this.title);   // 'Notes': arrow function keeps `this`
    }, 100);
  }
}
```

Angular templates call methods on the component instance, so `(click)="save()"` always has the right `this`. The problem only appears when **you** pass a method as a callback (`setTimeout(this.save)`, `array.map(this.format)`).

### ES modules

```javascript
// shared/format.js
export function formatTitle(t) { return t.trim(); }
export const MAX_TITLE = 200;
export default class Formatter {}

// features/notes/page.js
import Formatter, { formatTitle, MAX_TITLE } from '../../shared/format.js';
import * as fmt from '../../shared/format.js';     // namespace import
const lazy = await import('./heavy.js');           // dynamic import = lazy loading
```

Angular's `loadComponent: () => import('./note-detail')` is exactly a dynamic import. The build tool turns it into a separate file that loads on demand.

### Copying vs mutating

Objects and arrays are **references**. Assigning them doesn't copy them.

```javascript
const a = { title: 'Old' };
const b = a;            // same object
b.title = 'New';
console.log(a.title);   // 'New' — you changed both
```

## 6. Syntax reference

| Feature | Example |
|---|---|
| Destructuring | `const { title, tags = [] } = note;` · `const [first, ...rest] = notes;` |
| Spread (copy) | `const copy = { ...note, title: 'New' };` · `const list = [...notes, newNote];` |
| Optional chaining | `note.category?.name` → `undefined` if category is null/undefined |
| Nullish coalescing | `note.color ?? '#fff'` → default only for `null`/`undefined` (not `''` or `0`) |
| Template literal | `` `/api/v1/notes/${id}` `` |
| Shorthand properties | `const payload = { title, tags };` |
| Array methods | `map`, `filter`, `find`, `findIndex`, `some`, `every`, `reduce`, `sort` (mutates!), `toSorted` (copy) |
| `structuredClone` | deep copy: `const deep = structuredClone(note);` |

## 7. Smallest working example

```javascript
const notes = [
  { id: 'n1', title: 'Angular Notes', pinned: false },
  { id: 'n2', title: 'AWS Deployment', pinned: true },
];

const pinnedFirst = notes.toSorted((a, b) => Number(b.pinned) - Number(a.pinned));
console.log(pinnedFirst.map(n => n.title)); // ['AWS Deployment', 'Angular Notes']
```

## 8. Step-by-step

1. `toSorted` returns a **new** array (ES2023), so `notes` is untouched.
2. `Number(true)` is 1, `Number(false)` is 0, so pinned notes sort first.
3. `map` returns a new array of titles.

## 9. Myspce example — immutable updates for the Notes list

```javascript
// Add a note to the top
const addNote = (list, note) => [note, ...list];

// Update one note by id (toggle favorite)
const toggleFavorite = (list, id) =>
  list.map(n => (n.id === id ? { ...n, isFavorite: !n.isFavorite } : n));

// Remove a note (soft-delete moves it to Trash in the UI)
const removeNote = (list, id) => list.filter(n => n.id !== id);

// Count notes per category for the chips
const countByCategory = list =>
  list.reduce((acc, n) => {
    const key = n.category?.slug ?? 'uncategorised';
    acc[key] = (acc[key] ?? 0) + 1;
    return acc;
  }, {});
```

These four functions are the shape of every store update you'll write in Module 15.

## 10. Advanced example — grouping tasks into Today / Tomorrow / This week

```javascript
function groupTasks(tasks, now = new Date()) {
  const startOfDay = d => new Date(d.getFullYear(), d.getMonth(), d.getDate()).getTime();
  const today = startOfDay(now);
  const day = 24 * 60 * 60 * 1000;

  return tasks.reduce(
    (groups, task) => {
      if (!task.dueAt) return { ...groups, later: [...groups.later, task] };
      const due = startOfDay(new Date(task.dueAt));
      if (due === today) return { ...groups, today: [...groups.today, task] };
      if (due === today + day) return { ...groups, tomorrow: [...groups.tomorrow, task] };
      if (due < today + 7 * day) return { ...groups, week: [...groups.week, task] };
      return { ...groups, later: [...groups.later, task] };
    },
    { today: [], tomorrow: [], week: [], later: [] },
  );
}
```

## 11. Visual explanation

```text
Mutation                          Immutable update
--------                          ----------------
notes ──► [n1, n2]                notes ──► [n1, n2]
notes.push(n3)                    next = [...notes, n3]
notes ──► [n1, n2, n3]            notes ──► [n1, n2]       (old, unchanged)
(same reference: Angular          next  ──► [n1, n2, n3]   (new reference: Angular
 OnPush/signals may not notice)                              sees the change)
```

## 12. Common mistakes

| Wrong | Why it fails | Right |
|---|---|---|
| `notes.push(n)` then `signal.set(notes)` | Same reference; `signal.set` sees equal value and skips the update | `signal.update(list => [...list, n])` |
| `const copy = { ...note }; copy.category.name = 'x'` | Spread is **shallow**; `category` is still shared | `{ ...note, category: { ...note.category, name: 'x' } }` |
| `note.color \|\| 'white'` when `color` is `''` | `\|\|` treats `''` as missing | `note.color ?? 'white'` if `''` is a valid value |
| `notes.sort(...)` on state | `sort` mutates in place | `notes.toSorted(...)` or `[...notes].sort(...)` |
| `setTimeout(this.save, 0)` | loses `this` | `setTimeout(() => this.save(), 0)` |

## 13. Debugging guide

- **`Cannot read properties of undefined (reading 'name')`** — you read a property of something that doesn't exist yet. Use `?.` or check for loading state first.
- **UI not updating after a change** — check if you mutated an array/object instead of creating a new one.
- **`this is undefined`** — a method was passed as a callback. Wrap it in an arrow function.

## 14. Best practices

- Default to `const`; use `let` only when you reassign.
- Treat state as immutable everywhere; it makes Angular's change detection predictable.
- Prefer `toSorted`, `toReversed`, `with` (ES2023 copying methods) over mutating ones.

## 15. Angular vs React

Identical rules. React's `setState` also needs a new reference. If you already write `setNotes(prev => [...prev, n])`, you'll write `notes.update(prev => [...prev, n])` in Angular.

## 16. Testing

```javascript
import { describe, it, expect } from 'vitest';
describe('toggleFavorite', () => {
  it('returns a new array and does not mutate the input', () => {
    const list = [{ id: 'n1', isFavorite: false }];
    const next = toggleFavorite(list, 'n1');
    expect(next).not.toBe(list);
    expect(list[0].isFavorite).toBe(false);
    expect(next[0].isFavorite).toBe(true);
  });
});
```

## 17. Practice questions

1. What does `[...notes]` copy, and what does it not copy?
2. Rewrite `removeNote` using `findIndex` and `toSpliced` (ES2023).
3. Write `renameCategory(list, slug, newName)` that updates the category name on every note with that slug, immutably.
4. Why is `structuredClone` slower than spread, and when is it worth it?

## 18. Interview questions

- **Q: Difference between `??` and `||`?** A: `??` only falls back for `null`/`undefined`; `||` falls back for any falsy value (`0`, `''`, `false`).
- **Q: Why do arrow functions fix `this`?** A: They capture `this` lexically from the surrounding scope instead of getting it from the call site.
- **Q: What is a shallow copy?** A: A new top-level object whose nested objects are still shared references.

## 19. Revision notes

| Concept | One-liner |
|---|---|
| `this` | decided by the call site; arrows capture it |
| Modules | every file is a module; dynamic `import()` = lazy loading |
| Immutability | new reference for every change |
| `?.` / `??` | safe access / default for null-ish only |

## 20. Completion checklist

- [ ] I wrote the four Myspce list helpers from memory.
- [ ] I can explain why mutation breaks OnPush/signals.
- [ ] I fixed a `this`-is-undefined bug on purpose.
