# Cleanup and Memory Leaks

Back to [Module 09](README.md) · Previous: [Change detection](02-change-detection.md) · Next module: [10 Routing](../10-routing/README.md)

## 1. Learning objectives

- Know what Angular cleans up for you and what it doesn't.
- Use `DestroyRef`, `takeUntilDestroyed`, effect `onCleanup` and `ngOnDestroy`.
- Clean up timers, DOM listeners, observers and third-party instances.
- Detect leaks with DevTools.

## 2. Prerequisites

[Lifecycle hooks](01-lifecycle-hooks.md), basic Observables.

## 3. What is it?

A **memory leak** is memory that stays allocated after it's no longer needed — typically a destroyed component still referenced by a timer, a global listener or a long-lived subscription. In SPAs that run for hours (Myspce stays open all day), leaks accumulate and slow the tab down.

## 4. What Angular cleans up automatically

| Thing | Cleaned up when the component is destroyed? |
|---|---|
| Template event listeners `(click)` | ✅ |
| `host` listeners, including `document:`/`window:` | ✅ |
| `effect()` created in the component | ✅ |
| `toSignal()` subscriptions | ✅ |
| Resources (`httpResource`, …) | ✅ |
| `async` pipe subscriptions | ✅ |
| `HttpClient` requests (complete after one response) | ✅ usually harmless; still cancelled if you unsubscribe |
| Your `setInterval`/`setTimeout` | ❌ |
| Your `addEventListener` on `window`/`document` | ❌ |
| Your `.subscribe()` on long-lived Observables (`interval`, router events, store subjects, WebSocket) | ❌ |
| `ResizeObserver`, `IntersectionObserver`, `MutationObserver` | ❌ |
| Third-party instances (Leaflet map, chart, editor) | ❌ |

## 5. How it works — `DestroyRef`

Every injector and every component/directive has a `DestroyRef`. Callbacks registered with `onDestroy` run when that thing is destroyed. Services provided in root get destroyed only when the app is destroyed.

## 6. Syntax and API reference

```typescript
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class Something {
  private destroyRef = inject(DestroyRef);

  constructor() {
    // 1. timers / observers / libraries
    const id = setInterval(() => this.tick(), 60_000);
    this.destroyRef.onDestroy(() => clearInterval(id));

    // 2. Observables — in an injection context, no argument needed
    interval(5000).pipe(takeUntilDestroyed()).subscribe(() => this.poll());
  }

  later() {
    // 3. Outside an injection context, pass the DestroyRef
    this.router.events.pipe(takeUntilDestroyed(this.destroyRef)).subscribe(e => this.log(e));
  }
}

// 4. inside effects
effect(onCleanup => {
  const ro = new ResizeObserver(entries => this.width.set(entries[0]!.contentRect.width));
  ro.observe(this.el.nativeElement);
  onCleanup(() => ro.disconnect());
});

// 5. classic hook
ngOnDestroy() { this.editor?.destroy(); }
```

`onDestroy` returns an unregister function if you need to cancel the callback early.

## 7. Smallest working example

```typescript
// shared/directives/visible.ts — emits when the host scrolls into view
import { Directive, DestroyRef, ElementRef, afterNextRender, inject, output } from '@angular/core';

@Directive({ selector: '[appVisible]' })
export class Visible {
  appVisible = output<void>();
  private el = inject<ElementRef<HTMLElement>>(ElementRef);

  constructor() {
    const destroyRef = inject(DestroyRef);
    afterNextRender(() => {
      const io = new IntersectionObserver(([entry]) => entry?.isIntersecting && this.appVisible.emit());
      io.observe(this.el.nativeElement);
      destroyRef.onDestroy(() => io.disconnect());
    });
  }
}
```

## 8. Step-by-step

1. Inject `DestroyRef` in the constructor (injection context).
2. Create the observer after render (browser only).
3. Register `disconnect` so the observer doesn't keep the element alive.

## 9. Myspce real-world example — idle auto-lock

Settings has **"Lock App on Inactivity"** (confirmed). This service lives for the whole session (root), but it still must stop listening on logout:

```typescript
// core/security/idle-lock.ts
import { Service, inject, signal } from '@angular/core';
import { DOCUMENT } from '@angular/common';
import { fromEvent, merge, switchMap, timer, Subscription, startWith, throttleTime } from 'rxjs';

@Service()
export class IdleLock {
  private doc = inject(DOCUMENT);
  readonly locked = signal(false);
  private sub: Subscription | null = null;

  start(minutes: number) {
    this.stop();
    const activity$ = merge(
      fromEvent(this.doc, 'pointerdown'),
      fromEvent(this.doc, 'keydown'),
      fromEvent(this.doc, 'visibilitychange'),
    ).pipe(throttleTime(1000), startWith(null));

    this.sub = activity$.pipe(switchMap(() => timer(minutes * 60_000))).subscribe(() => this.locked.set(true));
  }

  stop() {
    this.sub?.unsubscribe();
    this.sub = null;
  }

  unlock() { this.locked.set(false); }
}
```

