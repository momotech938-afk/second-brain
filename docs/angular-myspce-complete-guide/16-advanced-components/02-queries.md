# View and Content Queries

Back to [Module 16](README.md) · Previous: [Content projection](01-content-projection.md) · Next: [Dynamic components](03-dynamic-components.md)

## 1. Learning objectives

- Query elements, components and directives in your own template (`viewChild`, `viewChildren`).
- Query projected content (`contentChild`, `contentChildren`).
- Use `read` to choose what you get (element, component, directive, `TemplateRef`, `ViewContainerRef`).
- Understand query timing and legacy decorator queries.

## 2. Prerequisites

[Content projection](01-content-projection.md).

## 3. What is it?

Queries give your component class a reference to things in its template (view queries) or in content projected into it (content queries). Signal queries (stable since v19) return **signals** that update automatically.

## 4. Why does Angular need it?

Some tasks need direct access: focus an input, call a child's method (`modal.open()`), measure an element, hand a DOM node to a library, find all `Tab` directives projected into `Tabs`.

## 5. How it works internally

The compiler records each query. As views are created/destroyed (e.g. by `@if`/`@for`), Angular updates the query results and notifies the signal. Values become available after the relevant view is created: for `viewChild`, once the component's view exists (read them in `computed`, `effect`, `afterNextRender` or event handlers — not in the constructor).

## 6. Syntax and API reference

```typescript
import { viewChild, viewChildren, contentChild, contentChildren, ElementRef, ViewContainerRef, TemplateRef } from '@angular/core';

searchInput = viewChild<ElementRef<HTMLInputElement>>('search');          // by template ref name → Signal<ElementRef | undefined>
searchRequired = viewChild.required<ElementRef<HTMLInputElement>>('search'); // Signal<ElementRef>, throws if missing when read
modal = viewChild(Modal);                                                  // by component type
cards = viewChildren(NoteCard);                                            // Signal<readonly NoteCard[]>
host = viewChild('slot', { read: ViewContainerRef });                      // choose what to read
tabs = contentChildren(Tab, { descendants: true });                        // projected directives (incl. nested)
footerTpl = contentChild('footer', { read: TemplateRef });
```

| Option | Meaning |
|---|---|
| locator: string | template reference variable name (`#search`) |
| locator: type | component/directive class (or an `InjectionToken`) |
| `read` | what to return: `ElementRef`, `ViewContainerRef`, `TemplateRef`, a directive on that element |
| `descendants` (content queries) | include nested projected descendants (default `false` for `contentChildren`) |

Legacy decorators (read-only knowledge): `@ViewChild('x', { static: true }) x!: ElementRef;`, `@ViewChildren(NoteCard) cards!: QueryList<NoteCard>;`, `@ContentChildren(Tab) tabs!: QueryList<Tab>;`. Values are available in `ngAfterViewInit` / `ngAfterContentInit`. Migrate with `ng generate @angular/core:signal-queries-migration`.

## 7. Smallest working example

```typescript
// features/notes/components/quick-add.ts
@Component({
  selector: 'app-quick-add',
  template: `
    <button type="button" (click)="open()">+ Quick note</button>
    @if (editing()) {
      <input #title class="rounded border px-2 py-1" placeholder="Title" (keydown.escape)="editing.set(false)" />
    }
  `,
})
export class QuickAdd {
  protected editing = signal(false);
  private title = viewChild<ElementRef<HTMLInputElement>>('title');

  constructor() {
    // when the input appears, focus it
    effect(() => this.title()?.nativeElement.focus());
  }
  open() { this.editing.set(true); }
}
```

(Imports: `Component, ElementRef, effect, signal, viewChild` from `@angular/core`.)

## 8. Step-by-step

1. The input only exists while `editing()` is true.
2. `viewChild('title')` is `undefined`, then becomes the `ElementRef` when `@if` renders it.
3. The effect re-runs when the query result changes and focuses the input.

## 9. Myspce real-world example — scroll to and highlight a credential

Dashboard → "AWS Account updated 4 hours ago" → navigates to `/credentials?highlight=<id>`:

