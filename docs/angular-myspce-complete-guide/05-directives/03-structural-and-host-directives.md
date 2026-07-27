# Structural Directives and Host Directives

Back to [Module 05](README.md) · Previous: [Custom attribute directives](02-custom-attribute-directives.md) · Next module: [06 Pipes](../06-pipes/README.md)

## 1. Learning objectives

- Understand `<ng-template>`, `TemplateRef`, `ViewContainerRef` and the `*` syntax.
- Write a custom structural directive (`*appHasRole`).
- Compose behaviour onto components with `hostDirectives`.
- Know when a structural directive is the wrong tool.

## 2. Prerequisites

[Custom attribute directives](02-custom-attribute-directives.md).

## 3. What is it?

- A **structural directive** decides whether/how many times a template is rendered. It receives the template as a `TemplateRef` and renders it into a `ViewContainerRef`.
- **Host directives** let a component or directive apply other directives to its own host element, declared in its metadata.

## 4. Why does Angular need it?

Built-in `@if`/`@for` cover most cases. Custom structural directives remain useful for **reusable conditional rendering with logic** — e.g. "render only if the current user has one of these roles" — used across many templates. Host directives solve "every button-like component should also have `Ripple`, `Tooltip` and `FocusRing`" without inheritance.

## 5. How it works internally

`*appHasRole="['admin']"` is sugar. The compiler rewrites:

```html
<a *appHasRole="['admin']" routerLink="/admin">Admin</a>
```

into

```html
<ng-template [appHasRole]="['admin']">
  <a routerLink="/admin">Admin</a>
</ng-template>
```

The directive sits on the `<ng-template>`. `inject(TemplateRef)` gives it the template; `inject(ViewContainerRef)` gives it the location to render into. `createEmbeddedView(templateRef, context)` renders one instance; `clear()` removes all.

## 6. Syntax and API reference

| API | Purpose |
|---|---|
| `inject(TemplateRef<C>)` | the template the directive is attached to |
| `inject(ViewContainerRef)` | anchor where views are inserted |
| `vcr.createEmbeddedView(tpl, ctx?)` | render the template (returns `EmbeddedViewRef`) |
| `vcr.clear()` / `vcr.remove(i)` / `vcr.length` | manage rendered views |
| `static ngTemplateContextGuard(dir, ctx): ctx is C` | types `let-` variables for strict templates |
| `hostDirectives: [Dir, { directive: Dir2, inputs: ['a'], outputs: ['b'] }]` | compose directives onto a host |

## 7. Smallest working example — `*appRepeat`

```typescript
// shared/directives/repeat.ts
import { Directive, TemplateRef, ViewContainerRef, effect, inject, input, numberAttribute } from '@angular/core';

interface RepeatContext { $implicit: number; }

@Directive({ selector: '[appRepeat]' })
export class Repeat {
  count = input(0, { alias: 'appRepeat', transform: numberAttribute });
  private tpl = inject<TemplateRef<RepeatContext>>(TemplateRef);
  private vcr = inject(ViewContainerRef);

  constructor() {
    effect(() => {
      this.vcr.clear();
      for (let i = 0; i < this.count(); i++) this.vcr.createEmbeddedView(this.tpl, { $implicit: i });
    });
  }

  static ngTemplateContextGuard(_d: Repeat, ctx: unknown): ctx is RepeatContext { return true; }
}
```

```html
<!-- skeleton placeholders -->
<div *appRepeat="6; let i" class="h-24 animate-pulse rounded-xl bg-slate-200"></div>
```

(For this case `@for (i of [0,1,2,3,4,5]; track i)` is simpler — the example teaches the mechanics.)

## 8. Step-by-step

1. `*appRepeat="6; let i"` → `<ng-template [appRepeat]="6" let-i>`.
2. The `effect` re-renders when `count` changes. Here `effect` is appropriate: it syncs to the **view container**, which is outside the signal graph.
3. `ngTemplateContextGuard` tells strict templates that `i` is a `number`.

## 9. Myspce real-world example — `*appHasRole`

Myspce has an admin panel (confirmed) and roles `super_admin | admin | user` (assumption A3). The sidebar shows "Admin" only for admins.

```typescript
// shared/directives/has-role.ts
import { Directive, TemplateRef, ViewContainerRef, effect, inject, input } from '@angular/core';
import { AuthStore } from '../../core/auth/auth-store';
import type { User } from '../../core/models';

type Role = User['role'];

@Directive({ selector: '[appHasRole]' })
export class HasRole {
  roles = input.required<Role[]>({ alias: 'appHasRole' });
  elseTpl = input<TemplateRef<unknown> | null>(null, { alias: 'appHasRoleElse' });

  private tpl = inject(TemplateRef);
  private vcr = inject(ViewContainerRef);
  private auth = inject(AuthStore);
  private shown: 'main' | 'else' | null = null;

  constructor() {
    effect(() => {
      const role = this.auth.user()?.role;
      const allowed = !!role && this.roles().includes(role);
      const next = allowed ? 'main' : this.elseTpl() ? 'else' : null;
      if (next === this.shown) return;            // avoid re-creating the same view
      this.vcr.clear();
      if (next === 'main') this.vcr.createEmbeddedView(this.tpl);
      if (next === 'else') this.vcr.createEmbeddedView(this.elseTpl()!);
      this.shown = next;
    });
  }
}
```

