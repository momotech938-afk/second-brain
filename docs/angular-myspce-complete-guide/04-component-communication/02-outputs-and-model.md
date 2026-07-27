# Outputs and `model()`

Back to [Module 04](README.md) · Previous: [Inputs](01-inputs.md) · Next: [Communication patterns](03-communication-patterns.md)

## 1. Learning objectives

- Declare outputs with `output()` and emit typed payloads.
- Design event payloads and names that read well in parent templates.
- Bridge Observables and outputs (`outputFromObservable`, `outputToObservable`).
- Build two-way bindable components with `model()`.

## 2. Prerequisites

[Inputs](01-inputs.md), [Event binding](../02-components-and-templates/02-data-binding.md).

## 3. What is it?

- An **output** is a custom event the child emits and the parent listens to: `(favoriteToggled)="…"`.
- A **model** is an input the child is also allowed to write, which notifies the parent: `[(value)]="query"`.

## 4. Why does Angular need it?

Children must report actions (click, toggle, delete) without knowing who handles them. `model()` covers the very common "controlled input" case (search boxes, toggles, chip selectors) without boilerplate.

## 5. How it works internally

`output()` creates an `OutputEmitterRef`. When the parent binds `(open)="handle($event)"`, Angular subscribes the handler; `emit(x)` calls it synchronously with `x` as `$event`. Outputs don't bubble through the DOM; only the direct parent's binding hears them.

`model()` creates a `ModelSignal`: readable like an input, writable with `set`/`update`. Writing it emits an implicit `<name>Change` output. With `[(value)]="query"`, Angular writes the new value into the parent's `query` signal (or field).

## 6. Syntax and API reference

```typescript
import { output, model } from '@angular/core';
import { outputFromObservable, outputToObservable } from '@angular/core/rxjs-interop';

export class NoteCard {
  open = output<string>();                          // emits a note id
  favoriteToggled = output<{ id: string; value: boolean }>();
  deleted = output<void>();                         // emit() with no argument
  closed = output({ alias: 'dismiss' });            // parent binds (dismiss)
}

export class SearchBox {
  value = model('');                                // [(value)]="query"
  required = model.required<string>();              // must be bound
}

export class LiveClock {
  tick = outputFromObservable(interval(1000));      // Observable → output
}
// parent side, programmatically: outputToObservable(childRef.instance.open).subscribe(...)
```

Emitting: `this.open.emit(this.note().id)`. Writing a model: `this.value.set('abc')` or `this.value.update(v => v.trim())`.

### Naming

- Name outputs after **what happened**, past tense or noun: `toggled`, `deleted`, `selectionChange`, `open` (for "request to open").
- Don't prefix with `on` (`onSave`) — the parent's template reads `(onSave)` which is redundant.
- Don't reuse native event names (`click`, `change`) for custom outputs — it confuses readers and can collide with DOM events on the host.

### Payload design

| Payload | When |
|---|---|
| `void` | the event itself is the information (`closed`) |
| an id | the parent owns the entity list (`open: output<string>()`) |
| `{ id, value }` | the parent needs the new state |
| the whole entity | the parent doesn't have it (rare for lists) |

### v22 rule

A component can't have both `model('user')` and an explicit `output('userChange')` — it's a compile error (`NG1054`) because `model` already creates `userChange`. The v22 migration rewrites such code to `input` + `linkedSignal` + `output`.

## 7. Smallest working example

```typescript
// shared/ui/toggle.ts
import { Component, input, model } from '@angular/core';

@Component({
  selector: 'app-toggle',
  template: `
    <button type="button" role="switch" [attr.aria-checked]="checked()" [attr.aria-label]="label()"
            class="relative h-6 w-11 rounded-full transition"
            [class]="checked() ? 'bg-blue-600' : 'bg-slate-300 dark:bg-slate-700'"
            (click)="checked.update(v => !v)">
      <span class="absolute top-0.5 size-5 rounded-full bg-white transition-all"
            [class]="checked() ? 'left-5.5' : 'left-0.5'"></span>
    </button>
  `,
})
export class Toggle {
  checked = model(false);
  label = input.required<string>();
}
```

```html
<!-- features/settings/pages/settings-page.html -->
<app-toggle label="Lock app on inactivity" [(checked)]="settings.lockOnIdle" />
```

## 8. Step-by-step

1. The parent passes its signal `settings.lockOnIdle` with `[(checked)]`.
2. Clicking calls `checked.update(v => !v)` inside the child.
3. Angular emits `checkedChange` and writes the new value into `settings.lockOnIdle`.
4. Both sides now agree; there's one source of truth (the parent's signal).

## 9. Myspce real-world example — `ChipFilter`

The category chips on Notes and Credentials (All, Work, Personal, …):

