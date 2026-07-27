# TypeScript Advanced (What Angular Actually Uses)

Back to [Module 00](README.md) · Previous: [TypeScript fundamentals](02-typescript-fundamentals.md) · Next: [Async and error handling](04-async-and-error-handling.md)

## 1. Learning objectives

- Write and read generics (`Paginated<T>`, `signal<T>()`, `http.get<T>()`).
- Narrow unions with `typeof`, `in`, discriminants and type guards.
- Use utility types to derive DTOs from models.
- Understand decorators well enough to read `@Component`, `@Service`, `@Pipe`.
- Use `satisfies`, `as const`, `keyof` and indexed access types.

## 2. Prerequisites

[TypeScript fundamentals](02-typescript-fundamentals.md).

## 3. What is it?

The type-level tools that let you write one function or type that works for many shapes, and let the compiler prove which shape you have at each point.

## 4. Why does Angular need it?

Almost every Angular API is generic: `signal<Note[]>()`, `input.required<Note>()`, `output<string>()`, `InjectionToken<string>`, `HttpClient.get<T>`, `FormControl<string>`, `resource<T, P>`. If generics confuse you, error messages will too.

## 5. How it works

A generic is a **type parameter**: a placeholder filled in when the function or type is used.

```typescript
function first<T>(items: T[]): T | undefined {
  return items[0];
}
const n = first([note1, note2]);   // T inferred as Note → n: Note | undefined
```

The compiler infers `T` from the argument. Constraints limit what `T` can be:

```typescript
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find(i => i.id === id);
}
```

## 6. Syntax reference

### Generics

```typescript
interface Paginated<T> {
  data: T[];
  meta: { total: number; page: number; limit: number };
}
type NotesPage = Paginated<Note>;

class Cache<K, V> {
  #map = new Map<K, V>();
  get(k: K): V | undefined { return this.#map.get(k); }
  set(k: K, v: V): void { this.#map.set(k, v); }
}
```

### Narrowing

```typescript
function label(value: string | number): string {
  if (typeof value === 'number') return value.toFixed(0);  // number here
  return value.trim();                                       // string here
}

// Discriminated union: the shared literal field `kind` tells them apart
type SearchResult =
  | { kind: 'note'; id: string; title: string }
  | { kind: 'credential'; id: string; name: string; url: string | null }
  | { kind: 'task'; id: string; title: string; dueAt: string | null };

function resultTitle(r: SearchResult): string {
  switch (r.kind) {
    case 'note': return r.title;
    case 'credential': return r.name;
    case 'task': return r.title;
    default: {
      const never: never = r;   // compile error if a new kind is added and not handled
      return never;
    }
  }
}

// `in` narrowing
if ('url' in r) { r.url; }

// Custom type guard
function isApiError(e: unknown): e is { statusCode: number; message: string } {
  return typeof e === 'object' && e !== null && 'statusCode' in e && 'message' in e;
}
```

Angular's `@switch` + `@default never;` (v21.2+) gives the same exhaustiveness check in templates ([Module 03](../03-template-control-flow/01-if-for-switch.md)).

### Utility types

| Type | Gives | Myspce use |
|---|---|---|
| `Partial<T>` | all fields optional | `UpdateNoteDto = Partial<CreateNoteDto>` |
| `Required<T>` | all fields required | settings with defaults applied |
| `Readonly<T>` | all fields readonly | state exposed from a store |
| `Pick<T, K>` | only keys K | `Pick<Note, 'id' \| 'title'>` for search results |
| `Omit<T, K>` | all but K | `CreateNoteDto = Omit<Note, 'id' \| 'createdAt' \| 'updatedAt'>` |
| `Record<K, V>` | object with keys K | `Record<CategoryModule, Category[]>` |
| `ReturnType<F>` | a function's return type | typing a factory |
| `Awaited<T>` | unwraps a Promise | `Awaited<ReturnType<typeof loadUser>>` |
| `NonNullable<T>` | removes null/undefined | after a null check |

### `keyof`, indexed access, `as const`, `satisfies`

```typescript
type NoteKey = keyof Note;               // 'id' | 'title' | …
type NoteTitle = Note['title'];          // string

const SORT_OPTIONS = ['-updatedAt', 'updatedAt', 'title'] as const;
type SortOption = (typeof SORT_OPTIONS)[number];   // '-updatedAt' | 'updatedAt' | 'title'

const NAV = [
  { label: 'Dashboard', route: '/dashboard', icon: 'layout-dashboard' },
  { label: 'Notes', route: '/notes', icon: 'notebook' },
] satisfies { label: string; route: string; icon: string }[];
// `satisfies` checks the shape but keeps the precise literal types
```

### Decorators

A decorator is a function applied to a class or member with `@`. Angular's decorators attach **metadata** that the compiler reads at build time.

```typescript
@Component({ selector: 'app-note-card', template: '…' })
export class NoteCard {}
```

At build time, the Angular compiler turns that metadata into a static definition (`ɵcmp`) on the class containing compiled template instructions. You never write that yourself; you just need to know a decorator is "configuration attached to a class".

