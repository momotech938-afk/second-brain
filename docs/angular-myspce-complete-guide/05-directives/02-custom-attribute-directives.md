# Custom Attribute Directives

Back to [Module 05](README.md) · Previous: [Built-in directives](01-built-in-directives.md) · Next: [Structural and host directives](03-structural-and-host-directives.md)

## 1. Learning objectives

- Create attribute directives with inputs, outputs and `host` bindings.
- Access the host element safely with `ElementRef` and `Renderer2` when bindings aren't enough.
- Inject services into directives and clean up correctly.
- Recognise and migrate `@HostListener` / `@HostBinding`.

## 2. Prerequisites

[Components](../02-components-and-templates/01-components.md), [Inputs](../04-component-communication/01-inputs.md), [Outputs](../04-component-communication/02-outputs-and-model.md).

## 3. What is it?

A class decorated with `@Directive({ selector: '[appX]' })`. When an element in a template has the `appX` attribute (and the directive is in the component's `imports`), Angular creates an instance bound to that element.

## 4. Why does Angular need it?

Some behaviour applies to **any** element: copy text to the clipboard, autofocus, detect clicks outside, show a tooltip. A component would force a wrapper element; a directive attaches to what's already there.

## 5. How it works internally

The directive gets the same injection context as components. `inject(ElementRef)` gives its host element. `host: {…}` bindings compile into the host element's update/listener instructions, exactly like a component's `host`. A directive lives and dies with its host element.

## 6. Syntax and API reference

```typescript
import { Directive, ElementRef, Renderer2, inject, input, output, DestroyRef } from '@angular/core';

@Directive({
  selector: '[appCopyToClipboard]',        // attribute selector
  exportAs: 'copy',                        // <button appCopyToClipboard #c="copy">
  host: {
    '(click)': 'copy()',                   // listener on host
    '[attr.aria-label]': 'ariaLabel()',    // attribute binding on host
    '[class.text-emerald-400]': 'copied()',
    type: 'button',                        // static attribute
  },
})
export class CopyToClipboard {
  text = input.required<string>({ alias: 'appCopyToClipboard' });   // value passed on the selector attribute
  copiedChange = output<boolean>();
  private el = inject<ElementRef<HTMLElement>>(ElementRef);
  private renderer = inject(Renderer2);
  private destroyRef = inject(DestroyRef);
}
```

| Tool | Use for |
|---|---|
| `host: { '(event)': 'method($event)' }` | listen on the host (or `document:`/`window:` targets) |
| `host: { '[prop]': 'expr' }` | bind properties/attributes/classes/styles on the host |
| `inject(ElementRef)` | access the native element (escape hatch) |
| `inject(Renderer2)` | DOM changes that work with SSR and custom renderers |
| `inject(DestroyRef)` | register cleanup |
| `input()` aliased to the selector | `<button [appCopyToClipboard]="value">` |

### Legacy decorators

```typescript
@HostBinding('class.active') active = false;
@HostListener('click', ['$event']) onClick(e: MouseEvent) {}
```

Angular's style guide recommends the `host` property instead. Both compile to the same thing.

## 7. Smallest working example — autofocus

```typescript
// shared/directives/autofocus.ts
import { Directive, ElementRef, afterNextRender, booleanAttribute, inject, input } from '@angular/core';

@Directive({ selector: '[appAutofocus]' })
export class Autofocus {
  enabled = input(true, { alias: 'appAutofocus', transform: booleanAttribute });
  private el = inject<ElementRef<HTMLElement>>(ElementRef);

  constructor() {
    afterNextRender(() => {
      if (this.enabled()) this.el.nativeElement.focus();
    });
  }
}
```

```html
<input appAutofocus placeholder="Note title" />
```

## 8. Step-by-step

1. `selector: '[appAutofocus]'` matches the attribute.
2. The input is aliased to the selector so `appAutofocus` alone (empty string → `true` via `booleanAttribute`) or `[appAutofocus]="false"` both work.
3. `afterNextRender` runs once, after the element is in the DOM, and only in the browser (SSR-safe).

## 9. Myspce real-world example — copy password with auto-clear

The Credentials design has a copy button on each row. Requirements (from the build plan): copy, show feedback, **clear the clipboard after 30 s**.

```typescript
// shared/directives/copy-to-clipboard.ts
import { Directive, DestroyRef, inject, input, output, signal } from '@angular/core';

@Directive({
  selector: '[appCopyToClipboard]',
  exportAs: 'copy',
  host: {
    '(click)': 'copy()',
    '[attr.data-copied]': 'copied() ? "" : null',
  },
})
export class CopyToClipboard {
  /** Text to copy, or a function that resolves it (e.g. fetches a secret). */
  source = input.required<string | (() => Promise<string>)>({ alias: 'appCopyToClipboard' });
  clearAfterMs = input(30_000);
  copySucceeded = output<void>();
  copyFailed = output<unknown>();

  readonly copied = signal(false);
  private timers: ReturnType<typeof setTimeout>[] = [];

  constructor() {
    inject(DestroyRef).onDestroy(() => this.timers.forEach(clearTimeout));
  }

  async copy(): Promise<void> {
    try {
      const src = this.source();
      const text = typeof src === 'function' ? await src() : src;
      await navigator.clipboard.writeText(text);
      this.copied.set(true);
      this.copySucceeded.emit();
      this.timers.push(setTimeout(() => this.copied.set(false), 2_000));
      const ms = this.clearAfterMs();
      if (ms > 0) {
        this.timers.push(setTimeout(async () => {
          // Only clear if the clipboard still holds our text (don't wipe something the user copied later).
          try { if ((await navigator.clipboard.readText()) === text) await navigator.clipboard.writeText(''); } catch { /* read permission denied */ }
        }, ms));
      }
    } catch (err) {
      this.copyFailed.emit(err);
    }
  }
}
```

```html
<!-- features/credentials/components/credential-row.html -->
<button type="button" class="rounded p-2 hover:bg-slate-800"
        [appCopyToClipboard]="revealSecret"
        #c="copy"
        (copySucceeded)="toast.success('Password copied')"
        (copyFailed)="toast.error('Could not copy')"
        [attr.aria-label]="c.copied() ? 'Copied' : 'Copy password for ' + credential().name">
  @if (c.copied()) { ✓ } @else { ⧉ }
</button>
```

```typescript
// credential-row.ts
protected revealSecret = () => firstValueFrom(this.api.reveal(this.credential().id)).then(r => r.secret);
```

Notes:

- `navigator.clipboard` needs a **secure context** (HTTPS or localhost) and a user gesture.
- `readText()` may be blocked by permissions; the `catch` keeps the copy working anyway.
- In a stricter design the row would emit `copyRequested` and the page would do the API call ([Module 04](../04-component-communication/02-outputs-and-model.md)); the directive version shows how to accept an async source.

## 10. Advanced example — click outside (for dropdown menus)

```typescript
// shared/directives/click-outside.ts
import { Directive, ElementRef, inject, output } from '@angular/core';

@Directive({
  selector: '[appClickOutside]',
  host: { '(document:pointerdown)': 'onDocPointerDown($event)' },
})
export class ClickOutside {
  appClickOutside = output<PointerEvent>();
  private el = inject<ElementRef<HTMLElement>>(ElementRef);

  onDocPointerDown(e: PointerEvent) {
    if (!this.el.nativeElement.contains(e.target as Node)) this.appClickOutside.emit(e);
  }
}
```

```html
@if (menuOpen()) {
  <div role="menu" class="absolute right-0 z-10 rounded-lg bg-slate-900 p-1 shadow-lg"
       (appClickOutside)="menuOpen.set(false)" (keydown.escape)="menuOpen.set(false)">
    <button role="menuitem" type="button" (click)="edit()">Edit</button>
    <button role="menuitem" type="button" (click)="remove()">Delete</button>
  </div>
}
```

The `document:` listener is added only while the menu exists (the `@if` creates/destroys the directive), so there's no cost when closed. For production menus prefer CDK Overlay or Angular Aria, which also handle focus management.

### When to use `Renderer2`

```typescript
// Add a class to an element you don't control via a template
this.renderer.addClass(this.el.nativeElement, 'ring-2');
this.renderer.setAttribute(this.el.nativeElement, 'aria-busy', 'true');
const unlisten = this.renderer.listen(this.el.nativeElement, 'scroll', () => this.onScroll());
inject(DestroyRef).onDestroy(unlisten);
```

Prefer `host` bindings; use `Renderer2` for imperative cases (dynamic listeners, third-party elements).

## 11. Visual explanation

```text
<button [appCopyToClipboard]="revealSecret" #c="copy" (copySucceeded)="…">
   │         │                                  │          │
   │         └ input aliased to selector        │          └ output
   │                                            └ exportAs ref → c.copied()
   └ host element (click listener added by host: { '(click)': 'copy()' })
```

## 12. Common mistakes

| Mistake | Why | Fix |
|---|---|---|
| `el.nativeElement.innerHTML = userText` | XSS risk, bypasses sanitization | bind with `[textContent]` or `[innerHTML]` (sanitised) |
| `document.addEventListener` without cleanup | memory leaks, handlers on destroyed views | `host: {'(document:x)'}` or `DestroyRef` |
| Touching the DOM in the constructor | element may not be rendered | `afterNextRender` |
| Directive selector without prefix (`[copy]`) | collides with other libraries/attributes | `[appCopyToClipboard]` |
| Forgetting to add the directive to `imports` | silently does nothing (attribute is just an attribute) | add it |

The last one is sneaky: an unknown **attribute** isn't an error, so a missing directive import just means nothing happens.

## 13. Debugging guide

- Directive "does nothing": check `imports`; in DevTools, select the element and run `ng.getDirectives($0)`.
- `NG0201 No provider for ElementRef`: you called `inject(ElementRef)` outside an injection context (e.g. inside a setTimeout). Inject in a field initializer.
- Clipboard `NotAllowedError`: not a secure context or no user gesture.

## 14. Best practices

- One behaviour per directive; compose with `hostDirectives` ([next note](03-structural-and-host-directives.md)).
- `host` for bindings; `Renderer2` only when needed; raw DOM only as a last resort.
- Clean up timers, listeners, observers in `DestroyRef`.

## 15. Angular vs React

Attribute directives ≈ custom hooks that take a `ref` (`useClickOutside(ref, cb)`), but attached declaratively in markup instead of wiring refs.

## 16. Testing

Test directives through a tiny host component:

```typescript
// shared/directives/click-outside.spec.ts
import { Component, signal } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { ClickOutside } from './click-outside';

@Component({
  imports: [ClickOutside],
  template: `<div id="inside" (appClickOutside)="count.set(count() + 1)">inside</div><p id="outside">outside</p>`,
})
class Host { count = signal(0); }

describe('ClickOutside', () => {
  it('emits only for clicks outside the host', async () => {
    const fixture = TestBed.createComponent(Host);
    await fixture.whenStable();
    const el: HTMLElement = fixture.nativeElement;
    el.querySelector('#inside')!.dispatchEvent(new PointerEvent('pointerdown', { bubbles: true }));
    el.querySelector('#outside')!.dispatchEvent(new PointerEvent('pointerdown', { bubbles: true }));
    expect(fixture.componentInstance.count()).toBe(1);
  });
});
```

The component under test must be attached to the document for `document:` listeners — `TestBed` fixtures are attached by default in recent versions; if not, append `fixture.nativeElement` to `document.body` first.

## 17. Practice questions

1. Write `appTruncateTooltip` that sets `title` on the host only when its text is truncated.
2. Convert a directive using `@HostListener` and `@HostBinding` to `host`.
3. Add `aria-live` feedback to `CopyToClipboard`.

## 18. Interview questions

- **Beginner — Attribute directive vs component?** Directive adds behaviour to an existing element and has no template.
- **Intermediate — Why prefer `host` over `@HostListener`?** It's declarative metadata, consistent with components, recommended by the style guide, and keeps all host bindings in one place.
- **Advanced — Why use `Renderer2` instead of `nativeElement`?** It abstracts the platform (SSR, web workers, custom renderers) and avoids direct DOM coupling.

## 19. Revision notes

`@Directive({ selector: '[appX]', host, exportAs })` · input aliased to selector · `inject(ElementRef/Renderer2/DestroyRef)` · `afterNextRender` for DOM work · always clean up.

## 20. Completion checklist

- [ ] `Autofocus`, `ClickOutside`, `CopyToClipboard` built and tested.
- [ ] No raw `document.addEventListener` without cleanup in my code.
- [ ] I can explain what happens if I forget to import a directive.
