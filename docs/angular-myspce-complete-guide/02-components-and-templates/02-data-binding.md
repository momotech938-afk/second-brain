# Data Binding

Back to [Module 02](README.md) · Previous: [Components](01-components.md) · Next: [Class, style and attribute binding](03-class-style-attribute-binding.md)

## 1. Learning objectives

- Use interpolation, property binding, event binding and two-way binding correctly.
- Explain the direction of each binding.
- Use template reference variables and `@let`.
- Understand what template type checking enforces.

## 2. Prerequisites

[Components](01-components.md), [DOM events](../00-prerequisites/05-dom-css-responsive.md).

## 3. What is it?

Binding is how the template and the component class talk. Data flows **class → template** (show values) and **template → class** (react to user actions).

## 4. Why does Angular need it?

Without binding you'd write `element.textContent = …` and `addEventListener` by hand and keep them in sync yourself. Bindings are declarative: you describe *what* should be shown, Angular keeps the DOM in sync.

## 5. How it works internally

The compiler turns each binding into an instruction in the update pass. For `{{ note().title }}` Angular stores the last value; on each check it re-reads the expression and touches the DOM **only if the value changed**. Event bindings become DOM listeners that call your method and then mark the component for checking.

## 6. Syntax and API reference

| Binding | Syntax | Direction | Example |
|---|---|---|---|
| Interpolation | `{{ expr }}` | class → view (text) | `{{ user().name }}` |
| Property | `[prop]="expr"` | class → view (DOM property / input) | `[disabled]="saving()"`, `[note]="n"` |
| Event | `(event)="statement"` | view → class | `(click)="save()"` |
| Two-way | `[(prop)]="signalOrField"` | both | `[(value)]="query"` |
| Template ref | `#name` | template-local reference | `<input #q>` then `q.value` |
| Local variable | `@let name = expr;` | template-local value | `@let count = notes().length;` |

### Interpolation

```html
<h1>Good evening, {{ user().name }} 👋</h1>
<p>{{ notes().length }} notes · last edit {{ lastEdited() | date: 'shortTime' }}</p>
<img [src]="user().avatarUrl ?? '/avatar.svg'" [alt]="user().name + ' avatar'" />
```

Interpolation always produces a **string**. Use property binding when you need a non-string value.

### Property binding

Binds to a **DOM property** or a **component input**, not an HTML attribute:

```html
<input [value]="query()" [disabled]="loading()" />
<button type="submit" [disabled]="form.invalid">Save</button>
<app-note-card [note]="note" [compact]="true" />
```

`[disabled]="false"` really sets `disabled = false`; writing `disabled="false"` (no brackets) would set the attribute to the string `"false"` — which still disables the button.

### Event binding

```html
<button type="button" (click)="toggleFavorite(note.id)">☆</button>
<input (input)="query.set($any($event.target).value)" />
<input (keydown.enter)="search()" (keydown.escape)="clear()" />
<form (submit)="$event.preventDefault(); save()"></form>
```

- `$event` is the DOM event (or the payload of a component `output()`).
- Key filters: `keydown.enter`, `keydown.escape`, `keydown.control.k`, `keyup.shift.tab`.
- `$any(...)` disables type checking for one expression — use sparingly; prefer a template ref:

```html
<input #q (input)="query.set(q.value)" />
```

### Two-way binding

`[(x)]="v"` is sugar for `[x]="v" (xChange)="v = $event"`. With signals, Angular writes into the signal for you:

```typescript
query = signal('');
```

```html
<app-search-box [(value)]="query" />   <!-- child declares value = model('') -->
```

For native inputs without forms, use `[value]` + `(input)`, or `FormsModule`'s `[(ngModel)]` ([Module 11](../11-template-driven-forms/01-template-driven-forms.md)), or Signal Forms' `[formField]` ([Module 12](../12-reactive-forms/04-signal-forms.md)).

### Template reference variables

```html
<input #search type="search" placeholder="Search anything…" />
<button type="button" (click)="search.focus()">Focus</button>

<app-modal #confirm />
<button type="button" (click)="confirm.open()">Delete</button>
```

