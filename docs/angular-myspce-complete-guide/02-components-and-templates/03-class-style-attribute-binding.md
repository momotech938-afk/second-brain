# Class, Style and Attribute Binding

Back to [Module 02](README.md) · Previous: [Data binding](02-data-binding.md) · Next: [Composition and organisation](04-composition-and-organization.md)

## 1. Learning objectives

- Toggle single classes and sets of classes from state.
- Bind inline styles with units.
- Bind HTML attributes and ARIA attributes.
- Combine these with Tailwind without breaking Tailwind's class scanning.

## 2. Prerequisites

[Data binding](02-data-binding.md).

## 3. What is it?

Specialised property bindings for the `class` attribute, the `style` attribute, and arbitrary HTML attributes.

## 4. Why does Angular need it?

Myspce's UI is state-driven: the active sidebar item, the selected chip, a completed task (strikethrough), category colours, progress bars. Class/style bindings express those states declaratively.

## 5. How it works

Angular **merges** static classes, `[class]` bindings, `[class.x]` bindings and host class bindings. It tracks what each binding added, so removing a binding's class doesn't remove a static one.

## 6. Syntax reference

```html
<!-- one class -->
<a [class.bg-blue-600]="active()" [class.text-white]="active()">Notes</a>

<!-- many classes: string, array or object -->
<div [class]="'rounded-xl p-4 ' + toneClass()"></div>
<div [class]="['rounded-xl', 'p-4', toneClass()]"></div>
<div [class]="{ 'line-through text-slate-400': done(), 'font-medium': !done() }"></div>

<!-- one style, with unit suffix -->
<div [style.width.%]="progress()"></div>
<div [style.height.px]="height()"></div>
<div [style.background-color]="note().color ?? 'white'"></div>
<div [style.--accent]="category().color"></div>     <!-- CSS custom property -->

<!-- many styles -->
<div [style]="{ width: progress() + '%', opacity: loading() ? 0.5 : 1 }"></div>

<!-- attributes -->
<button [attr.aria-label]="'Copy password for ' + cred().name">⧉</button>
<td [attr.colspan]="wide() ? 2 : null">…</td>          <!-- null removes the attribute -->
<div [attr.data-testid]="'note-' + note().id"></div>
<a [attr.aria-current]="active() ? 'page' : null">…</a>
```

`NgClass` and `NgStyle` still exist but `[class]` and `[style]` do everything they do with less overhead. Prefer the bindings.

### Tailwind rule: write complete class names

Tailwind generates CSS only for class names it finds **as complete strings** in your source.

```typescript
// ❌ Tailwind can't see 'bg-emerald-500' — it's built at runtime
tone = computed(() => `bg-${this.color()}-500`);

// ✅ map to complete names
const TONE: Record<Tone, string> = { green: 'bg-emerald-500', red: 'bg-rose-500', blue: 'bg-blue-500' };
tone = computed(() => TONE[this.color()]);
```

For **user-defined** colours (category colours from the admin panel), Tailwind classes can't work at all; bind a style or a CSS variable instead:

```html
<span class="rounded-full px-2 text-xs bg-(--chip) text-white" [style.--chip]="category().color">{{ category().name }}</span>
```

(`bg-(--chip)` is Tailwind v4 syntax for `background-color: var(--chip)`.)

## 7. Smallest working example

```typescript
// shared/ui/progress-bar.ts
import { Component, computed, input } from '@angular/core';

@Component({
  selector: 'app-progress-bar',
  template: `
    <div class="h-2 w-full overflow-hidden rounded-full bg-slate-200 dark:bg-slate-800"
         role="progressbar" [attr.aria-valuenow]="clamped()" aria-valuemin="0" aria-valuemax="100"
         [attr.aria-label]="label()">
      <div class="h-full rounded-full bg-emerald-500 transition-[width]" [style.width.%]="clamped()"></div>
    </div>
  `,
})
export class ProgressBar {
  value = input.required<number>();
  label = input('Progress');
  protected clamped = computed(() => Math.min(100, Math.max(0, Math.round(this.value()))));
}
```

## 8. Step-by-step

1. `[style.width.%]` sets `width: 50%` from the number 50.
2. `role="progressbar"` + `aria-valuenow` make it announced by screen readers.
3. `clamped` keeps the value valid without trusting the caller.

## 9. Myspce real-world example — sidebar nav item