```typescript
// shared/ui/chip-filter.ts
import { Component, input, model } from '@angular/core';

export interface ChipOption {
  value: string;
  label: string;
  count?: number;
}

@Component({
  selector: 'app-chip-filter',
  template: `
    <div role="radiogroup" [attr.aria-label]="label()" class="flex gap-2 overflow-x-auto pb-1">
      @for (opt of options(); track opt.value) {
        <button type="button" role="radio" [attr.aria-checked]="value() === opt.value"
                class="shrink-0 rounded-lg px-3 py-1.5 text-sm transition"
                [class]="value() === opt.value
                  ? 'bg-blue-600 text-white'
                  : 'bg-slate-100 text-slate-600 hover:bg-slate-200 dark:bg-slate-800 dark:text-slate-300'"
                (click)="value.set(opt.value)">
          {{ opt.label }}
          @if (opt.count !== undefined) { <span class="ml-1 opacity-70">{{ opt.count }}</span> }
        </button>
      }
    </div>
  `,
})
export class ChipFilter {
  options = input.required<ChipOption[]>();
  value = model('all');
  label = input('Filter');
}
```

```html
<app-chip-filter label="Note category" [options]="store.categoryOptions()" [(value)]="store.category" />
```

## 10. Advanced example — outputs that carry intent (Credentials)

Revealing a password must go through the API (re-auth, audit log). The row only **asks**:

```typescript
// features/credentials/components/credential-row.ts (additions)
copyRequested = output<string>();      // credential id
revealRequested = output<string>();
editRequested = output<string>();
protected menuOpen = signal(false);    // local UI state for the "⋯" menu (import signal)
```

```html
<button type="button" class="rounded p-2 hover:bg-slate-800" [attr.aria-label]="'Copy password for ' + credential().name"
        (click)="copyRequested.emit(credential().id)">⧉</button>
<button type="button" class="rounded p-2 hover:bg-slate-800" [attr.aria-label]="'More actions for ' + credential().name"
        (click)="menuOpen.set(true)">⋯</button>
```

The page decides what "copy" means (call `POST /credentials/:id/reveal`, copy, clear clipboard after 30 s, show a toast). The row stays dumb and testable.

## 11. Visual explanation

```text
Parent                                 Child
query = signal('')                     value = model('')
<app-search-box [(value)]="query">
     │  [value]="query()"  ─────────►  value()  (reads)
     │                                 value.set('ang')  (user types)
     └─ query.set($event) ◄─────────── (valueChange) emits 'ang'
```

## 12. Common mistakes

| Wrong | Why | Right |
|---|---|---|
| `@Output() save = new EventEmitter()` in new code | legacy | `save = output<Note>()` |
| Emitting inside a `computed()` or the template | side effects during render | emit in event handlers |
| `[(value)]="store.query()"` | you passed the **value**, not the signal | `[(value)]="store.query"` |
| Output named `change` | collides with native `change` on host | `selectionChange` |
| Child holding its own copy of the selected value *and* emitting | two sources of truth | use `model()` |

## 13. Debugging guide

- Handler never runs: check the event name matches the output (or its alias) and that you bound it on the right element.
- `$event` has the wrong type: the output's generic determines it; fix `output<T>()`.
- Two-way binding doesn't update the parent: the parent expression must be **writable** (a signal or a field), not a function call.

## 14. Best practices

- `model()` for "value + change" pairs; `output()` for actions.
- Small, purposeful payloads (ids).
- Children request; parents decide.

## 15. Angular vs React

`output()` ≈ an `onX` callback prop. `model()` ≈ a controlled component's `value` + `onChange` pair in one declaration. Angular's `[(value)]` has no React equivalent; React always wires both halves manually.

## 16. Testing

```typescript
it('emits a new value when a chip is clicked', async () => {
  const fixture = TestBed.createComponent(ChipFilter);
  fixture.componentRef.setInput('options', [{ value: 'all', label: 'All' }, { value: 'work', label: 'Work' }]);
  const emitted: string[] = [];
  fixture.componentInstance.value.subscribe(v => emitted.push(v));   // ModelSignal is subscribable like an output
  await fixture.whenStable();
  (fixture.nativeElement.querySelectorAll('button')[1] as HTMLButtonElement).click();
  expect(emitted).toEqual(['work']);
});
```

## 17. Practice questions

1. Add `pinToggled` and `deleted` outputs to `NoteCard` with good payloads.
2. Build `SearchBox` with `model('')`, a clear button, and Escape to clear.
3. Why shouldn't `CredentialRow` call the reveal API itself?

## 18. Interview questions

- **Beginner — How does a child send data to a parent?** Through an output: `x = output<T>()`, `this.x.emit(value)`, parent binds `(x)="…"`.
- **Intermediate — What does `model()` generate?** A writable signal input plus an implicit `<name>Change` output, enabling `[(name)]`.
- **Advanced — Do outputs bubble?** No. They're delivered only to bindings on that component's host element in the parent template; use a service for wider communication.

## 19. Revision notes

`output<T>()` + `emit()` · past-tense names, no `on` prefix · `model()` = writable input + `xChange` · `[(x)]` needs a writable target · `outputFromObservable` for streams.

## 20. Completion checklist

- [ ] I built `Toggle`, `ChipFilter` and `SearchBox` with `model()`.
- [ ] `CredentialRow` emits intent outputs only.
- [ ] I tested an output.
