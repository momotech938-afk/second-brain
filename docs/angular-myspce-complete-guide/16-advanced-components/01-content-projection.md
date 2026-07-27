# Content Projection

Back to [Module 16](README.md) · Next: [View and content queries](02-queries.md)

## 1. Learning objectives

- Project content with default and named `<ng-content>` slots.
- Use `ngProjectAs` and fallback content.
- Understand when projected content is created and destroyed.
- Pass templates with `ng-template` for conditional or repeated rendering.

## 2. Prerequisites

[Components](../02-components-and-templates/01-components.md), [Built-in directives](../05-directives/01-built-in-directives.md).

## 3. What is it?

Content projection lets a parent put markup **inside** a child component's layout:

```html
<app-card>
  <h3>Angular Notes</h3>          <!-- projected into <ng-content> inside app-card -->
</app-card>
```

## 4. Why does Angular need it?

Layout components (cards, page headers, modals, tabs) shouldn't know what goes inside them. Projection keeps them generic without dozens of inputs.

## 5. How it works internally

- The **parent** creates projected nodes (they belong to the parent's view, bindings refer to the parent's class).
- The child decides **where** they appear with `<ng-content select="…">`.
- Projected content is **always instantiated** by the parent, even if the child never renders the `<ng-content>` (e.g. inside a false `@if`). Components inside are created and run their lifecycle.
- To render content lazily or multiple times, pass an `<ng-template>` instead and render it with `ngTemplateOutlet`.

## 6. Syntax and API reference

```html
<!-- child: card.html -->
<article class="rounded-xl border p-4">
  <header class="mb-2 flex items-center justify-between">
    <ng-content select="[card-title]" />
    <ng-content select="[card-actions]" />
  </header>
  <ng-content />                                    <!-- default slot: everything not matched -->
  <footer class="mt-3 text-xs text-slate-500">
    <ng-content select="app-card-footer, [card-footer]">No footer</ng-content>   <!-- fallback content (v18+) -->
  </footer>
</article>
```

```html
<!-- parent -->
<app-card>
  <h3 card-title>Angular Notes</h3>
  <button card-actions type="button" aria-label="More">⋯</button>
  <p>Components, services, routing…</p>
  <ng-container ngProjectAs="[card-footer]">Updated 2 hours ago</ng-container>
</app-card>
```

| Selector form | Matches |
|---|---|
| `select="[card-title]"` | elements with attribute `card-title` |
| `select="app-card-footer"` | that element/component |
| `select=".x"` | class |
| (none) | default slot |
| `ngProjectAs="[card-footer]"` | makes an element (often `ng-container`) match a slot |

## 7. Smallest working example — `PageHeader`

```typescript
// shared/ui/page-header.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-page-header',
  template: `
    <div class="mb-6 flex flex-wrap items-center justify-between gap-3">
      <div>
        <h1 class="text-2xl font-bold">{{ title() }}</h1>
        <ng-content select="[subtitle]" />
      </div>
      <div class="flex items-center gap-2"><ng-content select="[actions]" /></div>
    </div>
    <ng-content />
  `,
})
export class PageHeader {
  title = input.required<string>();
}
```

```html
<app-page-header title="Credentials">
  <p subtitle class="text-sm text-slate-500">Securely store and manage your accounts.</p>
  <app-search-box actions placeholder="Search credentials…" [(value)]="query" />
  <button actions type="button" class="rounded-lg bg-blue-600 px-3 py-2 text-sm text-white" (click)="add()">+ Add Credential</button>
</app-page-header>
```

## 8. Step-by-step

1. `[actions]` elements go to the right side; `[subtitle]` under the title.
2. Bindings (`[(value)]="query"`, `(click)="add()"`) refer to the **parent's** class.
3. The header stays identical on every page.

## 9. Myspce real-world example — conditional content with templates (Tabs)

The Settings page and Tasks page use tabs. Inactive tab content shouldn't be created until shown, so panels take templates:

```typescript
// shared/ui/tabs/tab.ts
import { Directive, TemplateRef, inject, input } from '@angular/core';

@Directive({ selector: 'ng-template[appTab]' })
export class Tab {
  label = input.required<string>({ alias: 'appTab' });
  id = input.required<string>();
  readonly template = inject(TemplateRef);
}
```

