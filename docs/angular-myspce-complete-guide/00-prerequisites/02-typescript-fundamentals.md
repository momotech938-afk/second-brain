# TypeScript Fundamentals

Back to [Module 00](README.md) · Previous: [JavaScript for Angular](01-javascript-for-angular.md) · Next: [TypeScript advanced](03-typescript-advanced.md)

## 1. Learning objectives

- Annotate variables, parameters and return types.
- Model Myspce data with `interface` and `type`.
- Use union and literal types for states like `'todo' | 'done'`.
- Write classes with access modifiers, `readonly` and parameter properties.
- Understand what `strict` mode (and Angular's strict templates) protects you from.

## 2. Prerequisites

[JavaScript for Angular](01-javascript-for-angular.md).

## 3. What is it?

TypeScript adds a **type layer** to JavaScript. The compiler (`tsc`, run by the Angular CLI) checks your types and then removes them; the browser receives plain JavaScript. Angular 22 requires **TypeScript 6.0**.

## 4. Why does Angular need it?

- Templates are type-checked against your component (strict templates are on by default in v22), so `{{ note().titel }}` fails at build time.
- DI uses classes and typed tokens.
- HttpClient is generic: `http.get<Note[]>()` tells the rest of your code what comes back.

## 5. How it works

Types exist only at compile time. That has two consequences:

1. `http.get<Note>()` does **not** validate the server response. It's a promise you make to the compiler. (Runtime validation is covered in [Module 13](../13-http-and-api-integration/03-error-handling-retry.md).)
2. You can't do `if (x instanceof Note)` with an interface; interfaces don't exist at runtime.

## 6. Syntax reference

```typescript
// Primitives and arrays
let title: string = 'Angular Notes';
let count: number = 124;
let pinned: boolean = false;
let tags: string[] = ['angular'];
let pair: [string, number] = ['notes', 124];   // tuple

// Literal and union types
type TaskStatus = 'todo' | 'done';
type Theme = 'light' | 'dark' | 'system';
let status: TaskStatus = 'todo';

// Object shapes
interface Category {
  id: string;
  name: string;
  color: string;
}

interface Note {
  readonly id: string;          // can't be reassigned
  title: string;
  color?: string;               // optional: string | undefined
  category: Category | null;    // explicitly nullable
  tags: string[];
}

// Functions
function preview(text: string, max = 120): string {
  return text.length > max ? text.slice(0, max) + '…' : text;
}
const isDone = (t: { status: TaskStatus }): boolean => t.status === 'done';

// Enums: prefer union types in Angular apps (no runtime code, better tree-shaking)
```

### `interface` vs `type`

| | `interface` | `type` |
|---|---|---|
| Object shapes | ✅ | ✅ |
| Unions (`'a' \| 'b'`) | ❌ | ✅ |
| Extending | `interface A extends B` | `type A = B & { … }` |
| Declaration merging | ✅ (rarely wanted) | ❌ |

Rule of thumb: `interface` for data shapes (API models), `type` for unions, aliases and computed types.

### Classes

```typescript
class NotesApi {
  private readonly baseUrl = '/api/v1/notes';   // only inside the class, never reassigned
  protected cache = new Map<string, Note>();    // class + subclasses
  public lastError: string | null = null;       // anyone (default)
  #secret = 'true-private';                     // JS private field, enforced at runtime

  constructor(private readonly http: HttpLike) {}   // parameter property: declares + assigns

  url(id: string): string {
    return `${this.baseUrl}/${id}`;
  }
}
```

In modern Angular you rarely write constructors; you use `inject()` in field initializers instead ([Module 07](../07-services-and-dependency-injection/01-services-and-inject.md)). You still need to read constructor injection for older code.

### Inheritance

```typescript
abstract class BaseApi<T> {
  protected abstract resource: string;
  list(): string { return `/api/v1/${this.resource}`; }
}
class TasksApi extends BaseApi<Task> {
  protected resource = 'tasks';
}
```

In Angular, prefer **composition** (injecting helpers) over deep class inheritance.

## 7. Smallest working example

```typescript
type TaskStatus = 'todo' | 'done';
interface Task { id: string; title: string; status: TaskStatus }

function toggle(task: Task): Task {
  return { ...task, status: task.status === 'done' ? 'todo' : 'done' };
}

toggle({ id: 't1', title: 'Read docs', status: 'todo' });     // ✅
toggle({ id: 't1', title: 'Read docs', status: 'finished' }); // ❌ compile error
```

## 8. Step-by-step

1. `TaskStatus` allows exactly two strings.
2. `toggle` promises to return a `Task`; returning something without `title` would be an error.
3. Passing `'finished'` fails because it's not in the union — this is the bug you'd otherwise find in production.

## 9. Myspce example — the core models

See the full set in [MYSPCE_PROJECT_CONTEXT.md](../MYSPCE_PROJECT_CONTEXT.md#4-data-entities-used-in-examples). The important choices:

```typescript
// core/models/index.ts
export type Ulid = string;                          // documents intent: a 26-char ULID
export type CategoryModule = 'note' | 'credential' | 'task' | 'memory' | 'file' | 'event';

export interface CredentialSummary {
  id: Ulid;
  name: string;
  url: string | null;          // null = no URL (Office WiFi)
  username: string;
  category: Category | null;
  lastViewedAt: string | null; // ISO string from the API, not Date
}
```

- Dates are `string` because JSON has no date type; convert with the `date` pipe or `new Date()` when needed.
- `CredentialSummary` has **no password field**. The type itself documents the security rule "the list never returns secrets".

## 10. Advanced example — strictness that catches real bugs

```typescript
// tsconfig.json (generated by ng new)
{
  "compilerOptions": {
    "strict": true,                       // strictNullChecks, noImplicitAny, strictPropertyInitialization…
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  },
  "angularCompilerOptions": {
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true               // default in v22 even if omitted
  }
}
```

```typescript
function categoryName(note: Note): string {
  return note.category.name;      // ❌ 'note.category' is possibly 'null'
}
function categoryName2(note: Note): string {
  return note.category?.name ?? 'Uncategorised';   // ✅
}
```

## 11. Visual explanation

```text
 your .ts files ──► TypeScript compiler ──► type errors? ──yes──► build fails
                         │ no
                         ▼
                 plain .js (types erased) ──► bundler ──► browser
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Using `any` to silence errors | Use `unknown` and narrow, or write the right type |
| `title!: string` everywhere (definite assignment) | Initialise the field, or use `input.required()` in components |
| `enum Status { Todo, Done }` sent to an API that expects strings | Use string unions: `'todo' \| 'done'` |
| Typing API dates as `Date` | JSON gives strings; type them as `string` |
| Making every field optional "just in case" | Optional means "may really be missing"; be precise |

## 13. Debugging guide

- **`TS2322: Type 'X' is not assignable to type 'Y'`** — read the last line of the error first; it names the exact property that mismatches.
- **`TS2531: Object is possibly 'null'`** — narrow with `if`, `?.`, or fix the type if it can't really be null.
- **`TS7006: Parameter 'x' implicitly has an 'any' type`** — add a type annotation.
- Hover in VS Code: it shows the inferred type. Use it constantly.

## 14. Best practices

- Let TypeScript **infer** local variables; annotate function parameters, return types of public methods, and API models.
- Keep models in one place (`core/models`) and share them with the backend later (a shared types library).
- Never use `any` in committed code. `unknown` is the safe "I don't know yet".

## 15. Angular vs React

Same TypeScript. Differences: Angular checks **templates** too (React checks JSX via TypeScript directly), and Angular relies on **classes** for components and services where React uses functions.

## 16. Testing

Types are tested by the compiler. Make "type tests" part of your workflow: `ng build` (or `npx tsc --noEmit -p tsconfig.app.json`) in CI catches type regressions before unit tests run.

## 17. Practice questions

1. Model `Memory` from the design: title, date, location name, lat/lng, photos, category.
2. Why is `url: string | null` better than `url?: string` for credentials?
3. Convert `enum Priority { Low, Medium, High }` into a union type and explain the benefit.
4. Write a class `IdGenerator` with a private counter and a public `next()` method.

## 18. Interview questions

- **Q: `unknown` vs `any`?** A: Both accept anything, but `unknown` forces you to narrow before use; `any` turns off checking.
- **Q: Does `http.get<User>()` validate the response?** A: No. Types are erased; it only tells the compiler what to expect.
- **Q: What does `strictNullChecks` do?** A: Makes `null`/`undefined` separate types, so you must handle them explicitly.

## 19. Revision notes

| Use | Write |
|---|---|
| Data shape | `interface Note { … }` |
| Fixed set of strings | `type Status = 'a' \| 'b'` |
| Nullable | `T \| null` |
| Optional | `field?: T` |
| Read-only | `readonly id: string` |
| Private | `private` (TS) or `#field` (runtime) |

## 20. Completion checklist

- [ ] I modelled all v1 Myspce entities as interfaces.
- [ ] My `tsconfig` is strict and I understand each flag.
- [ ] I fixed three real compile errors by reading the message, not by adding `any`.
