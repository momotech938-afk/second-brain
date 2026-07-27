# Angular vs React

Back to [Module 01](README.md) · Previous: [Configuration and builds](04-configuration-and-builds.md) · Next module: [02 Components](../02-components-and-templates/README.md)

You already know React. This note maps what you know to Angular, and flags where the mapping is misleading.

## 1. Learning objectives

- Translate React concepts to their Angular equivalents.
- Understand the three real mindset shifts: templates, DI and fine-grained reactivity.
- Avoid carrying React habits that hurt in Angular.

## 2. Concept map

| React | Angular 22 | Notes |
|---|---|---|
| Function component | Class with `@Component` | Angular components are instances that live as long as they're rendered |
| JSX | HTML template (`.html` or inline) | Compiled ahead of time; no JS expressions beyond template syntax |
| Props | `input()` / `input.required()` | Signals: read with `note()` |
| Callback props (`onSave`) | `output()` | Parent binds `(save)="…"` |
| Controlled component (`value` + `onChange`) | `model()` + `[(value)]` | Built-in two-way binding |
| `children` | `<ng-content>` / named slots | Content projection |
| `useState` | `signal()` | Writable; `set`/`update` |
| `useMemo` | `computed()` | Auto-tracks dependencies; no deps array |
| `useEffect` | `effect()`, lifecycle hooks, `afterNextRender()` | **Different**: see below |
| `useRef` (DOM) | `viewChild()` / template ref `#el` | |
| `useRef` (mutable value) | a plain class field | |
| Context | Dependency injection | Hierarchical, typed, lazy |
| Custom hooks | Services + functions using `inject()` | `injectX()` helper functions are common |
| React Router | `@angular/router` | Built in; guards and resolvers included |
| React Query / SWR | `httpResource()` / `resource()` / stores | Built in since v19, stable in v22 |
| React Hook Form / Formik | Signal Forms or Reactive Forms | Built in |
| `fetch`/axios | `HttpClient` + interceptors | Observables; interceptors built in |
| Redux / Zustand | Signal stores, NgRx | Start with signal services |
| `React.lazy` + `Suspense` | `loadComponent`, `@defer` | `@defer` also handles placeholders/loading/error |
| Error boundary | `ErrorHandler`; `@boundary` is being worked on | No template-level boundary in 22.0 |
| Vite | `@angular/build` (esbuild + Vite) | Managed by the CLI |
| Jest/Vitest + RTL | Vitest + TestBed (+ Angular Testing Library) | |
| `key` in lists | `track` in `@for` | **Required** in Angular |
| `className={cx(...)}` | `[class]="{ … }"`, `[class.x]` | |
| `dangerouslySetInnerHTML` | `[innerHTML]` (sanitised) | Angular sanitises by default |

## 3. The three real mindset shifts

### 3.1 Templates are compiled, not JavaScript

In React, JSX is JavaScript: you can `map`, call any function, use any expression. Angular templates are a **restricted language** compiled at build time:

```html
<!-- React habit (doesn't work) -->
{notes.filter(n => n.pinned).map(n => <NoteCard note={n} />)}

<!-- Angular way: derive in the class, loop in the template -->
@for (note of pinnedNotes(); track note.id) {
  <app-note-card [note]="note" />
}
```

```typescript
pinnedNotes = computed(() => this.notes().filter(n => n.pinned));
```

Benefits: type-checked templates, no re-creating arrays each render, logic stays testable in the class.

### 3.2 Components don't re-run; signals update the DOM directly

A React component **function re-runs** on every state change, and React diffs the result. An Angular component **class is created once**. When a signal changes, Angular refreshes only the template bindings that read it.

So in Angular:

- No stale-closure bugs, no dependency arrays.
- A class field computed once in the constructor **won't update**; make it `computed()`.
- `effect()` is **not** `useEffect`. Don't use it to fetch data or to derive state. Use `computed()` for derived values and resources for async data. `effect()` is for syncing to the outside world (localStorage, logging, a non-Angular library).

### 3.3 Dependency injection instead of props drilling and context

React shares things with Context providers wrapped around trees. Angular has a **typed injector hierarchy**:

```typescript
@Service()
export class NotesStore { /* state + methods */ }

export class NotesPage   { store = inject(NotesStore); }
export class NotesToolbar { store = inject(NotesStore); }   // same instance, no providers to wrap
```

You can still scope a service to a subtree with `providers: [NotesStore]` on a component or route — the closest thing to a Context provider.

## 4. Side-by-side: the Notes page

**React**

```tsx
function NotesPage() {
  const [category, setCategory] = useState('all');
  const { data, isLoading } = useQuery({ queryKey: ['notes', category], queryFn: () => api.notes(category) });
  const pinned = useMemo(() => data?.filter(n => n.pinned) ?? [], [data]);

  if (isLoading) return <Spinner />;
  return (
    <>
      <ChipFilter value={category} onChange={setCategory} />
      {pinned.map(n => <NoteCard key={n.id} note={n} />)}
    </>
  );
}
```

**Angular 22**

```typescript
@Component({
  selector: 'app-notes-page',
  imports: [ChipFilter, NoteCard, Spinner],
  template: `
    <app-chip-filter [(value)]="category" />
    @if (notes.isLoading()) {
      <app-spinner />
    } @else {
      @for (note of pinned(); track note.id) {
        <app-note-card [note]="note" />
      }
    }
  `,
})
export class NotesPage {
  category = signal('all');
  notes = httpResource<Note[]>(() => `/api/v1/notes?category=${this.category()}`);
  pinned = computed(() => (this.notes.value() ?? []).filter(n => n.pinned));
}
```

Same behaviour; no dependency arrays, no query keys (the URL function's signal reads are the "keys").

## 5. Habits to drop

| React habit | Problem in Angular | Do instead |
|---|---|---|
| Logic inside templates | Not allowed / hard to test | `computed()` in the class |
| `effect()` for fetching | Races, loops, no loading state | `httpResource()` / `resource()` |
| `effect()` to sync two signals | Glitches, extra renders | `computed()` or `linkedSignal()` |
| Prop-drilling a store | Verbose | `inject(Store)` |
| One giant component | Hard to maintain | Smart page + presentational children |
| Forgetting `key` | n/a in React (warning) | `track` is **required** in `@for` |

## 6. Where Angular is stricter (and why that's good for Myspce)

- **Typed templates:** typos fail the build.
- **One way to do routing, forms, HTTP:** less decision fatigue.
- **DI:** swapping a real `NotesApi` for a mock in tests is one line.
- **Opinionated structure:** you'll find your own code faster six months later.

## 7. Practice questions

1. Convert a React component you wrote recently to Angular using `input()`, `output()`, `signal()` and `computed()`.
2. List three things you'd do with `useEffect` in React and say which Angular API replaces each.
3. Why doesn't Angular need dependency arrays?

## 8. Interview questions

- **Q: What's the Angular equivalent of `useMemo`?** `computed()`, which tracks its dependencies automatically and caches until they change.
- **Q: Is `effect()` the same as `useEffect`?** No. It runs when read signals change and is meant only for side effects; derived state uses `computed()`, async data uses resources.
- **Q: How does Angular share state without Context?** Through dependency injection: services are created by injectors and injected where needed, optionally scoped to routes or components.

## 9. Completion checklist

- [ ] I can explain the three mindset shifts.
- [ ] I rewrote one React component in Angular.
- [ ] I know which React habits to drop.