`AuthStore.logout()` calls `idleLock.stop()`; `login()` calls `start(settings.lockTimeoutMin)`. Because it's a root service, `takeUntilDestroyed` wouldn't help (root never destroys); explicit `stop()` is the right lifecycle.

## 10. Advanced example — wrapping a third-party editor

```typescript
// features/notes/components/rich-editor.ts (sketch)
export class RichEditor {
  content = model<unknown>(null);
  private host = viewChild.required<ElementRef<HTMLDivElement>>('host');
  private editor: { destroy(): void; setContent(c: unknown): void; getJSON(): unknown; on(e: string, cb: () => void): void } | null = null;

  constructor() {
    afterNextRender(async () => {
      const { Editor } = await import('@tiptap/core');
      const { default: StarterKit } = await import('@tiptap/starter-kit');
      this.editor = new Editor({ element: this.host().nativeElement, extensions: [StarterKit], content: this.content() ?? '' }) as never;
      this.editor!.on('update', () => this.content.set(this.editor!.getJSON()));
    });
    inject(DestroyRef).onDestroy(() => this.editor?.destroy());
  }
}
```

Lazy import + `afterNextRender` + `DestroyRef` is the standard recipe for any DOM library.

## 11. Visual explanation

```text
NotesPage destroyed
   ├── template listeners ......... removed by Angular
   ├── effects / toSignal ......... stopped by Angular
   ├── setInterval(id) ............ ❌ still firing → holds NotesPage in memory
   └── window.addEventListener ..... ❌ still attached → holds NotesPage in memory
Fix: DestroyRef.onDestroy(() => { clearInterval(id); window.removeEventListener(...) })
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `subscribe()` with no teardown on router events/intervals | `takeUntilDestroyed()` |
| `takeUntilDestroyed()` called in a method | pass `this.destroyRef` |
| `takeUntilDestroyed` not last in `pipe` | put it last so later operators don't keep subscriptions alive |
| Anonymous function passed to `addEventListener` | you can't remove it — keep a reference, or use `host`/`fromEvent` |
| Expecting root services to be destroyed on navigation | they live for the app; manage lifecycle explicitly |

## 13. Debugging guide — finding leaks

1. Chrome DevTools → **Memory** → take a heap snapshot.
2. Navigate to Notes and back to Dashboard ten times.
3. Take another snapshot; switch to **Comparison**.
4. Search for your component class name (`NotesPage`). If the count keeps growing, something references destroyed instances.
5. Click an instance → **Retainers** shows what holds it (a timer, a listener, a subject).

Performance monitor (`Ctrl+Shift+P` → "Show Performance Monitor") shows JS heap size and DOM node count growing over time.

## 14. Best practices

- Prefer Angular-managed constructs (`host` listeners, `toSignal`, resources, effects) — they clean up automatically.
- Every manual resource gets a cleanup **on the same line or the next**.
- Root services with subscriptions expose explicit `start/stop`.

## 15. Angular vs React

`DestroyRef.onDestroy` ≈ returning a cleanup function from `useEffect`. `takeUntilDestroyed()` ≈ unsubscribing in that cleanup.

## 16. Testing

```typescript
it('clears its interval on destroy', () => {
  const clear = vi.spyOn(globalThis, 'clearInterval');
  const fixture = TestBed.createComponent(Clock);
  fixture.destroy();
  expect(clear).toHaveBeenCalled();
});
```

## 17. Practice questions

1. Audit a component with three manual subscriptions and fix each.
2. Why can't `takeUntilDestroyed()` stop a root service's subscription on logout?
3. Reproduce a leak with `window.addEventListener('resize', ...)` and find it with heap snapshots.

## 18. Interview questions

- **Q: What does `takeUntilDestroyed` do?** Completes the source when the associated `DestroyRef` (by default the current injection context) is destroyed.
- **Q: What does Angular clean up automatically?** Template/host listeners, effects, `toSignal`, resources, async pipe; not your timers, global listeners, observers or manual subscriptions.

## 19. Revision notes

Auto: listeners, effects, toSignal, resources, async pipe · Manual: timers, global listeners, observers, long subscriptions, libraries · Tools: `DestroyRef.onDestroy`, `takeUntilDestroyed`, effect `onCleanup`, `ngOnDestroy`.

## 20. Completion checklist

- [ ] `IdleLock` with explicit start/stop.
- [ ] Every `subscribe` in my code has a teardown.
- [ ] I found a leak using heap snapshot comparison.
