# Lab 7 — Solution

Back to [Lab 7](../lab-07-ui-states.md) · [Labs index](../README.md)

## `toMessage` and `errorInterceptor`

Use the versions from [Module 13 note 3](../../13-http-and-api-integration/03-error-handling-retry.md) unchanged:

| File | Exports |
|---|---|
| `core/http/error-messages.ts` | `ApiErrorBody`, `toMessage(err)` |
| `core/http/http-context.ts` | `SKIP_ERROR_TOAST` (and later `SKIP_AUTH` lives in `core/auth/auth-context.ts`) |
| `core/http/error-interceptor.ts` | `errorInterceptor` — toasts only for status 0, 5xx, 429; always rethrows |

```typescript
// core/http/http-context.ts
import { HttpContextToken } from '@angular/common/http';
export const SKIP_ERROR_TOAST = new HttpContextToken<boolean>(() => false);
```

Registration (outermost first) for this lab:

```typescript
provideHttpClient(withInterceptors([
  errorInterceptor,
  ...(environment.useMockApi ? [mockBackendInterceptor] : []),
])),
```

The mock sits **innermost** so every other interceptor behaves exactly as it will against the real server. Lab 8 inserts `authInterceptor` between them.

## `Skeleton`, `NoteCardSkeleton`, `ErrorState`

```typescript
// shared/ui/skeleton.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-skeleton',
  host: { 'aria-hidden': 'true', class: 'block animate-pulse bg-slate-200 dark:bg-slate-800',
          '[style.width]': 'width()', '[style.height]': 'height()', '[class.rounded-full]': 'round()', '[class.rounded-md]': '!round()' },
  template: '',
})
export class Skeleton {
  width = input('100%');
  height = input('1rem');
  round = input(false);
}
```

```typescript
// features/notes/components/note-card-skeleton.ts
@Component({
  selector: 'app-note-card-skeleton',
  imports: [Skeleton],
  template: `
    <div class="flex h-44 flex-col gap-3 rounded-xl bg-white p-4 shadow-sm dark:bg-slate-900">
      <app-skeleton width="60%" height="1.1rem" />
      <app-skeleton /><app-skeleton /><app-skeleton width="80%" />
      <div class="mt-auto flex justify-between"><app-skeleton width="4rem" height=".75rem" /><app-skeleton width="3rem" height=".75rem" round /></div>
    </div>
  `,
})
export class NoteCardSkeleton {}
```

```typescript
// shared/ui/error-state.ts
import { Component, input, output } from '@angular/core';

@Component({
  selector: 'app-error-state',
  template: `
    <div role="alert" class="rounded-xl border border-rose-200 bg-rose-50 p-6 text-center dark:border-rose-900 dark:bg-rose-950">
      <p class="font-medium text-rose-800 dark:text-rose-200">{{ title() }}</p>
      @if (message()) { <p class="mt-1 text-sm text-rose-700 dark:text-rose-300">{{ message() }}</p> }
      <button type="button" data-test="retry" (click)="retry.emit()"
              class="mt-4 rounded-lg bg-rose-600 px-4 py-2 text-sm font-medium text-white">Try again</button>
    </div>
  `,
})
export class ErrorState {
  title = input('Something went wrong.');
  message = input<string | null>(null);
  retry = output<void>();
}
```

## Notes page states

```html
<!-- features/notes/notes-page.html (content area) -->
@let list = store.list;

@if (list.isLoading() && list.hasValue()) {
  <div class="fixed inset-x-0 top-0 z-50 h-0.5 animate-pulse bg-blue-600" role="progressbar" aria-label="Refreshing notes"></div>
}

<section [attr.aria-busy]="list.isLoading()" aria-live="polite">
  @if (list.error(); as err) {
    <app-error-state title="We couldn't load your notes." [message]="toMessage(err)" (retry)="list.reload()" />
  } @else if (!list.hasValue()) {
    <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
      @for (i of skeletons; track i) { <app-note-card-skeleton /> }
    </div>
    <p class="sr-only" role="status">Loading notes…</p>
  } @else if (!store.notes().length) {
    @if (store.query()) {
      <app-empty-state [title]="'No notes match “' + store.query() + '”'" hint="Try another word.">
        <button action type="button" (click)="store.query.set('')">Clear search</button>
      </app-empty-state>
    } @else {
      <app-empty-state title="No notes yet" hint="Capture your first idea.">
        <a action routerLink="/notes/new" class="rounded-lg bg-blue-600 px-4 py-2 text-white">New note</a>
      </app-empty-state>
    }
  } @else {
    <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
      @for (note of store.notes(); track note.id) {
        <app-note-card [note]="note" (favoriteToggled)="toggleFav($event)" />
      }
    </div>
  }
</section>
```

