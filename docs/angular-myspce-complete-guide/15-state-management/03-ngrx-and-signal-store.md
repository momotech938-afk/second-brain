# NgRx Store and NgRx SignalStore

Back to [Module 15](README.md) · Previous: [Signal stores](02-signal-stores.md) · Next module: [16 Advanced components](../16-advanced-components/README.md)

> NgRx is a third-party library (not part of Angular). Its major versions track Angular's; check https://ngrx.io for the version that supports Angular 22 before installing. This note teaches the concepts and shapes so you can read and write NgRx code at work — **Myspce v1 doesn't need it**.

## 1. Learning objectives

- Explain the Redux pattern: actions, reducers, selectors, effects.
- Read and write classic NgRx Store code.
- Use NgRx Entity for collections.
- Build a store with `@ngrx/signals` (`signalStore`).
- Decide objectively whether a project needs NgRx.

## 2. Prerequisites

[Signal stores](02-signal-stores.md), [RxJS operators](../14-rxjs/02-operators.md).

## 3. What is it?

- **NgRx Store:** a global, immutable state tree changed only by dispatching **actions** handled by pure **reducers**; read through memoised **selectors**; side effects (HTTP) in **effects**.
- **NgRx SignalStore (`@ngrx/signals`):** a lighter, signal-based store builder with composable features (`withState`, `withComputed`, `withMethods`, `withHooks`, `withEntities`).

## 4. Why might a project need it?

| Signal services are enough when… | NgRx helps when… |
|---|---|
| one or a few developers | many teams touching shared state |
| feature-local state | lots of cross-feature coordination |
| simple async (load/save) | complex event flows (undo/redo, offline queues, real-time sync) |
| debugging by reading code | you want Redux DevTools, time-travel, action logs |

Myspce (single user, one developer, CRUD features) is firmly in the left column.

## 5. How classic NgRx works

```mermaid
flowchart LR
  C[Component] -- dispatch(action) --> S[(Store)]
  S --> R[Reducer: (state, action) => newState]
  R --> S
  S -- select(selector) --> C
  S -- actions stream --> E[Effects]
  E -- HTTP --> API[(API)]
  E -- dispatch(success/failure) --> S
```

## 6. Syntax — a Tasks feature in classic NgRx

```typescript
// tasks.actions.ts
import { createActionGroup, emptyProps, props } from '@ngrx/store';

export const TasksActions = createActionGroup({
  source: 'Tasks',
  events: {
    'Load': emptyProps(),
    'Load Success': props<{ tasks: Task[] }>(),
    'Load Failure': props<{ error: string }>(),
    'Toggle': props<{ id: string }>(),
    'Toggle Failure': props<{ id: string; error: string }>(),
  },
});
```

```typescript
// tasks.reducer.ts
import { createFeature, createReducer, on } from '@ngrx/store';
import { createEntityAdapter, EntityState } from '@ngrx/entity';

interface TasksState extends EntityState<Task> { status: 'idle' | 'loading' | 'loaded' | 'error'; error: string | null; }
const adapter = createEntityAdapter<Task>();
const initialState: TasksState = adapter.getInitialState({ status: 'idle', error: null });

const flip = (t: Task): Task['status'] => (t.status === 'done' ? 'todo' : 'done');

export const tasksFeature = createFeature({
  name: 'tasks',
  reducer: createReducer(
    initialState,
    on(TasksActions.load, s => ({ ...s, status: 'loading' as const })),
    on(TasksActions.loadSuccess, (s, { tasks }) => adapter.setAll(tasks, { ...s, status: 'loaded' as const })),
    on(TasksActions.loadFailure, (s, { error }) => ({ ...s, status: 'error' as const, error })),
    on(TasksActions.toggle, (s, { id }) => adapter.updateOne({ id, changes: { status: flip(s.entities[id]!) } }, s)),   // optimistic
    on(TasksActions.toggleFailure, (s, { id }) => adapter.updateOne({ id, changes: { status: flip(s.entities[id]!) } }, s)),  // revert
  ),
  extraSelectors: ({ selectTasksState }) => ({ ...adapter.getSelectors(selectTasksState) }),
});
```

```typescript
// tasks.effects.ts
import { inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, exhaustMap, map, mergeMap, of } from 'rxjs';

export const loadTasks = createEffect(
  (actions$ = inject(Actions), api = inject(TasksApi)) =>
    actions$.pipe(
      ofType(TasksActions.load),
      exhaustMap(() => api.list().pipe(
        map(r => TasksActions.loadSuccess({ tasks: r.data })),
        catchError(e => of(TasksActions.loadFailure({ error: toMessage(e) }))),
      )),
    ),
  { functional: true },
);

export const toggleTask = createEffect(
  (actions$ = inject(Actions), api = inject(TasksApi)) =>
    actions$.pipe(
      ofType(TasksActions.toggle),
      mergeMap(({ id }) => api.toggle(id).pipe(
        map(() => ({ type: '[Tasks] Toggle Synced' })),
        catchError(e => of(TasksActions.toggleFailure({ id, error: toMessage(e) }))),
      )),
    ),
  { functional: true },
);
```

