# Signal Stores (Service + Signals)

Back to [Module 15](README.md) · Previous: [State fundamentals](01-state-fundamentals.md) · Next: [NgRx and SignalStore](03-ngrx-and-signal-store.md)

## 1. Learning objectives

- Write a complete feature store with signals, resources and actions.
- Implement optimistic updates with rollback.
- Handle mutations and invalidation.
- Use `linkedSignal` for dependent writable state.
- Test stores with fake APIs.

## 2. Prerequisites

[Signals and resources](../08-signals-and-reactivity/README.md), [API layer](../07-services-and-dependency-injection/04-myspce-api-layer.md).

## 3. What is it?

A plain service (`@Service()`) that holds a feature's state in signals, exposes read-only views, and offers methods (actions) that change state and call APIs. No library required.

## 4. Why does Angular need it?

Pages stay thin, logic is in one testable place, and every component reading the store sees the same data.

## 5. The store template

```typescript
@Service()
export class XStore {
  // 1. dependencies
  private api = inject(XApi);

  // 2. private writable state
  private readonly _state = signal<…>(…);

  // 3. server data (resource) driven by state
  readonly items = rxResource({ params: () => …, stream: ({ params }) => this.api.list(params) });

  // 4. public read-only selectors
  readonly visible = computed(() => …);

  // 5. actions
  async create(dto: …) { … }
}
```

## 6. Myspce real-world example — `TasksStore`

```typescript
// features/tasks/data-access/tasks-store.ts
import { Service, computed, inject, signal } from '@angular/core';
import { rxResource } from '@angular/core/rxjs-interop';
import { firstValueFrom } from 'rxjs';
import { TasksApi, CreateTaskDto } from './tasks-api';
import { Task } from '../../../core/models';
import { ToastService } from '../../../shared/ui/toast/toast-service';
import { toMessage } from '../../../core/http/error-messages';

const DAY = 86_400_000;
const startOfDay = (d: Date) => new Date(d.getFullYear(), d.getMonth(), d.getDate()).getTime();

@Service()
export class TasksStore {
  private api = inject(TasksApi);
  private toast = inject(ToastService);

  /** Reload trigger: bump to refetch. */
  private readonly version = signal(0);

  readonly tasks = rxResource({
    params: () => ({ v: this.version() }),
    stream: () => this.api.list({ limit: 200 }),
    defaultValue: { data: [], meta: { total: 0, page: 1, limit: 200 } },
  });

  private readonly list = computed(() => (this.tasks.hasValue() ? this.tasks.value().data : []));

  readonly grouped = computed(() => {
    const today = startOfDay(new Date());
    const g = { today: [] as Task[], tomorrow: [] as Task[], week: [] as Task[], later: [] as Task[], completed: [] as Task[] };
    for (const t of this.list()) {
      if (t.status === 'done') { g.completed.push(t); continue; }
      const due = t.dueAt ? startOfDay(new Date(t.dueAt)) : null;
      if (due === null) g.later.push(t);
      else if (due <= today) g.today.push(t);            // overdue counts as today
      else if (due === today + DAY) g.tomorrow.push(t);
      else if (due < today + 7 * DAY) g.week.push(t);
      else g.later.push(t);
    }
    return g;
  });

  /** "This Week Progress" on the dashboard: done / total for tasks due this week. */
  readonly weekProgress = computed(() => {
    const today = startOfDay(new Date());
    const week = this.list().filter(t => t.dueAt && startOfDay(new Date(t.dueAt)) < today + 7 * DAY && startOfDay(new Date(t.dueAt)) >= today);
    const done = week.filter(t => t.status === 'done').length;
    return { done, total: week.length, percent: week.length ? Math.round((done / week.length) * 100) : 0 };
  });

  // ---- actions ----

  /** Optimistic toggle with rollback. */
  async toggle(id: string): Promise<void> {
    const before = this.tasks.value();
    this.tasks.update(p => ({ ...p, data: p.data.map(t => (t.id === id ? { ...t, status: t.status === 'done' ? 'todo' : 'done' } : t)) }));
    try {
      const updated = await firstValueFrom(this.api.toggle(id));
      this.tasks.update(p => ({ ...p, data: p.data.map(t => (t.id === id ? updated : t)) }));   // trust the server's version
    } catch (err) {
      this.tasks.set(before);
      this.toast.error(toMessage(err));
    }
  }

  async create(dto: CreateTaskDto): Promise<Task | null> {
    try {
      const task = await firstValueFrom(this.api.create(dto));
      this.tasks.update(p => ({ ...p, data: [task, ...p.data], meta: { ...p.meta, total: p.meta.total + 1 } }));
      return task;
    } catch (err) {
      this.toast.error(toMessage(err));
      return null;
    }
  }

  async remove(id: string): Promise<void> {
    const before = this.tasks.value();
    this.tasks.update(p => ({ ...p, data: p.data.filter(t => t.id !== id) }));
    try {
      await firstValueFrom(this.api.remove(id));
      this.toast.success('Moved to Trash', { action: { label: 'Undo', run: () => this.restore(id) } });
    } catch (err) {
      this.tasks.set(before);
      this.toast.error(toMessage(err));
    }
  }

  async restore(id: string) {
    await firstValueFrom(this.api.restore(id));
    this.refresh();
  }

  refresh() { this.version.update(v => v + 1); }
}
```