```typescript
// features/credentials/pages/credentials-page.ts (excerpt)
highlight = input<string>();                              // from ?highlight=
private rows = viewChildren(CredentialRow, { read: ElementRef });
private rowCmps = viewChildren(CredentialRow);

constructor() {
  afterRenderEffect(() => {                                // runs after render when its signals change
    const id = this.highlight();
    if (!id) return;
    const i = this.rowCmps().findIndex(r => r.credential().id === id);
    const el = this.rows()[i]?.nativeElement as HTMLElement | undefined;
    el?.scrollIntoView({ behavior: 'smooth', block: 'center' });
    el?.classList.add('ring-2', 'ring-blue-500');
  });
}
```

`afterRenderEffect` (v20+) is an effect that runs in the after-render phase — ideal for DOM work that depends on signals. Two `viewChildren` queries on the same locator give you both the component instances and their host elements in the same order.

## 10. Advanced example — content queries for a compound component

`Tabs` from the [previous note](01-content-projection.md) uses `contentChildren(Tab)` to discover panels the parent projected. A form-field wrapper can find the projected input to wire labels and errors:

```typescript
@Component({
  selector: 'app-form-field',
  template: `
    <label [for]="inputId()" class="text-sm font-medium">{{ label() }}</label>
    <ng-content />
    <ng-content select="[hint]" />
  `,
})
export class FormField2 {
  label = input.required<string>();
  private control = contentChild.required(InputRefDirective);     // a tiny directive on the projected <input appInput>
  protected inputId = computed(() => this.control().id());
}
```

(Named `FormField2` here only to avoid confusion with Signal Forms' `FormField` directive — pick a distinct name such as `FieldShell` in real code.)

## 11. Visual explanation

```text
Component template (view)          Projected content (from parent)
viewChild / viewChildren  ◄──────  <input #search>, <app-note-card>
contentChild / contentChildren ◄── <ng-template appTab>, <input appInput>
read: ElementRef | component | directive | TemplateRef | ViewContainerRef
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Reading a view query in the constructor | read in `computed`/`effect`/`afterNextRender`/handlers |
| `viewChild.required` on something inside a false `@if` | use optional `viewChild` |
| Expecting `contentChildren` to see nested descendants | `{ descendants: true }` |
| Using queries to pass data parent→child | inputs |
| DOM manipulation with `nativeElement.innerHTML` | bindings, or sanitised `[innerHTML]` |

## 13. Debugging guide

- `NG0951: Child query result is required but no value is available` → `viewChild.required` read before the element exists or it doesn't exist; use optional.
- Query returns the component but you need the element → `{ read: ElementRef }`.

## 14. Best practices

- Signal queries for new code.
- Prefer declarative bindings; use queries for focus, measurement, imperative child APIs, third-party libraries.
- Pair DOM work with `afterNextRender`/`afterRenderEffect`.

## 15. Angular vs React

`viewChild` ≈ `useRef` attached to an element/component; `viewChildren` ≈ an array of refs; content queries have no direct React equivalent (React passes children as props you can inspect).

## 16. Testing

```typescript
it('focuses the input when opened', async () => {
  const f = TestBed.createComponent(QuickAdd);
  await f.whenStable();
  f.nativeElement.querySelector('button').click();
  await f.whenStable();
  expect(document.activeElement).toBe(f.nativeElement.querySelector('input'));
});
```

(Attach the fixture to the document if focus assertions fail in your environment.)

## 17. Practice questions

1. Use `viewChild(Modal)` to open a modal from the page.
2. Count `NoteCard`s rendered with `viewChildren` and display it.
3. When would `read: ViewContainerRef` be useful?

## 18. Interview questions

- **Q: View vs content queries?** View queries search the component's own template; content queries search content projected into it.
- **Q: When are signal query results available?** After the relevant view is created; they're signals, so read them reactively.

## 19. Revision notes

`viewChild(ren)` · `contentChild(ren)` · `.required` · `read` · `descendants` · signals update with `@if/@for` · `afterRenderEffect` for DOM work.

## 20. Completion checklist

- [ ] Quick-add focus and credential highlight implemented.
- [ ] I migrated one decorator query to a signal query.