```typescript
// core/layout/nav-item.ts
import { Component, input } from '@angular/core';
import { RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-nav-item',
  imports: [RouterLink, RouterLinkActive],
  template: `
    <a [routerLink]="route()"
       routerLinkActive="bg-blue-600 text-white shadow-sm"
       [routerLinkActiveOptions]="{ exact: exact() }"
       ariaCurrentWhenActive="page"
       class="flex items-center gap-3 rounded-lg px-3 py-2 text-sm text-slate-600 hover:bg-slate-100
              dark:text-slate-300 dark:hover:bg-slate-800">
      <app-icon [name]="icon()" class="size-4" />
      <span>{{ label() }}</span>
      @if (badge()) {
        <span class="ml-auto rounded-full bg-slate-200 px-2 text-xs dark:bg-slate-700">{{ badge() }}</span>
      }
    </a>
  `,
})
export class NavItem {
  label = input.required<string>();
  route = input.required<string>();
  icon = input.required<string>();
  badge = input<number | null>(null);
  exact = input(false);
}
```

`routerLinkActive` adds classes when the route is active; `ariaCurrentWhenActive` sets `aria-current="page"` for screen readers.

## 10. Advanced example — task row states

```html
<!-- features/tasks/components/task-item.html -->
<li class="flex items-center gap-3 rounded-lg px-3 py-2"
    [class]="{
      'bg-rose-50 dark:bg-rose-950/40': overdue() && !done(),
      'opacity-60': done()
    }">
  <input type="checkbox" class="size-4" [checked]="done()" (change)="toggled.emit(task().id)"
         [attr.aria-label]="'Mark ' + task().title + (done() ? ' as not done' : ' as done')" />
  <span class="flex-1" [class.line-through]="done()">{{ task().title }}</span>
  @if (task().category; as cat) {
    <span class="rounded-full px-2 py-0.5 text-xs text-white bg-(--chip)" [style.--chip]="cat.color">{{ cat.name }}</span>
  }
  <time class="text-xs text-slate-500" [attr.datetime]="task().dueAt">{{ task().dueAt | date: 'h:mm a' }}</time>
</li>
```

## 11. Visual explanation

```text
static class="flex gap-3"  +  [class]="{opacity-60: done}"  +  [class.line-through]="done"
                    └──────── merged by Angular ────────┘
result when done:  class="flex gap-3 opacity-60 line-through"
```

## 12. Common mistakes

| Wrong | Fix |
|---|---|
| `class="{{ active ? 'a' : 'b' }}"` | `[class.a]="active()"` |
| `[attr.aria-label]` on a component input expecting a string | Use the input: `[label]="…"` |
| `[style.width]="50"` (no unit) | `[style.width.%]="50"` or `'50%'` |
| Dynamic Tailwind class strings | Map to complete class names |
| Hex colours hard-coded in components | Theme tokens or CSS variables |

## 13. Debugging guide

- Class not applied? Inspect the element: is the class in the DOM? If yes but no styles, Tailwind didn't generate it (dynamic string). If no, the binding expression is false.
- `[attr.x]` with `undefined` leaves the attribute; use `null` to remove it.

## 14. Best practices

- `[class.x]` for one toggle; `[class]="{…}"` for several; never string concatenation for state.
- Use `null` to remove attributes.
- Every interactive state must be visible **and** announced (ARIA).

## 15. Angular vs React

`[class]="{ a: x }"` ≈ `className={clsx({ a: x })}`. `[style.width.%]` has no React equivalent (React uses `style={{ width: x + '%' }}`).

## 16. Testing

```typescript
it('strikes through completed tasks', async () => {
  const fixture = TestBed.createComponent(TaskItem);
  fixture.componentRef.setInput('task', { ...task, status: 'done' });
  await fixture.whenStable();
  expect(fixture.nativeElement.querySelector('span').classList).toContain('line-through');
});
```

## 17. Practice questions

1. Build the category chip for Notes with a user-defined colour.
2. Make the "This Week Progress" bar from the dashboard (5/10 tasks = 50%).
3. Add `aria-pressed` to the favourite star button.

## 18. Interview questions

- **Q: `[class]` vs `NgClass`?** Both accept strings/arrays/objects; `[class]` is built in, needs no import and integrates with Angular's class merging.
- **Q: How do you remove an attribute with binding?** Bind `null`.

## 19. Revision notes

`[class.x]` · `[class]="{}"` · `[style.prop.unit]` · `[style.--var]` · `[attr.x]` (null removes) · complete Tailwind class names.

## 20. Completion checklist

- [ ] `NavItem`, `ProgressBar` and `TaskItem` built.
- [ ] Every icon button has an `aria-label`.
- [ ] No dynamic Tailwind strings in my code.
