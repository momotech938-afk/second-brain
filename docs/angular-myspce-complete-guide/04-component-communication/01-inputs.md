# Inputs

Back to [Module 04](README.md) · Next: [Outputs and `model()`](02-outputs-and-model.md)

## 1. Learning objectives

- Declare signal inputs: optional with defaults, required, aliased, transformed.
- Read inputs in templates, `computed()` and `effect()`.
- React to input changes the modern way, and understand `ngOnChanges` for legacy code.
- Design input APIs that are hard to misuse.

## 2. Prerequisites

[Components](../02-components-and-templates/01-components.md), [Signals preview in Module 01](../01-angular-fundamentals/05-angular-vs-react.md).

## 3. What is it?

An **input** is a value a parent passes into a child: `<app-note-card [note]="n" />`. `input()` (stable since v19) creates it as a **read-only signal** on the child.

## 4. Why does Angular need it?

Inputs make presentational components reusable: one `NoteCard` renders any note. Because they're signals, anything derived from them updates automatically.

## 5. How it works internally

The compiler records each `input()` in the component definition. When the parent's binding value changes, Angular sets the child's input signal during the parent's update pass. Anything reading that signal (child template, `computed`) is marked dirty. With OnPush (the v22 default), an input change is one of the triggers that refreshes the child.

Inputs are compared with `Object.is`: passing the **same** object reference again is not a change.

## 6. Syntax and API reference

```typescript
import { input, booleanAttribute, numberAttribute } from '@angular/core';

export class NoteCard {
  note = input.required<Note>();                       // must be bound
  compact = input(false);                              // optional, default false → InputSignal<boolean>
  maxTags = input(3, { transform: numberAttribute });  // accepts "5" from static attributes
  selectable = input(false, { transform: booleanAttribute });  // <app-note-card selectable />
  label = input<string | undefined>();                 // optional, no default → string | undefined
  theme = input('light', { alias: 'cardTheme' });      // parent writes [cardTheme]
}
```

| API | Type | Parent usage |
|---|---|---|
| `input<T>()` | `InputSignal<T \| undefined>` | optional |
| `input(default)` | `InputSignal<T>` | optional with default |
| `input.required<T>()` | `InputSignal<T>` | required — compile error if missing |
| `{ transform }` | changes the accepted type | `booleanAttribute`, `numberAttribute`, or your function |
| `{ alias }` | public name differs from field | avoid unless matching an existing API |

Reading: `this.note()` in the class, `note()` in the template. Input signals are **read-only**; you can't `set()` them.

### Transforms

```typescript
function trimToNull(v: string | null | undefined): string | null {
  const t = v?.trim();
  return t ? t : null;
}
hint = input(null, { transform: trimToNull });   // accepts string | null | undefined, stores string | null
```

Keep transforms **pure and cheap**; they run on every change.

### Reacting to changes

```typescript
// 1. Derived values: computed
preview = computed(() => this.note().content.slice(0, 120));

// 2. Side effects: effect (e.g. tell a third-party map to recenter)
constructor() {
  effect(() => this.map?.setView([this.lat(), this.lng()]));
}

// 3. Writable local copy that resets when the input changes: linkedSignal (Module 08)
draftTitle = linkedSignal(() => this.note().title);
```

### Legacy: `@Input()` and `ngOnChanges`

```typescript
export class OldNoteCard implements OnChanges {
  @Input({ required: true }) note!: Note;
  @Input({ transform: booleanAttribute }) compact = false;

  ngOnChanges(changes: SimpleChanges) {
    if (changes['note'] && !changes['note'].firstChange) {
      console.log('note changed from', changes['note'].previousValue, 'to', changes['note'].currentValue);
    }
  }
}
```

`ngOnChanges` still works with signal inputs, but you rarely need it. Migrate with `ng generate @angular/core:signal-input-migration`.

## 7. Smallest working example

```typescript
// shared/ui/avatar.ts
import { Component, computed, input } from '@angular/core';

@Component({
  selector: 'app-avatar',
  template: `
    @if (src()) {
      <img [src]="src()" [alt]="name()" class="size-full rounded-full object-cover" />
    } @else {
      <span class="grid size-full place-items-center rounded-full bg-blue-600 text-sm font-semibold text-white" aria-hidden="true">{{ initials() }}</span>
      <span class="sr-only">{{ name() }}</span>
    }
  `,
  host: { class: 'inline-block size-9' },
})
export class Avatar {
  name = input.required<string>();
  src = input<string | null>(null);
  protected initials = computed(() =>
    this.name().split(/\s+/).filter(Boolean).slice(0, 2).map(p => p[0]!.toUpperCase()).join(''),
  );
}
```

```html
<app-avatar name="Pranjal Goyal" [src]="user().avatarUrl" />
```

## 8. Step-by-step

1. `name` is required, so forgetting it fails the build.
2. `src` defaults to `null`; the template falls back to initials.
3. `initials` recomputes only when `name` changes.

## 9. Myspce real-world example — `CredentialRow`