```html
<app-nav-item *appHasRole="['admin', 'super_admin']" label="Admin" route="/admin" icon="shield" />

<button *appHasRole="['super_admin']; else readOnly" type="button" (click)="resetTheme()">Reset theme</button>
<ng-template #readOnly><p class="text-sm text-slate-500">Only the owner can reset the theme.</p></ng-template>
```

`*appHasRole="[...]; else readOnly"` maps `else` to the `appHasRoleElse` input (the microsyntax prefixes keys with the selector name).

> Hiding UI is **not security**. The NestJS API must reject non-admin requests to `/admin/*` regardless. Frontend role checks only improve UX.

Equivalent with built-ins: `@if (auth.hasRole('admin')) { … }`. Use the directive when the check appears in many templates or needs extra logic.

## 10. Advanced example — `hostDirectives`

Give every interactive card the same keyboard + focus behaviour without repeating attributes:

```typescript
// shared/directives/focus-ring.ts
@Directive({ host: { class: 'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-500' } })
export class FocusRing {}

// shared/directives/activate-on-enter.ts
@Directive({ host: { tabindex: '0', '(keydown.enter)': 'activated.emit()', '(keydown.space)': '$event.preventDefault(); activated.emit()' } })
export class ActivateOnEnter {
  activated = output<void>();
}

// features/notes/components/note-card.ts
@Component({
  selector: 'app-note-card',
  hostDirectives: [FocusRing, ActivateOnEnter],
  host: { role: 'button', '(click)': 'open.emit()' },
  templateUrl: './note-card.html',
})
export class NoteCard {
  note = input.required<Note>();
  open = output<void>();

  constructor() {
    // Enter/Space (from the host directive) and click both mean "open".
    inject(ActivateOnEnter).activated.subscribe(() => this.open.emit());
  }
}
```

**Trap:** you might be tempted to write `{ directive: ActivateOnEnter, outputs: ['activated: open'] }` to expose the directive's output as `open`. Combined with the component's own `open` output, that's two outputs with one public name — a compile error in v22 (`NG1054`). Expose it under a unique name, or forward it manually as above.

Host directives:

- are applied **statically** (no conditional application),
- must be standalone,
- expose inputs/outputs only if listed (`inputs: ['x']`, `outputs: ['y: alias']`),
- can be injected by the host (`inject(ActivateOnEnter)`).

## 11. Visual explanation

```text
<ng-template [appHasRole]="['admin']">          ViewContainerRef anchor (a comment node)
   <app-nav-item …/>     ── TemplateRef ──►     createEmbeddedView() inserts the item here
</ng-template>                                   clear() removes it
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Re-creating views on every effect run | track what's shown and only change on transitions |
| Structural directive where `@if` would do | prefer `@if` for one-off conditions |
| Relying on `*appHasRole` for security | enforce on the server |
| Exposing a host directive output with a name the component already uses | rename or forward manually |

## 13. Debugging guide

- `NG0201: No provider for TemplateRef` → you used the directive without `*` (on a normal element). Use `*appHasRole` or put it on `<ng-template>`.
- Typed `let-` variables show as `any` → add `ngTemplateContextGuard`.
- `NG1054` with host directives → duplicate public input/output names.

## 14. Best practices

- Reach for built-in control flow first.
- Keep structural directives tiny and pure in what they decide.
- Use host directives for cross-cutting behaviours shared by many components.

## 15. Angular vs React

A structural directive ≈ a wrapper component like `<RequireRole roles={['admin']}>{children}</RequireRole>`. Host directives ≈ composing multiple hooks into a component, but applied to the host element declaratively.

## 16. Testing

```typescript
@Component({ imports: [HasRole], template: `<span *appHasRole="['admin']">secret</span>` })
class Host {}

it('renders only for admins', async () => {
  const user = signal<User | null>({ role: 'user' } as User);
  TestBed.configureTestingModule({ providers: [{ provide: AuthStore, useValue: { user } }] });
  const fixture = TestBed.createComponent(Host);
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).not.toContain('secret');
  user.set({ role: 'admin' } as User);
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).toContain('secret');
});
```

## 17. Practice questions

1. Add an `appHasRoleElse` example to the Settings "Customization" tab.
2. Write `*appFeature="'memories'"` that renders only when the admin flag is on.
3. Build a `Tooltip` host directive and apply it to `IconButton`.

## 18. Interview questions

- **Q: What does `*` do?** Desugars the element into an `<ng-template>` with the directive bound to it.
- **Q: `TemplateRef` vs `ViewContainerRef`?** The first is the blueprint; the second is where instances of it are inserted.
- **Q: What are host directives?** Directives a component applies to its own host element via metadata, enabling composition without inheritance.

## 19. Revision notes

`*dir` → `<ng-template [dir]>` · `TemplateRef` + `ViewContainerRef` · `createEmbeddedView`/`clear` · context guard for types · `hostDirectives` for composition · UI checks ≠ security.

## 20. Completion checklist

- [ ] `*appHasRole` with `else` works and is tested.
- [ ] I composed two host directives onto a component.
- [ ] I can desugar `*` syntax on paper.