Angular's own decorators you'll use: `@Component`, `@Directive`, `@Pipe`, `@Service` (v22), `@Injectable`, and legacy `@Input`, `@Output`, `@HostListener`, `@HostBinding`, `@ViewChild`, `@NgModule`.

## 7. Smallest working example

```typescript
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map(i => i[key]);
}
const titles = pluck(notes, 'title');   // string[]
pluck(notes, 'titel');                  // ❌ '"titel"' is not assignable to keyof Note
```

## 8. Step-by-step

1. `K extends keyof T` restricts `key` to real property names of `T`.
2. `T[K]` is the type of that property, so `titles` is `string[]`.
3. A typo is caught at compile time.

## 9. Myspce example — typed API DTOs derived from models

```typescript
// features/notes/data-access/notes.models.ts
import { Note } from '../../../core/models';

export type CreateNoteDto = Pick<Note, 'title' | 'color' | 'tags'> & { categoryId: string | null; content: unknown };
export type UpdateNoteDto = Partial<CreateNoteDto> & { pinned?: boolean };

export interface NoteQuery {
  page?: number;
  limit?: number;
  q?: string;
  category?: string;
  favorite?: boolean;
  sort?: '-updatedAt' | 'updatedAt' | 'title';
}
```

Deriving from `Note` means a renamed field updates every DTO automatically.

## 10. Advanced example — a generic store helper

```typescript
// shared/utils/entity.ts
export function upsert<T extends { id: string }>(list: readonly T[], item: T): T[] {
  const i = list.findIndex(x => x.id === item.id);
  return i === -1 ? [item, ...list] : list.with(i, item);   // `with` = immutable replace (ES2023)
}

export function patch<T extends { id: string }>(list: readonly T[], id: string, changes: Partial<T>): T[] {
  return list.map(x => (x.id === id ? { ...x, ...changes } : x));
}
```

Used by every store: `this.notes.update(list => patch(list, id, { pinned: true }))`.

## 11. Visual explanation

```text
SearchResult (union)
 ├─ kind: 'note'        → title
 ├─ kind: 'credential'  → name, url
 └─ kind: 'task'        → title, dueAt
switch (r.kind) narrows r to exactly one branch
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `as Note` to force a type | Narrow properly or fix the source; `as` hides bugs |
| `function f<T>(x: T) { x.id }` | Add a constraint: `<T extends { id: string }>` |
| Forgetting `default: never` checks | Add exhaustive checks on discriminated unions |
| Using `Record<string, X>` and assuming keys exist | Values may be `undefined`; enable `noUncheckedIndexedAccess` if you want TS to enforce it |

## 13. Debugging guide

- Long generic errors: scroll to the **bottom**; the last "Type X is not assignable to Y" is the real cause.
- Use a temporary `type Debug = typeof myValue;` and hover it to see what TypeScript thinks.
- `Type instantiation is excessively deep` — simplify recursive types; don't over-engineer.

## 14. Best practices

- Generics should make call sites simpler. If callers must write `<A, B, C>` manually, rethink the API.
- Prefer discriminated unions for "one of several shapes" (search results, resource states).
- Use `satisfies` for config objects (routes, nav, theme tokens) so typos are caught but literals stay precise.

## 15. Angular vs React

React uses the same TypeScript features (`useState<Note[]>`, `React.FC<Props>`). Angular leans more on decorators and generic DI tokens; React leans on generic hooks.

## 16. Testing

Use `expectTypeOf` from Vitest for type-level tests on shared helpers:

```typescript
import { expectTypeOf, it } from 'vitest';
it('pluck returns the property type', () => {
  expectTypeOf(pluck(notes, 'pinned')).toEqualTypeOf<boolean[]>();
});
```

## 17. Practice questions

1. Write `groupBy<T, K extends string>(items: T[], key: (t: T) => K): Record<K, T[]>`.
2. Model resource state as a discriminated union: idle, loading, success (with data), error (with message).
3. Derive `CreateTaskDto` from `Task` with `Omit`.
4. What does `as const` change about `['a', 'b']`?

## 18. Interview questions

- **Q: What's a discriminated union?** A: A union of object types that share a literal "tag" field; checking the tag narrows to one member.
- **Q: `satisfies` vs a type annotation?** A: An annotation widens the value to the annotated type; `satisfies` checks compatibility but keeps the inferred, narrower type.
- **Q: What does `@Component` do?** A: Attaches metadata the Angular compiler uses to generate the component's definition and template code.

## 19. Revision notes

| Tool | Purpose |
|---|---|
| `<T>` | placeholder type |
| `T extends X` | constraint |
| `keyof T` / `T[K]` | keys / value type |
| `Partial/Pick/Omit/Record` | derive types |
| Discriminated union + `never` | exhaustive handling |
| `satisfies` | check without widening |

## 20. Completion checklist

- [ ] I wrote `upsert` and `patch` with generics.
- [ ] I modelled search results as a discriminated union with an exhaustive switch.
- [ ] I can explain what `@Component` does to a class.