A ref on a plain element gives the element; on a component it gives the component instance; with `#f="ngForm"` it gives the directive instance named by `exportAs`.

### `@let`

```html
@let s = summary();
<p>{{ s.notes }} notes, {{ s.credentials }} credentials</p>
@let pct = (done() / total()) * 100;
<div class="h-2 rounded bg-slate-200"><div class="h-2 rounded bg-blue-500" [style.width.%]="pct"></div></div>
```

`@let` values are read-only and scoped to the template block where they're declared.

### Template expression rules

Allowed: property access, method calls, signals, operators, `?.`, `??`, ternaries, pipes, literals, template literals (v19.2+), `typeof` (v19+), `in` (v20+), `void` (v20+), exponent `**`.
Not allowed: assignments (except in event bindings), `new`, `++`/`--`, bitwise operators, global objects like `window` or `console` (they're not in template scope).

Keep expressions **cheap and side-effect free**; they run on every check of the view.

### Template type checking (strict by default in v22)

```html
{{ note().titel }}                 <!-- ❌ Property 'titel' does not exist on type 'Note' -->
<app-note-card [note]="task" />     <!-- ❌ Type 'Task' is not assignable to type 'Note' -->
@if (note().category) { {{ note().category.name }} }   <!-- ✅ narrowed by the @if (v22 understands ?. guards too) -->
```

## 7. Smallest working example

```typescript
// greeting.ts
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `
    <input #name placeholder="Your name" (keydown.enter)="greet(name.value)" />
    <button type="button" (click)="greet(name.value)" [disabled]="!name.value">Greet</button>
    <p>{{ message() }}</p>
  `,
})
export class Greeting {
  message = signal('');
  greet(name: string) { this.message.set(`Hello, ${name}!`); }
}
```

## 8. Step-by-step

1. `#name` references the input element.
2. `(keydown.enter)` and `(click)` call `greet` with its current value.
3. `[disabled]="!name.value"` — note: this only re-evaluates when the view is checked (e.g., after an event). That's fine here because typing fires events; but it's a hint that state belongs in signals.
4. `message()` is a signal read; updating it re-renders the `<p>`.

## 9. Myspce real-world example — the dashboard greeting header

```typescript
// features/dashboard/components/greeting-header.ts
import { Component, computed, input } from '@angular/core';
import { DatePipe } from '@angular/common';

@Component({
  selector: 'app-greeting-header',
  imports: [DatePipe],
  template: `
    <section class="relative overflow-hidden rounded-2xl bg-slate-900 p-6 text-white">
      <div class="flex items-start justify-between gap-4">
        <div>
          <p class="text-sm text-slate-300">{{ greeting() }},</p>
          <h1 class="text-3xl font-bold">{{ name() }} 👋</h1>
        </div>
        <div class="text-right text-sm text-slate-300">
          <p>{{ now() | date: 'EEE, d MMM y' }}</p>
          <p class="text-xl font-semibold text-white">{{ now() | date: 'hh:mm a' }}</p>
        </div>
      </div>
      @if (quote()) {
        <blockquote class="mt-4 max-w-md text-slate-200">“{{ quote() }}”</blockquote>
      }
    </section>
  `,
})
export class GreetingHeader {
  name = input.required<string>();
  quote = input<string | null>(null);
  now = input.required<Date>();

  protected greeting = computed(() => {
    const h = this.now().getHours();
    return h < 12 ? 'Good morning' : h < 17 ? 'Good afternoon' : 'Good evening';
  });
}
```

The parent provides `now` from a ticking signal (built in [Module 08](../08-signals-and-reactivity/02-computed-and-effect.md)) so the greeting updates itself when the hour changes.

## 10. Advanced example — keyboard shortcut to focus global search (Ctrl+K)

```typescript
// core/layout/topbar.ts
import { Component, ElementRef, viewChild } from '@angular/core';

@Component({
  selector: 'app-topbar',
  template: `
    <label class="sr-only" for="global-search">Search</label>
    <input #search id="global-search" type="search" placeholder="Search anything…  (Ctrl+K)"
           class="w-full rounded-lg bg-slate-100 px-3 py-2 dark:bg-slate-800"
           (keydown.escape)="search.blur()" />
  `,
  host: { '(document:keydown.control.k)': 'focusSearch($event)', '(document:keydown.meta.k)': 'focusSearch($event)' },
})
export class Topbar {
  private search = viewChild.required<ElementRef<HTMLInputElement>>('search');
  focusSearch(e: Event) {
    e.preventDefault();
    this.search().nativeElement.focus();
  }
}
```

`host` listeners can target `document:` or `window:` too. `viewChild` is covered in [Module 16](../16-advanced-components/02-queries.md).

## 11. Visual explanation

```text
             ┌──────────── {{ }}  [prop]  ───────────►┐
 Component   │                                        │  Template / DOM
   class     │◄──────────── (event) ──────────────────┤
             └◄──────── [(prop)] both ways ──────────►┘
```

## 12. Common mistakes

| Wrong | Why | Right |
|---|---|---|
| `[disabled]="true"` vs `disabled="{{ x }}"` | interpolation into boolean attributes gives strings | `[disabled]="x"` |
| `{{ notes.length }}` when `notes` is a signal | prints `undefined`/function | `{{ notes().length }}` |
| `(click)="save"` | references the function, doesn't call it | `(click)="save()"` |
| `{{ heavyCalc() }}` with expensive work | runs on every check | `computed()` |
| `[value]="query()"` + `(input)` and also `[(ngModel)]` on the same input | two sources of truth | pick one approach |

## 13. Debugging guide

- `NG0303: Can't bind to 'note' since it isn't a known property of 'app-note-card'` → the input doesn't exist (name typo) or the component isn't imported.
- `NG0100: ExpressionChangedAfterItHasBeenCheckedError` (dev mode) → a binding's value changed during the same check; usually caused by mutating state in a getter or lifecycle hook. Move the change to an event or a signal.
- Template typing errors point to the exact line in the `.html` file — read them like TypeScript errors.

## 14. Best practices

- Bind to signals; read them with `()`.
- Use template refs for "read this element's value now"; use signals for state that other parts depend on.
- Use `@let` to avoid repeating long signal reads.
- Never put side effects in bindings.

## 15. Angular vs React

| React | Angular |
|---|---|
| `{note.title}` | `{{ note().title }}` |
| `disabled={saving}` | `[disabled]="saving()"` |
| `onClick={() => save()}` | `(click)="save()"` |
| controlled `value` + `onChange` | `[(value)]` with `model()` |
| `ref={inputRef}` | `#input` or `viewChild()` |

## 16. Testing

```typescript
it('greets on click', async () => {
  const fixture = TestBed.createComponent(Greeting);
  await fixture.whenStable();
  const input: HTMLInputElement = fixture.nativeElement.querySelector('input');
  input.value = 'Pranjal';
  input.dispatchEvent(new Event('input'));
  fixture.nativeElement.querySelector('button').click();
  await fixture.whenStable();
  expect(fixture.nativeElement.querySelector('p').textContent).toContain('Hello, Pranjal!');
});
```

## 17. Practice questions

1. Why does `<button disabled="false">` still disable the button?
2. Build the "Today's Tasks" header with a "View all" link and a count badge using `@let`.
3. Add a `(keydown.control.n)` shortcut on the Notes page that opens "New Note".
4. Which expressions are not allowed in templates, and why?

## 18. Interview questions

- **Beginner — Property vs attribute binding?** Property binding sets the DOM property; attribute binding (`[attr.x]`) sets the HTML attribute. Some things only exist as attributes (ARIA, `colspan` on SVG).
- **Intermediate — What does `[(x)]` desugar to?** `[x]="v" (xChange)="v = $event"`.
- **Advanced — Why are template expressions restricted?** They're compiled to efficient update code run on each check; restricting them prevents side effects and keeps them statically type-checkable.

## 19. Revision notes

`{{ }}` text · `[ ]` property/input · `( )` event · `[( )]` two-way · `#ref` element/instance · `@let` local value · strict templates catch typos.

## 20. Completion checklist

- [ ] I built `GreetingHeader` with all four binding types.
- [ ] I added Ctrl+K to focus search.
- [ ] I can explain each binding's direction without notes.
