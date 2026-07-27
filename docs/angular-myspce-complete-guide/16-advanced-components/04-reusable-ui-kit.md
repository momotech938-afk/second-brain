# The Myspce Reusable UI Kit

Back to [Module 16](README.md) · Previous: [Dynamic components](03-dynamic-components.md) · Next module: [17 Performance](../17-performance/README.md)

## 1. Learning objectives

- Build an accessible modal on the native `<dialog>` element.
- Build a promise-based confirm service and a toast service with Undo.
- Build a dropdown menu with keyboard support (and know when to use CDK Overlay or Angular Aria).
- Design component APIs that are consistent across the kit.

## 2. Prerequisites

Notes 1–3 of this module, [Signals](../08-signals-and-reactivity/README.md).

## 3. What is it?

A small set of shared, presentational building blocks in `shared/ui/` that every Myspce page uses: `PageHeader`, `Card`, `ChipFilter`, `SearchBox`, `EmptyState`, `Toggle`, `Tabs`, `Modal`, `ConfirmDialog`, `Toast`, `Menu`, `Icon`.

## 4. Why does Angular need it?

Consistency (every modal behaves the same), accessibility done once, and faster feature work.

## 5. Options for overlays

| Option | Gives you | When |
|---|---|---|
| Native `<dialog>` + `showModal()` | top layer, backdrop, Escape, inert background, focus handling built into the browser | simple modals — **Myspce default** |
| Angular CDK `Dialog` / `Overlay` | positioning, scroll strategies, focus trap, stacking, injectors | menus/popovers positioned to a trigger, complex dialogs |
| Angular Aria (`@angular/aria`, stable in v22) | headless, accessible primitives (menus, tabs, listbox, combobox…) you style yourself | accessible widgets without building ARIA patterns by hand |
| Angular Material | fully styled components | if you adopt Material Design (not Myspce's look) |

## 6. Modal on native `<dialog>`

```typescript
// shared/ui/modal.ts
import { Component, ElementRef, effect, input, model, viewChild } from '@angular/core';

@Component({
  selector: 'app-modal',
  template: `
    <dialog #dlg class="w-full max-w-lg rounded-2xl bg-white p-0 text-slate-900 backdrop:bg-black/50 dark:bg-slate-900 dark:text-slate-100"
            [attr.aria-labelledby]="titleId" (close)="open.set(false)" (cancel)="onCancel($event)"
            (click)="onBackdrop($event)">
      <div class="p-6">
        <h2 [id]="titleId" class="mb-4 text-lg font-semibold">{{ title() }}</h2>
        <ng-content />
        <div class="mt-6 flex justify-end gap-2"><ng-content select="[actions]" /></div>
      </div>
    </dialog>
  `,
})
export class Modal {
  title = input.required<string>();
  open = model(false);
  dismissible = input(true);
  private dlg = viewChild.required<ElementRef<HTMLDialogElement>>('dlg');
  protected titleId = `modal-${Math.random().toString(36).slice(2)}`;

  constructor() {
    effect(() => {
      const el = this.dlg().nativeElement;
      if (this.open() && !el.open) el.showModal();    // focus moves into the dialog; background becomes inert
      if (!this.open() && el.open) el.close();        // browser returns focus to the previously focused element
    });
  }

  protected onCancel(e: Event) { if (!this.dismissible()) e.preventDefault(); }          // Escape
  protected onBackdrop(e: MouseEvent) {
    if (this.dismissible() && e.target === this.dlg().nativeElement) this.open.set(false);   // click outside content
  }
}
```

```html
<app-modal title="Add Credential" [(open)]="adding">
  <app-credential-form-signal [categories]="categories()" (submitted)="save($event)" />
</app-modal>
```

Accessibility checklist the native dialog covers: modal semantics, focus moves in, background inert, Escape closes, focus returns. You add: a labelled title (`aria-labelledby`), a visible close action, and `autofocus` on the first field if needed.

## 7. Confirm service (promise-based)

Pages and guards just `await confirm.ask(...)`:

```typescript
// shared/ui/confirm/confirm-service.ts
import { Service, signal } from '@angular/core';

export interface ConfirmOptions { title: string; message?: string; confirmLabel?: string; cancelLabel?: string; tone?: 'default' | 'danger'; }
interface Pending extends ConfirmOptions { resolve: (ok: boolean) => void; }

@Service()
export class ConfirmService {
  readonly current = signal<Pending | null>(null);

  ask(opts: ConfirmOptions): Promise<boolean> {
    this.current()?.resolve(false);                       // only one at a time
    return new Promise(resolve => this.current.set({ ...opts, resolve }));
  }
  answer(ok: boolean) {
    this.current()?.resolve(ok);
    this.current.set(null);
  }
}
```

```typescript
// shared/ui/confirm/confirm-host.ts — placed once in Shell
@Component({
  selector: 'app-confirm-host',
  imports: [Modal],
  template: `
    @if (svc.current(); as c) {
      <app-modal [title]="c.title" [open]="true" (openChange)="!$event && svc.answer(false)">
        @if (c.message) { <p class="text-sm text-slate-600 dark:text-slate-300">{{ c.message }}</p> }
        <button actions type="button" (click)="svc.answer(false)">{{ c.cancelLabel ?? 'Cancel' }}</button>
        <button actions type="button" class="rounded-lg px-3 py-2 text-white" autofocus
                [class]="c.tone === 'danger' ? 'bg-rose-600' : 'bg-blue-600'" (click)="svc.answer(true)">
          {{ c.confirmLabel ?? 'Confirm' }}
        </button>
      </app-modal>
    }
  `,
})
export class ConfirmHost { protected svc = inject(ConfirmService); }
```

Usage: `if (await this.confirm.ask({ title: 'Delete note?', message: 'It will move to Trash for 30 days.', confirmLabel: 'Delete', tone: 'danger' })) this.store.remove(id);`

## 8. Toast service with Undo

```typescript
// shared/ui/toast/toast-service.ts
import { Service, signal } from '@angular/core';

export interface ToastAction { label: string; run: () => void; }
export interface Toast { id: number; kind: 'success' | 'error' | 'info'; message: string; action?: ToastAction; }

@Service()
export class ToastService {
  readonly toasts = signal<Toast[]>([]);
  private nextId = 1;

  success(message: string, opts: { action?: ToastAction; ms?: number } = {}) { this.push('success', message, opts); }
  error(message: string, opts: { action?: ToastAction; ms?: number } = {}) { this.push('error', message, { ms: 6000, ...opts }); }
  info(message: string, opts: { action?: ToastAction; ms?: number } = {}) { this.push('info', message, opts); }

  dismiss(id: number) { this.toasts.update(ts => ts.filter(t => t.id !== id)); }

  private push(kind: Toast['kind'], message: string, { action, ms = 4000 }: { action?: ToastAction; ms?: number }) {
    const id = this.nextId++;
    this.toasts.update(ts => [...ts.slice(-2), { id, kind, message, action }]);   // keep at most 3
    setTimeout(() => this.dismiss(id), ms);
  }
}
```

```typescript
// shared/ui/toast/toast-host.ts — placed once in Shell
@Component({
  selector: 'app-toast-host',
  template: `
    <div class="pointer-events-none fixed bottom-4 right-4 z-50 flex w-80 flex-col gap-2" aria-live="polite" aria-atomic="false">
      @for (t of svc.toasts(); track t.id) {
        <div class="pointer-events-auto flex items-center gap-3 rounded-lg px-4 py-3 text-sm text-white shadow-lg"
             [class]="t.kind === 'error' ? 'bg-rose-600' : t.kind === 'success' ? 'bg-emerald-600' : 'bg-slate-800'"
             [attr.role]="t.kind === 'error' ? 'alert' : 'status'">
          <span class="flex-1">{{ t.message }}</span>
          @if (t.action; as a) { <button type="button" class="font-semibold underline" (click)="a.run(); svc.dismiss(t.id)">{{ a.label }}</button> }
          <button type="button" aria-label="Dismiss" (click)="svc.dismiss(t.id)">×</button>
        </div>
      }
    </div>
  `,
})
export class ToastHost { protected svc = inject(ToastService); }
```

This matches the API used throughout the guide (`toast.success('Moved to Trash', { action: { label: 'Undo', run } })`).

## 9. Dropdown menu (the "⋯" on cards)

```typescript
// shared/ui/menu.ts
@Component({
  selector: 'app-menu',
  imports: [ClickOutside],
  template: `
    <div class="relative inline-block" (appClickOutside)="open.set(false)">
      <button #trigger type="button" [attr.aria-expanded]="open()" aria-haspopup="menu" [attr.aria-label]="label()"
              (click)="toggle()" (keydown.arrowdown)="openAndFocus(0, $event)">⋯</button>
      @if (open()) {
        <div role="menu" class="absolute right-0 z-20 mt-1 min-w-36 rounded-lg bg-white p-1 shadow-lg dark:bg-slate-800"
             (keydown)="onKey($event)">
          <ng-content />
        </div>
      }
    </div>
  `,
})
export class Menu {
  label = input('More actions');
  protected open = signal(false);
  private host = inject<ElementRef<HTMLElement>>(ElementRef);
  private trigger = viewChild.required<ElementRef<HTMLButtonElement>>('trigger');

  toggle() { this.open.update(v => !v); }
  openAndFocus(i: number, e: Event) { e.preventDefault(); this.open.set(true); queueMicrotask(() => this.items()[i]?.focus()); }

  private items(): HTMLElement[] { return [...this.host.nativeElement.querySelectorAll<HTMLElement>('[role="menuitem"]')]; }

  protected onKey(e: KeyboardEvent) {
    const items = this.items();
    const i = items.indexOf(document.activeElement as HTMLElement);
    if (e.key === 'ArrowDown') { e.preventDefault(); items[(i + 1) % items.length]?.focus(); }
    if (e.key === 'ArrowUp') { e.preventDefault(); items[(i - 1 + items.length) % items.length]?.focus(); }
    if (e.key === 'Escape') { this.open.set(false); this.trigger().nativeElement.focus(); }
  }
}
```

```html
<app-menu [label]="'Actions for ' + note().title">
  <button role="menuitem" type="button" class="block w-full rounded px-3 py-1.5 text-left text-sm hover:bg-slate-100" (click)="edit.emit()">Edit</button>
  <button role="menuitem" type="button" class="block w-full rounded px-3 py-1.5 text-left text-sm text-rose-600 hover:bg-slate-100" (click)="remove.emit()">Delete</button>
</app-menu>
```

This hand-rolled menu shows the mechanics. Positioning near screen edges, typeahead, and full ARIA menu behaviour are what **Angular Aria** or **CDK Menu** give you for free — prefer them for production.

## 10. API design rules for the kit

| Rule | Example |
|---|---|
| Inputs describe data or a closed set of variants | `variant: 'primary' \| 'ghost' \| 'danger'` |
| Two-way state uses `model()` | `open`, `value`, `checked`, `active` |
| Events are past-tense intent | `submitted`, `cancelled`, `closed` |
| Content via projection, not string inputs, when markup is needed | modal body, page header actions |
| Accessible names required for icon-only controls | `label` input on `Menu` |
| No service calls inside UI components | modal doesn't save; the page does |
| Services for app-wide overlays | `ConfirmService`, `ToastService` |

## 11. Visual explanation

```text
Shell
 ├─ <router-outlet />           pages use <app-modal>, <app-menu>, <app-tabs> locally
 ├─ <app-confirm-host />        renders ConfirmService.current()
 └─ <app-toast-host />          renders ToastService.toasts()
Pages/guards/stores ──inject──► ConfirmService.ask() / ToastService.success()
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `div` overlays without focus management | native `<dialog>` / CDK / Aria |
| Every page has its own confirm markup | `ConfirmService` |
| Toasts without `aria-live` | screen readers miss them |
| Menus that only work with a mouse | arrow keys, Escape, focus return |
| Errors only as toasts | important errors also inline near the cause |

## 13. Debugging guide

- `showModal()` throws "InvalidStateError" → called when already open; guard with `el.open`.
- Focus lost after closing → something re-rendered the trigger (e.g. `@for` without stable `track`).
- Toast not visible over a modal → modal is in the top layer; render toasts inside the dialog or use a top-layer `popover` element.

## 14. Best practices

- Native platform features first (`<dialog>`, `popover`), libraries when they add real value.
- One host per app-wide overlay type, placed in the shell.
- Test keyboard paths, not just clicks.

## 15. Angular vs React

`ConfirmService.ask()` ≈ a `useConfirm()` hook backed by context; `ToastService` ≈ react-hot-toast. In Angular they're DI services, usable from guards and stores, not only components.

## 16. Testing

```typescript
it('resolves true when confirmed', async () => {
  const svc = TestBed.inject(ConfirmService);
  const p = svc.ask({ title: 'Delete?' });
  svc.answer(true);
  await expect(p).resolves.toBe(true);
});

it('keeps at most three toasts', () => {
  const t = TestBed.inject(ToastService);
  ['a', 'b', 'c', 'd'].forEach(m => t.info(m));
  expect(t.toasts().map(x => x.message)).toEqual(['b', 'c', 'd']);
});
```

## 17. Practice questions

1. Add a `size` input (`sm | md | lg`) to `Modal`.
2. Replace the hand-rolled `Menu` with Angular Aria's menu primitives and compare.
3. Make the delete flow: confirm → optimistic remove → toast with Undo → restore.

## 18. Interview questions

- **Q: Why use the native `<dialog>`?** It provides modal semantics, top-layer rendering, inert background, Escape handling and focus management natively.
- **Q: How do you show a confirm dialog from a route guard?** A service that returns a Promise resolved by a host component rendered in the shell.

## 19. Revision notes

`<dialog>` + `showModal()` · `ConfirmService.ask(): Promise<boolean>` · `ToastService` + host with `aria-live` · menus need keyboard support · Angular Aria/CDK for production overlays · consistent API rules.

## 20. Completion checklist

- [ ] `Modal`, `ConfirmService`, `ToastService`, `Menu` built and used in Notes and Credentials.
- [ ] Every overlay works with keyboard only.
