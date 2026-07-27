# Change Detection: OnPush, Eager, Signals and Zoneless

Back to [Module 09](README.md) · Previous: [Lifecycle hooks](01-lifecycle-hooks.md) · Next: [Cleanup and memory leaks](03-cleanup-and-memory-leaks.md)

> **Version note:** Since **v21**, new apps are **zoneless** by default. Since **v22**, components without a `changeDetection` setting use **`OnPush`** by default; the old default is now called **`ChangeDetectionStrategy.Eager`** (`Default` is a deprecated alias). `ng update` to v22 adds `Eager` to existing components that relied on the old default.

## 1. Learning objectives

- Explain what change detection (CD) does and when it runs.
- State precisely what makes an OnPush component refresh.
- Explain how zoneless Angular schedules rendering.
- Use `ChangeDetectorRef` correctly (and rarely).
- Fix "my UI doesn't update" bugs.

## 2. Prerequisites

[Signals](../08-signals-and-reactivity/01-signal-basics.md), [Lifecycle hooks](01-lifecycle-hooks.md).

## 3. What is it?

**Change detection** is the process where Angular walks component views, re-evaluates template bindings, and updates the DOM where values changed.

## 4. Why does Angular need it?

Templates are declarative. When state changes, something must re-run the bindings. The question is **when** and **which views** — that's what strategies and zone/zoneless control.

## 5. How it works internally

### The tree

Each component has a **view**. CD starts at the root and visits views top-down. For each visited view it runs the update instructions (compare old/new binding values, patch DOM), then visits children.

### Strategies

| Strategy | A view is refreshed when… |
|---|---|
| **Eager** (old "Default") | every time CD runs and reaches it |
| **OnPush** (v22 default) | it's **marked dirty**, which happens when: (1) an **input** changes (new reference/value), (2) a **template event** fires inside it or its children, (3) a **signal** read in its template changes, (4) an `async` pipe in it emits, (5) you call `markForCheck()` |

When an OnPush view isn't dirty, Angular skips it **and its subtree** — unless a descendant is dirty because of a signal, in which case Angular refreshes just that descendant (targeted refresh, v17+).

### Zone.js vs zoneless

| | Zone.js apps (pre-v21 default) | Zoneless (v21+ default) |
|---|---|---|
| How Angular knows to run CD | Zone.js patches `setTimeout`, Promises, events, XHR… and runs CD after **any** async task | Angular schedules CD only on **notifications**: signal set (read by a template), template/host event, input change, `markForCheck()`, `ComponentRef.setInput`, attaching views, async pipe |
| Bundle | + ~30 kB zone.js | none |
| Surprises | CD runs a lot, "magic" updates | a plain field changed in `setTimeout` won't render |
| Provider | `provideZoneChangeDetection()` | nothing (default) — older apps opt in with `provideZonelessChangeDetection()` |

### Why OnPush + signals + zoneless fit together

Signals tell Angular *exactly* which views read changed state. OnPush skips everything else. Zoneless removes the guessing layer. The result: fewer checks, smaller bundle, predictable updates.

## 6. Syntax and API reference

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,   // explicit (redundant in v22, harmless)
  // changeDetection: ChangeDetectionStrategy.Eager, // opt back into "check always"
})
```

```typescript
const cdr = inject(ChangeDetectorRef);
cdr.markForCheck();   // mark this view and ancestors dirty → refreshed on next CD (the right tool when state isn't a signal)
cdr.detectChanges();  // run CD synchronously for this view and children — rarely needed
cdr.detach();         // stop checking this view; reattach() to resume — for very special cases
```

## 7. Smallest working example — the classic bug

```typescript
@Component({ selector: 'app-clock', template: `{{ time }}` })
export class ClockBroken {
  time = new Date().toLocaleTimeString();
  constructor() {
    setInterval(() => (this.time = new Date().toLocaleTimeString()), 1000);
    // ❌ zoneless + OnPush: nothing tells Angular to re-render; the clock freezes
  }
}