```typescript
// features/credentials/components/credential-row.ts
import { Component, booleanAttribute, computed, input } from '@angular/core';
import { CredentialSummary } from '../../../core/models';

@Component({
  selector: 'app-credential-row',
  templateUrl: './credential-row.html',
  host: { class: 'flex items-center gap-4 px-4 py-3' },
})
export class CredentialRow {
  credential = input.required<CredentialSummary>();
  /** When true, hides the username on narrow layouts. */
  dense = input(false, { transform: booleanAttribute });

  protected host = computed(() => {
    const url = this.credential().url;
    if (!url) return null;
    try { return new URL(url).hostname; } catch { return url; }
  });
}
```

```html
<!-- features/credentials/components/credential-row.html -->
<app-service-icon [name]="credential().name" [icon]="credential().icon" />
<div class="min-w-0 flex-1">
  <p class="truncate font-medium">{{ credential().name }}</p>
  <p class="truncate text-xs text-slate-400">{{ host() ?? 'No URL' }}</p>
</div>
@if (!dense()) {
  <p class="hidden truncate text-sm text-slate-300 sm:block">{{ credential().username }}</p>
}
<span class="font-mono tracking-widest text-slate-400" aria-label="Password hidden">••••••••</span>
```

(Add `ServiceIcon` — a small icon component — to `imports` when you build it.)

The row never receives the password — the list API doesn't return it. Revealing is an explicit action ([next note](02-outputs-and-model.md)).

## 10. Advanced example — designing an input API

Bad: many booleans that combine into impossible states.

```typescript
// ❌ What does primary + danger + ghost mean?
primary = input(false); danger = input(false); ghost = input(false); small = input(false);
```

Good: unions that describe one choice each.

```typescript
// ✅ shared/ui/button.ts
variant = input<'primary' | 'secondary' | 'ghost' | 'danger'>('primary');
size = input<'sm' | 'md'>('md');
```

Rules for input design:

- Required for what the component can't work without; defaults for the rest.
- Pass **data** (a `Note`), not view decisions computed by the parent (a `titleColor` string) — unless the variation is genuinely the parent's choice.
- Prefer a few well-typed inputs over many optional ones; use content projection for arbitrary content.

## 11. Visual explanation

```text
Parent template              Child (NoteCard)
[note]="selected()"  ─────►  note = input.required<Note>()  (read-only signal)
                                 │
                                 ├─► template: {{ note().title }}
                                 └─► preview = computed(() => note().content…)
```

## 12. Common mistakes

| Wrong | Why | Right |
|---|---|---|
| `this.note().title = 'x'` | mutates the parent's object; parent isn't notified | emit an output; owner updates immutably |
| Reading a required input in a field initializer: `title = this.note().title` | `NG0950` — not set yet; also wouldn't update | `title = computed(() => this.note().title)` |
| `@Input() note: Note;` with strict mode | "has no initializer" | use `input.required<Note>()` |
| Passing a new object literal each check `[config]="{ dense: true }"` | fine for primitives, but new objects defeat equality | define the object once or pass primitives |
| `effect()` to copy an input into a signal | glitchy | `computed` or `linkedSignal` |

## 13. Debugging guide

| Error | Fix |
|---|---|
| `NG8008: Required input 'note' from component NoteCard must be specified` | bind `[note]` |
| `NG0950: Input "note" is required but no value is available yet` | don't read it in the constructor/field initializers |
| `NG0303: Can't bind to 'notee'` | typo or missing import |
| `NG1054: Input 'x' is bound to both 'x' and 'y'` (v22) | two inputs share a public name via alias |

## 14. Best practices

- Signal inputs for all new code.
- `input.required` for essentials; defaults for the rest.
- Derive with `computed`; never sync with `effect`.
- Keep input objects immutable.

## 15. Angular vs React

`input()` ≈ a prop, but reading is `note()` and it's reactive like state. There's no re-render of the whole component when it changes — only what reads it updates.

## 16. Testing

```typescript
it('shows initials when there is no image', async () => {
  const fixture = TestBed.createComponent(Avatar);
  fixture.componentRef.setInput('name', 'Pranjal Goyal');
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).toContain('PG');
});
```

`componentRef.setInput()` is the correct way to set inputs in tests (it goes through Angular, so OnPush/signals work).

## 17. Practice questions

1. Add a `maxLines` input with `numberAttribute` to `NoteCard` and clamp the preview.
2. Why is `CredentialSummary` a safer input type than `Credential`?
3. Rewrite an `@Input` + `ngOnChanges` component with `input()` + `computed()`.

## 18. Interview questions

- **Beginner — How do you make an input required?** `input.required<T>()`; Angular errors at build time if it isn't bound.
- **Intermediate — How do you react to input changes with signal inputs?** `computed()` for derived values, `effect()` for side effects, `linkedSignal()` for a resettable local copy.
- **Advanced — Why shouldn't a child mutate an input object?** The parent owns the data; mutation bypasses change notifications, breaks OnPush/signal equality, and hides data flow.

## 19. Revision notes

`input()` optional · `input(default)` · `input.required<T>()` · `{ transform, alias }` · read with `()` · read-only · derive with `computed`.

## 20. Completion checklist

- [ ] I built `Avatar` and `CredentialRow` with signal inputs.
- [ ] I used a transform.
- [ ] I can explain NG0950 and fix it.