```typescript
// notes-page.ts (class excerpt)
protected readonly skeletons = [1, 2, 3, 4, 5, 6, 7, 8];
protected readonly toMessage = toMessage;
private toast = inject(ToastService);

async toggleFav(id: string) {
  try { await this.store.toggleFavorite(id); }
  catch (e) { this.toast.error(toMessage(e)); }
}
```

## Editor save without duplicate toasts

```typescript
// note-editor-page.ts (excerpt)
async save() {
  this.saveError.set(null);
  try {
    const ctx = new HttpContext().set(SKIP_ERROR_TOAST, true);
    const saved = await firstValueFrom(this.api.create(this.value(), ctx));
    this.toast.success('Note saved');
    await this.router.navigate(['/notes', saved.id]);
  } catch (e) {
    this.saveError.set(toMessage(e));   // shown inline with role="alert"; the draft stays
  }
}
```

For that, `NotesApi.create` gets an optional `context?: HttpContext` parameter passed to `http.post(..., { context })`.

## Dashboard `WidgetFrame`

```typescript
// features/dashboard/widget-frame.ts
import { Component, input } from '@angular/core';
import { Skeleton } from '../../shared/ui/skeleton';
import { ErrorState } from '../../shared/ui/error-state';
import { toMessage } from '../../core/http/error-messages';

@Component({
  selector: 'app-widget-frame',
  imports: [Skeleton, ErrorState],
  template: `
    <section class="rounded-xl bg-white p-4 shadow-sm dark:bg-slate-900" [attr.aria-busy]="res().isLoading()" [attr.aria-labelledby]="id">
      <h2 [id]="id" class="mb-3 font-semibold">{{ title() }}</h2>
      @if (res().error(); as e) {
        <app-error-state title="Couldn't load this widget." [message]="msg(e)" (retry)="reload()" />
      } @else if (!res().hasValue()) {
        <app-skeleton height="1.25rem" /><app-skeleton class="mt-2" height="1.25rem" width="70%" />
      } @else {
        <ng-content />
      }
    </section>
  `,
})
export class WidgetFrame {
  private static n = 0;
  protected id = `widget-${++WidgetFrame.n}`;
  title = input.required<string>();
  // any resource / httpResource / rxResource ref fits this shape
  res = input.required<{ isLoading(): boolean; hasValue(): boolean; error(): unknown; reload(): boolean }>();
  protected msg = toMessage;
  reload() { this.res().reload(); }
}
```

Usage: `<app-widget-frame title="Today's tasks" [res]="today">…list…</app-widget-frame>` with `today = httpResource<Task[]>(() => '/api/v1/dashboard/today')`. A failing widget shows its own error without blanking the others.

## Independent task — offline banner

```typescript
// core/network/network-status.ts
import { DestroyRef, Service, inject, signal } from '@angular/core';
import { DOCUMENT } from '@angular/common';

@Service()
export class NetworkStatus {
  private win = inject(DOCUMENT).defaultView!;
  readonly online = signal(this.win.navigator.onLine);

  constructor() {
    const on = () => this.online.set(true);
    const off = () => this.online.set(false);
    this.win.addEventListener('online', on);
    this.win.addEventListener('offline', off);
    inject(DestroyRef).onDestroy(() => { this.win.removeEventListener('online', on); this.win.removeEventListener('offline', off); });
  }
}
```

```html
<!-- app.html -->
@if (!network.online()) {
  <div role="status" class="sticky top-0 z-50 bg-amber-500 px-4 py-2 text-center text-sm font-medium text-black">
    You're offline — changes won't be saved.
  </div>
}
```

Disable save buttons with `[disabled]="form().submitting() || !network.online()"`. Credentials and Tasks use the same `@if` chain as Notes; the Tasks "Today" list uses `EmptyState` "Nothing due today 🎉".
