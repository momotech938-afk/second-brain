# Built-in Directives

Back to [Module 05](README.md) · Next: [Custom attribute directives](02-custom-attribute-directives.md)

## 1. Learning objectives

- Classify directives: components, attribute directives, structural directives.
- Know which built-in directives are still worth using in Angular 22 and which are superseded.
- Use `NgTemplateOutlet` and `NgComponentOutlet` for reusable templates and dynamic rendering.

## 2. Prerequisites

[Module 03](../03-template-control-flow/README.md).

## 3. What is it?

A **directive** is a class that Angular attaches to elements matching its selector.

| Kind | Has a template? | Changes | Example |
|---|---|---|---|
| Component | yes | renders its own view | `NoteCard` |
| Attribute directive | no | behaviour/appearance of its host element | `RouterLinkActive`, `CopyToClipboard` |
| Structural directive | no (uses an `<ng-template>`) | adds/removes DOM | `*ngIf`, `*appHasRole` |

## 4. Why does Angular need it?

Behaviour like "add a class when this route is active" or "render this template here" is needed on many elements that aren't otherwise related. Directives package that behaviour once.

## 5. How it works

The compiler matches each element in a template against the selectors of directives in the component's `imports` (and built-in ones). For each match it creates an instance attached to the element and wires its inputs/host bindings. Multiple directives can match one element.

## 6. Syntax and API reference — what's still useful

| Directive | Package | Status in v22 | Use instead / notes |
|---|---|---|---|
| `NgIf`, `NgFor`, `NgSwitch*` | `@angular/common` | legacy | `@if`, `@for`, `@switch` |
| `NgClass` | common | works | `[class]="{…}"` preferred |
| `NgStyle` | common | works | `[style]="{…}"` preferred |
| `NgTemplateOutlet` | common | **useful** | render an `ng-template` with context |
| `NgComponentOutlet` | common | **useful** | render a component class chosen at runtime |
| `NgOptimizedImage` (`[ngSrc]`) | common | **useful** | performance ([Module 17](../17-performance/01-rendering-performance.md)) |
| `NgPlural` | common | rare | ICU messages / i18n |
| `RouterLink`, `RouterLinkActive`, `RouterOutlet` | router | **useful** | navigation |
| `NgModel`, `NgForm` | forms | template-driven forms | [Module 11](../11-template-driven-forms/01-template-driven-forms.md) |
| `FormControlName`, `FormGroupDirective` | forms | reactive forms | [Module 12](../12-reactive-forms/01-reactive-forms-core.md) |
| `FormField` (`[formField]`) | `@angular/forms/signals` | Signal Forms | [Module 12](../12-reactive-forms/04-signal-forms.md) |

### `NgTemplateOutlet`

Render a template defined elsewhere, optionally with a context object:

```html
<ng-template #chipTpl let-cat let-active="active">
  <span class="rounded-full px-2 text-xs" [class.ring-2]="active">{{ cat.name }}</span>
</ng-template>

@for (c of categories(); track c.id) {
  <ng-container *ngTemplateOutlet="chipTpl; context: { $implicit: c, active: c.id === selectedId() }" />
}
```

- `let-cat` receives `$implicit`; `let-active="active"` receives the `active` key.
- `<ng-container>` renders no element of its own.
- Typing context: add a static guard or a typed directive if you need strict context types (advanced).

### `NgComponentOutlet`

```html
<ng-container *ngComponentOutlet="widgetComponent(); inputs: { config: widgetConfig() }" />
```

Used for the admin-driven dashboard widget registry in [Module 16](../16-advanced-components/03-dynamic-components.md).

### Router directives

```html
<a routerLink="/notes" [queryParams]="{ category: 'work' }"
   routerLinkActive="bg-blue-600 text-white" [routerLinkActiveOptions]="{ exact: false }"
   ariaCurrentWhenActive="page">Notes</a>
```

## 7. Smallest working example

```typescript
// legacy → modern migration in one component
// before
// <div [ngClass]="{ 'opacity-50': loading }" *ngIf="notes.length; else empty">…</div>
// after
@if (notes().length) {
  <div [class.opacity-50]="loading()">…</div>
} @else {
  <app-empty-state title="No notes" />
}
```

## 8. Step-by-step