```typescript
// shared/ui/tabs/tabs.ts
import { Component, computed, contentChildren, model } from '@angular/core';
import { NgTemplateOutlet } from '@angular/common';
import { Tab } from './tab';

@Component({
  selector: 'app-tabs',
  imports: [NgTemplateOutlet],
  template: `
    <div role="tablist" class="mb-4 flex gap-2 border-b border-slate-200 dark:border-slate-800" (keydown)="onKey($event)">
      @for (t of tabs(); track t.id()) {
        <button type="button" role="tab" [id]="'tab-' + t.id()" [attr.aria-selected]="current() === t.id()"
                [attr.aria-controls]="'panel-' + t.id()" [tabindex]="current() === t.id() ? 0 : -1"
                class="-mb-px border-b-2 px-3 py-2 text-sm"
                [class]="current() === t.id() ? 'border-blue-600 text-blue-600' : 'border-transparent text-slate-500'"
                (click)="active.set(t.id())">{{ t.label() }}</button>
      }
    </div>
    @for (t of tabs(); track t.id()) {
      @if (current() === t.id()) {
        <div role="tabpanel" [id]="'panel-' + t.id()" [attr.aria-labelledby]="'tab-' + t.id()" tabindex="0">
          <ng-container *ngTemplateOutlet="t.template" />
        </div>
      }
    }
  `,
})
export class Tabs {
  protected tabs = contentChildren(Tab);
  /** Two-way bindable active tab id; null means "the first tab". */
  active = model<string | null>(null);
  protected current = computed(() => this.active() ?? this.tabs()[0]?.id() ?? null);

  protected onKey(e: KeyboardEvent) {
    const ids = this.tabs().map(t => t.id());
    const i = Math.max(0, ids.indexOf(this.current() ?? ''));
    if (e.key === 'ArrowRight') this.active.set(ids[(i + 1) % ids.length]!);
    if (e.key === 'ArrowLeft') this.active.set(ids[(i - 1 + ids.length) % ids.length]!);
  }
}
```

```html
<!-- features/memories/pages/memories-page.html -->
<app-tabs [(active)]="view">
  <ng-template appTab="Timeline" id="timeline"><app-memory-timeline [memories]="memories()" /></ng-template>
  <ng-template appTab="Gallery" id="gallery"><app-memory-gallery [memories]="memories()" /></ng-template>
  <ng-template appTab="Map" id="map"><app-memory-map [pins]="pins()" /></ng-template>
</app-tabs>
```

Only the active panel's components exist — the Leaflet map isn't created until "Map" is clicked. (For production tabs, Angular Aria provides accessible tab primitives; see [note 4](04-reusable-ui-kit.md).)

`current` derives the effective tab: the bound value, or the first tab when nothing is selected yet — no effects or timing tricks needed.

## 10. Advanced example — template inputs with context

A list component where the parent controls row rendering and the child controls layout/empty state:

```typescript
@Component({
  selector: 'app-list',
  imports: [NgTemplateOutlet],
  template: `
    @for (item of items(); track $index) {
      <ng-container *ngTemplateOutlet="row(); context: { $implicit: item, index: $index }" />
    } @empty {
      <ng-content select="[empty]" />
    }
  `,
})
export class List<T> {
  items = input.required<T[]>();
  row = input.required<TemplateRef<{ $implicit: T; index: number }>>();
}
```

## 11. Visual explanation

```text
Parent template                       Child template (app-card)
<app-card>                            <header>
  <h3 card-title>…</h3>  ───────────►   <ng-content select="[card-title]">
  <p>…</p>               ───────────►   </header> <ng-content>
</app-card>
Projected nodes are created by the PARENT (even if the child hides the slot).
<ng-template> is only a blueprint — the CHILD decides if/when/how often to render it.
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Wrapping `<ng-content>` in `@if` to "lazy load" content | content is still created; pass `ng-template` instead |
| Projecting the same `<ng-content>` twice | content projects once (into the first matching slot) |
| Expecting the child's class members in projected bindings | projected bindings use the parent's context |
| Too many slots for a "generic" component | split into variants |

## 13. Debugging guide

- Content not appearing → selector mismatch (attribute name typo) or element is inside a wrapper that doesn't match; use `ngProjectAs`.
- Child lifecycle runs for hidden projected content → switch to `ng-template`.

## 14. Best practices

- Named slots by attribute (`[actions]`, `[subtitle]`) for readability.
- `ng-template` for conditional/repeated/lazy content.
- Fallback content for optional slots.

## 15. Angular vs React

`<ng-content>` ≈ `props.children`; named slots ≈ separate props containing JSX (`header={<h3/>}`); `ng-template` + context ≈ render props.

## 16. Testing

Test projection through a host component:

```typescript
@Component({ imports: [PageHeader], template: `<app-page-header title="Notes"><button actions>New</button></app-page-header>` })
class Host {}
it('projects actions', async () => {
  const f = TestBed.createComponent(Host);
  await f.whenStable();
  expect(f.nativeElement.querySelector('h1').textContent).toBe('Notes');
  expect(f.nativeElement.querySelector('button').textContent).toBe('New');
});
```

## 17. Practice questions

1. Build `Card` with title/actions/default/footer slots and fallback footer.
2. Convert Settings tabs (currently routes) into `Tabs` for a modal. Which is better for Settings and why?
3. Why is the Map tab cheaper with `ng-template` than with `<ng-content>`?

## 18. Interview questions

- **Q: When is projected content instantiated?** When the parent's view is created — regardless of whether the child displays it.
- **Q: How do you project conditionally?** Pass an `ng-template` and render it with `ngTemplateOutlet` when needed.

## 19. Revision notes

`<ng-content select>` slots · default slot · fallback content · `ngProjectAs` · parent context · always instantiated · `ng-template` + `ngTemplateOutlet` for lazy/repeated.

## 20. Completion checklist

- [ ] `PageHeader`, `Card`, `Tabs` built.
- [ ] Map tab created lazily.