@Component({ selector: 'app-clock', template: `{{ time() }}` })
export class Clock {
  time = signal(new Date().toLocaleTimeString());
  constructor() {
    const id = setInterval(() => this.time.set(new Date().toLocaleTimeString()), 1000);
    inject(DestroyRef).onDestroy(() => clearInterval(id));
    // ✅ signal set → template is a consumer → Angular schedules a refresh
  }
}
```

## 8. Step-by-step

1. In `ClockBroken`, the field changes but no notification reaches Angular (no zone, not a signal, not an event).
2. In `Clock`, `time.set()` notifies the template consumer, which marks the view dirty and schedules CD.

## 9. Myspce real-world example — what triggers updates where

| Myspce interaction | Trigger | Works with OnPush/zoneless? |
|---|---|---|
| Click a category chip | template event + signal set | ✅ |
| `httpResource` finishes loading | resource signals change | ✅ |
| `TasksStore.toggle()` updates a signal | signal read by `TasksPage` | ✅ |
| Parent passes a new `note` object to `NoteCard` | input change | ✅ |
| Parent **mutates** `note.title` | nothing (same reference) | ❌ → update immutably |
| Third-party callback (Leaflet marker click) sets a plain field | nothing | ❌ → set a signal |
| WebSocket message pushed into an array field | nothing | ❌ → signal or `toSignal` |
| `subscribe(v => this.x = v)` | nothing | ❌ → `toSignal`, or set a signal in `subscribe` |

## 10. Advanced example — migrating an Eager component

After `ng update` to v22 you'll see components like:

```typescript
@Component({
  selector: 'app-legacy-list',
  changeDetection: ChangeDetectionStrategy.Eager,   // added by the migration
  template: `@for (n of notes; track n.id) { <li>{{ n.title }}</li> }`,
})
export class LegacyList {
  notes: Note[] = [];
  constructor() {
    inject(NotesApi).list().subscribe(r => (this.notes = r.data));
  }
}
```

Steps to make it OnPush-ready:

1. Turn state into signals: `notes = toSignal(inject(NotesApi).list().pipe(map(r => r.data)), { initialValue: [] })` — or better, an `httpResource`.
2. Replace mutations with immutable updates.
3. Inputs: make sure parents pass new references.
4. Remove `changeDetection: ChangeDetectionStrategy.Eager`.
5. Click through the screen; anything that stops updating is a hidden mutation or non-signal field.

Searching the codebase for `Eager` gives you a migration checklist.

## 11. Visual explanation

```text
                App (OnPush)
               /            \
     Shell (OnPush)        Toasts (OnPush)
       |
  NotesPage (OnPush)  ← reads store.visibleNotes()  [dirty: signal changed]
     /        \
 ChipFilter   NotesGrid (OnPush) ← input `notes` new array [dirty]
               |
            NoteCard ×12 (OnPush) ← inputs: only cards whose `note` reference changed are refreshed
Toasts, Sidebar, Topbar: skipped (not dirty)
```

## 12. Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Mutating arrays/objects | child doesn't update | new references |
| Plain fields updated in callbacks | stale UI in zoneless | signals |
| Calling `detectChanges()` everywhere "to fix it" | hides design problems, can throw if view destroyed | fix the data flow; `markForCheck()` if truly needed |
| `ChangeDetectionStrategy.Default` in new code | deprecated alias | `Eager` only if you must; prefer OnPush |
| Expecting `setTimeout` to trigger rendering (zoneless) | nothing happens | signals |

## 13. Debugging guide

- **Angular DevTools → Profiler:** record, interact, see which components were checked and how long CD took.
- **`NG0100 ExpressionChangedAfterItHasBeenChecked`** (dev mode): a binding's value changed during the same CD pass — usually state written in a hook or getter.
- **Zoneless debugging:** `provideCheckNoChangesConfig({ exhaustive: true, interval: 1000 })` (dev only) periodically re-checks all views and reports bindings that changed without notifying Angular — perfect for finding non-signal state.
- Stale UI checklist: Is it a signal? Is it read with `()`? Was it mutated? Is the parent passing a new reference?

## 14. Best practices

- Leave new components at the v22 default (OnPush).
- All template state in signals, inputs, or resources.
- Immutable updates everywhere.
- `markForCheck()` only at boundaries with non-signal code you can't change.
- Never call `detectChanges()` in normal app code.

## 15. Angular vs React

React re-renders a component when its state/props change and then diffs the virtual DOM. Angular OnPush + signals is closer to "fine-grained" frameworks (Solid, Vue): no component re-execution, only bindings that read changed signals update.

## 16. Testing

Zoneless tests: use `await fixture.whenStable()` (it waits for Angular's scheduler) instead of relying on `fixture.detectChanges()` everywhere. TestBed is zoneless by default in zoneless apps. To catch non-signal state in tests, enable auto-detect checks or call `fixture.detectChanges()` which in dev mode also runs `checkNoChanges`.

```typescript
it('updates the clock', async () => {
  vi.useFakeTimers();
  const fixture = TestBed.createComponent(Clock);
  await fixture.whenStable();
  const first = fixture.nativeElement.textContent;
  vi.advanceTimersByTime(1000);
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).not.toBe(first);
  vi.useRealTimers();
});
```

## 17. Practice questions

1. List the five things that mark an OnPush view dirty.
2. Why does the broken clock work in an old Zone.js + Eager app?
3. Convert a component that uses `subscribe(x => this.field = x)` to signals.
4. What would you search for in a codebase after upgrading to v22, and why?

## 18. Interview questions

- **Beginner — What's OnPush?** A strategy where a component is only refreshed when its inputs change, an event fires inside it, a signal it reads changes, or it's explicitly marked.
- **Intermediate — How does a zoneless app know when to update?** Angular schedules CD on notifications: signal changes read by templates, template events, input changes, `markForCheck`, async pipe emissions.
- **Advanced — What changed in v22?** OnPush became the default; the previous default was renamed `Eager`; migrations add `Eager` to components that implicitly relied on it.

## 19. Revision notes

| Concept | Key point |
|---|---|
| OnPush (v22 default) | inputs, events, signals, async pipe, `markForCheck` |
| Eager | old default, check always |
| Zoneless (v21+ default) | no zone.js; notifications schedule CD |
| `markForCheck` | rare escape hatch |
| `detectChanges` | almost never |
| Debug | Profiler, `NG0100`, `provideCheckNoChangesConfig` |

## 20. Completion checklist

- [ ] I can explain why the broken clock freezes.
- [ ] I migrated an Eager component to OnPush.
- [ ] I used the Profiler to see which components were checked.