1. `*ngIf` + `else` template becomes `@if/@else`.
2. `[ngClass]` with one key becomes `[class.x]`.
3. No `CommonModule`/`NgIf`/`NgClass` imports needed anymore.

## 9. Myspce real-world example — one template, two layouts

Notes can be shown as a grid (Notes page) or a compact list (Favorites page) with the same card body:

```html
<!-- features/notes/components/notes-grid.html -->
<ng-template #body let-note>
  <h3 class="font-semibold">{{ note.title }}</h3>
  <p class="line-clamp-3 text-sm text-slate-600 dark:text-slate-400">{{ note.preview }}</p>
</ng-template>

@if (layout() === 'grid') {
  <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
    @for (n of notes(); track n.id) {
      <article class="rounded-xl p-4" [style.background]="n.color ?? null">
        <ng-container *ngTemplateOutlet="body; context: { $implicit: n }" />
      </article>
    }
  </div>
} @else {
  <ul class="divide-y">
    @for (n of notes(); track n.id) {
      <li class="py-2"><ng-container *ngTemplateOutlet="body; context: { $implicit: n }" /></li>
    }
  </ul>
}
```

Import `NgTemplateOutlet` in the component.

## 10. Advanced example — template inputs (passing templates to children)

A reusable list that lets the parent decide how each row looks:

```typescript
// shared/ui/data-list.ts
@Component({
  selector: 'app-data-list',
  imports: [NgTemplateOutlet],
  template: `
    <ul>
      @for (item of items(); track trackBy()(item)) {
        <li><ng-container *ngTemplateOutlet="row(); context: { $implicit: item }" /></li>
      }
    </ul>
  `,
})
export class DataList<T> {
  items = input.required<T[]>();
  row = input.required<TemplateRef<{ $implicit: T }>>();
  trackBy = input<(item: T) => unknown>((item: T) => item);
}
```

```html
<app-data-list [items]="credentials()" [row]="credRow" [trackBy]="byId" />
<ng-template #credRow let-c>{{ c.name }} — {{ c.username }}</ng-template>
```

## 11. Visual explanation

```text
<a routerLink routerLinkActive class="…">     one element, two directives:
      │             │                           RouterLink (navigates on click)
      ▼             ▼                           RouterLinkActive (adds classes when active)
   instance      instance
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Importing `CommonModule` "just in case" | import only what you use (`NgTemplateOutlet`, `DatePipe`…) |
| Two structural directives on one element (`*ngIf` + `*ngFor`) | not allowed; use `@if`/`@for` blocks or `<ng-container>` |
| `routerLinkActive` highlighting "Dashboard" on every page because its link is `/` | `[routerLinkActiveOptions]="{ exact: true }"` |

## 13. Debugging guide

- `NG0303: Can't bind to 'ngTemplateOutlet'` → import `NgTemplateOutlet`.
- `Can't have multiple template bindings on one element` → split structural directives.

## 14. Best practices

- Built-in control flow over structural directives.
- `[class]`/`[style]` over `NgClass`/`NgStyle`.
- `NgTemplateOutlet` for "same markup, different wrappers".

## 15. Angular vs React

`NgTemplateOutlet` ≈ render props / passing JSX as a prop. `NgComponentOutlet` ≈ `const C = registry[type]; <C {...props} />`.

## 16. Testing

Test template outlets through the host component's rendered DOM; there's nothing special to mock.

## 17. Practice questions

1. Migrate a component using `NgIf`, `NgFor` and `NgClass` to modern syntax.
2. Use `NgTemplateOutlet` to share a "category chip" template between Notes and Tasks.
3. Why can't you put `*ngIf` and `*ngFor` on the same element?

## 18. Interview questions

- **Q: Directive vs component?** A component is a directive with its own template; a directive only adds behaviour to an existing element.
- **Q: When would you still use `NgTemplateOutlet`?** To render a reusable template in multiple places or let a parent supply a template to a child.

## 19. Revision notes

Components ⊂ directives · attribute vs structural · modern: control flow, `[class]`, `[style]` · still useful: `NgTemplateOutlet`, `NgComponentOutlet`, `NgOptimizedImage`, router directives.

## 20. Completion checklist

- [ ] I migrated legacy directives in a sample component.
- [ ] I used `NgTemplateOutlet` with context.
- [ ] I configured `routerLinkActive` for the sidebar.