```typescript
// registration (app.config.ts or route providers)
provideStore(), provideState(tasksFeature), provideEffects({ loadTasks, toggleTask }), provideStoreDevtools({ maxAge: 25 })

// component
private store = inject(Store);
tasks = this.store.selectSignal(tasksFeature.selectAll);    // signal from a selector
status = this.store.selectSignal(tasksFeature.selectStatus);
ngOnInit() { this.store.dispatch(TasksActions.load()); }
toggle(id: string) { this.store.dispatch(TasksActions.toggle({ id })); }
```

Count the files and lines compared with `TasksStore` in [note 2](02-signal-stores.md). That ceremony buys you traceability (every change is a named action in DevTools) at the cost of boilerplate.

## 7. NgRx SignalStore — the middle ground

```typescript
// features/tasks/data-access/tasks.signal-store.ts
import { computed, inject } from '@angular/core';
import { patchState, signalStore, withComputed, withHooks, withMethods, withState } from '@ngrx/signals';
import { firstValueFrom } from 'rxjs';

type State = { tasks: Task[]; status: 'idle' | 'loading' | 'loaded' | 'error' };

export const TasksSignalStore = signalStore(
  { providedIn: 'root' },
  withState<State>({ tasks: [], status: 'idle' }),
  withComputed(({ tasks }) => ({
    open: computed(() => tasks().filter(t => t.status === 'todo')),
    doneCount: computed(() => tasks().filter(t => t.status === 'done').length),
  })),
  withMethods((store, api = inject(TasksApi)) => ({
    async load() {
      patchState(store, { status: 'loading' });
      try {
        const r = await firstValueFrom(api.list());
        patchState(store, { tasks: r.data, status: 'loaded' });
      } catch {
        patchState(store, { status: 'error' });
      }
    },
    async toggle(id: string) {
      const before = store.tasks();
      patchState(store, { tasks: before.map(t => (t.id === id ? { ...t, status: t.status === 'done' ? 'todo' : 'done' } : t)) });
      try { await firstValueFrom(api.toggle(id)); } catch { patchState(store, { tasks: before }); }
    },
  })),
  withHooks({ onInit: store => store.load() }),
);

// component: store = inject(TasksSignalStore); → store.open(), store.toggle(id)
```

SignalStore gives structure (state/computed/methods/hooks), reusable custom features (e.g. a `withRequestStatus()` feature shared by all stores), entity helpers (`withEntities`) and DevTools integration — with far less boilerplate than classic Store. If Myspce ever needs a library, this is the likely choice.

## 8. Decision checklist

Adopt NgRx (Store or SignalStore) only if you can tick **at least two**:

- [ ] Several features read and write the same complex state.
- [ ] You need an audit log of state changes or time-travel debugging.
- [ ] Complex async orchestration (offline queue, optimistic multi-step flows, websockets) is central.
- [ ] Multiple developers need enforced conventions.
- [ ] Your company codebase already uses it (consistency wins).

## 9. Common mistakes

| Mistake | Fix |
|---|---|
| Putting every piece of state in the global store (dialog open, hover) | local signals |
| Effects that dispatch the action they listen to | infinite loops |
| Non-pure reducers (dates, random, API calls) | reducers must be pure |
| `select()` + `async` pipe everywhere in new code | `selectSignal()` |
| Choosing NgRx for a small app "to be future-proof" | refactoring later is cheap with clean API/store layers |

## 10. Testing

Classic NgRx: test reducers as pure functions (`reducer(state, action)`), selectors with `projector`, effects with `provideMockActions`. SignalStore: inject the store with fake APIs, call methods, assert signals — the same as plain signal stores.

```typescript
it('reverts toggle on failure (reducer)', () => {
  const s1 = tasksFeature.reducer(undefined, TasksActions.loadSuccess({ tasks: [{ id: 't1', status: 'todo' } as Task] }));
  const s2 = tasksFeature.reducer(s1, TasksActions.toggle({ id: 't1' }));
  const s3 = tasksFeature.reducer(s2, TasksActions.toggleFailure({ id: 't1', error: 'x' }));
  expect(s3.entities['t1']!.status).toBe('todo');
});
```

## 11. Interview questions

- **Q: What are actions, reducers, selectors and effects?** Events describing what happened; pure functions producing new state; memoised queries over state; side-effect handlers that map actions to other actions.
- **Q: When is NgRx worth it?** Large teams, shared complex state, need for traceability/time-travel or complex async orchestration.
- **Q: What's NgRx SignalStore?** A signal-based store builder from NgRx with composable features and less boilerplate than the Redux-style Store.

## 12. Completion checklist

- [ ] I can read a classic NgRx feature and explain each file.
- [ ] I built `TasksSignalStore` and compared it with my plain `TasksStore`.
- [ ] I can defend "no NgRx in Myspce v1" with the checklist.