Notes on the design:

- **One resource, many views.** `TasksPage`, the dashboard's "Today's Tasks" and "This Week Progress" all read the same store — tick a task anywhere and every view updates.
- **Optimistic updates** make toggles instant; rollback keeps the UI honest on failure.
- **Server wins**: after success, replace with the server's entity.
- **Invalidation**: create inserts locally; restore refetches (simpler, since the restored item's position is server-defined).
- **Client-side grouping** is fine for a single user's ~hundreds of tasks; the API also supports `?view=today` if lists grow.

Using it:

```typescript
// features/dashboard/components/today-tasks-widget.ts
export class TodayTasksWidget {
  protected store = inject(TasksStore);
}
```

```html
@for (t of store.grouped().today.slice(0, 5); track t.id) {
  <app-task-item [task]="t" (toggled)="store.toggle($event)" />
} @empty {
  <p class="text-sm text-slate-500">Nothing due today 🎉</p>
}
<app-progress-bar [value]="store.weekProgress().percent" label="This week progress" />
<p class="text-xs text-slate-500">{{ store.weekProgress().done }}/{{ store.weekProgress().total }} tasks completed</p>
```

## 7. Dependent writable state with `linkedSignal`

```typescript
// selected task in a master/detail layout; keep the selection if it still exists after a refresh
readonly selectedId = linkedSignal<Task[], string | null>({
  source: () => this.list(),
  computation: (tasks, prev) => (prev?.value && tasks.some(t => t.id === prev.value) ? prev.value : (tasks[0]?.id ?? null)),
});
readonly selected = computed(() => this.list().find(t => t.id === this.selectedId()) ?? null);
```

## 8. Common mistakes

| Mistake | Fix |
|---|---|
| Public writable signals for server data | private + read-only views |
| Components calling APIs directly while a store exists | go through the store |
| Forgetting rollback in optimistic updates | snapshot `before`, restore on error |
| Computing "today" once at store creation | compute inside `computed` (or inject a `NOW` clock signal for midnight rollover) |
| Calling `refresh()` after every action | local updates where you have the data; refetch only when needed |

## 9. Debugging guide

- DevTools signal graph: confirm `grouped` depends on `tasks`.
- Add `debugName` to store signals.
- If a view doesn't update after `toggle`, check you used `update` with new objects, not mutation.

## 10. Testing

```typescript
describe('TasksStore', () => {
  const task = (over: Partial<Task>): Task => ({ id: 't1', title: 'Read Angular docs', notes: null, dueAt: new Date().toISOString(), priority: 'medium', status: 'todo', category: null, goalId: null, ...over });

  function setup(api: Partial<TasksApi>) {
    TestBed.configureTestingModule({ providers: [{ provide: TasksApi, useValue: api }, { provide: ToastService, useValue: { error: vi.fn(), success: vi.fn() } }] });
    const store = TestBed.inject(TasksStore);
    TestBed.tick();                // start the resource
    return store;
  }

  it('groups today and computes week progress', async () => {
    const store = setup({ list: () => of({ data: [task({}), task({ id: 't2', status: 'done' })], meta: { total: 2, page: 1, limit: 200 } }) });
    await TestBed.inject(ApplicationRef).whenStable();
    expect(store.grouped().today.map(t => t.id)).toEqual(['t1']);
    expect(store.weekProgress()).toEqual({ done: 1, total: 2, percent: 50 });
  });

  it('rolls back an optimistic toggle on error', async () => {
    const store = setup({ list: () => of({ data: [task({})], meta: { total: 1, page: 1, limit: 200 } }), toggle: () => throwError(() => new Error('boom')) });
    await TestBed.inject(ApplicationRef).whenStable();
    await store.toggle('t1');
    expect(store.tasks.value().data[0]!.status).toBe('todo');
  });
});
```

## 11. Practice questions

1. Build `NotesStore` with the same template: list resource, `visibleNotes`, optimistic `toggleFavorite`, `pin`, `remove` with Undo.
2. Add a `NOW` signal that ticks at midnight so "Today" groups roll over without a refresh.
3. When would you move tasks from a root store to a route-scoped store?

## 12. Interview questions

- **Q: What's a signal store?** A service with private writable signals, public read-only computed views and action methods that call APIs.
- **Q: How do you do optimistic updates safely?** Snapshot state, apply the change locally, call the API, replace with the server response on success, restore the snapshot on failure.

## 13. Completion checklist

- [ ] `TasksStore` drives the Tasks page and two dashboard widgets.
- [ ] Optimistic toggle and Undo-delete work.
- [ ] Store tests cover grouping and rollback.
