# Lab 11 — Solution

Back to [Lab 11](../lab-11-dialogs-and-toasts.md) · [Labs index](../README.md)

`Modal`, `ConfirmService`/`ConfirmHost`, `ToastService`/`ToastHost` and `Menu` are used unchanged from [Module 16 note 4](../../16-advanced-components/04-reusable-ui-kit.md).

## Shell

```html
<!-- core/layout/shell.html (end of file) -->
<app-confirm-host />
<app-toast-host />
```

## `NotesStore.remove` with Undo

This version works with the `rxResource` from Lab 6 (`list`) and the planned `POST /notes/:id/restore`.

```typescript
// features/notes/data-access/notes-store.ts (additions)
private confirm = inject(ConfirmService);
private toast = inject(ToastService);

async remove(id: string): Promise<void> {
  const current = this.list.hasValue() ? this.list.value() : undefined;
  const index = current?.data.findIndex(n => n.id === id) ?? -1;
  if (!current || index < 0) return;
  const note = current.data[index]!;

  const ok = await this.confirm.ask({
    title: 'Delete note?',
    message: `“${note.title}” will move to Trash.`,
    confirmLabel: 'Delete',
    tone: 'danger',
  });
  if (!ok) return;

  this.list.update(v => v && { ...v, data: v.data.filter(n => n.id !== id), meta: { ...v.meta, total: v.meta.total - 1 } });

  try {
    await firstValueFrom(this.api.remove(id, new HttpContext().set(SKIP_ERROR_TOAST, true)));
  } catch (e) {
    this.reinsert(note, index);
    this.toast.error(`Couldn't delete “${note.title}”. ${toMessage(e)}`);
    return;
  }

  this.toast.success('Moved to Trash', { action: { label: 'Undo', run: () => void this.restore(note, index) }, ms: 6000 });
}

private async restore(note: Note, index: number) {
  try {
    await firstValueFrom(this.api.restore(note.id));
    this.reinsert(note, index);
    this.toast.info('Note restored');
  } catch (e) {
    this.toast.error(toMessage(e));
  }
}

private reinsert(note: Note, index: number) {
  this.list.update(v => {
    if (!v || v.data.some(n => n.id === note.id)) return v;
    const data = [...v.data];
    data.splice(Math.min(index, data.length), 0, note);
    return { ...v, data, meta: { ...v.meta, total: v.meta.total + 1 } };
  });
}

/** Convenience for tests/templates */
readonly notes = computed(() => (this.list.hasValue() ? this.list.value().data : []));
```

`NotesApi` gains `restore(id)` (`POST /notes/:id/restore`, planned) and an optional `context` parameter on `remove`. If you built `NotesApi` on `createCrudApi` (Lab 6), `restore` already exists.

## `NoteCard` with menu

```html
<!-- note-card template header (replaces the two buttons from Lab 5) -->
<header class="mb-2 flex items-start justify-between gap-2">
  <h3 class="font-semibold text-slate-900">
    <a [routerLink]="['/notes', note().id]" class="after:absolute after:inset-0">{{ note().title }}</a>
  </h3>
  <div class="relative z-10 flex items-center gap-1">
    <button type="button" (click)="favoriteToggled.emit(note().id)" [attr.aria-pressed]="note().isFavorite"
            [attr.aria-label]="'Favourite ' + note().title">{{ note().isFavorite ? '★' : '☆' }}</button>
    <app-menu [label]="'Actions for ' + note().title">
      <button role="menuitem" type="button" class="block w-full rounded px-3 py-1.5 text-left text-sm hover:bg-slate-100"
              (click)="edit.emit(note().id)">Edit</button>
      <button role="menuitem" type="button" class="block w-full rounded px-3 py-1.5 text-left text-sm hover:bg-slate-100"
              (click)="pinToggled.emit(note().id)">{{ note().pinned ? 'Unpin' : 'Pin' }}</button>
      <button role="menuitem" type="button" class="block w-full rounded px-3 py-1.5 text-left text-sm text-rose-600 hover:bg-slate-100"
              (click)="remove.emit(note().id)">Delete</button>
    </app-menu>
  </div>
</header>
```

The `<article>` gets `class="relative …"` so the "stretched link" (`after:absolute after:inset-0`) makes the whole card clickable, while `relative z-10` keeps the star and menu above it.

```typescript
// note-card.ts additions
edit = output<string>();
remove = output<string>();
```

```html
<!-- notes-page.html -->
<app-note-card [note]="note" (favoriteToggled)="toggleFav($event)" (pinToggled)="store.togglePin($event)"
               (edit)="router.navigate(['/notes', $event])" (remove)="store.remove($event)" />
```

## Unsaved-changes guard (final)

Exactly the [Module 10 note 4](../../10-routing/04-guards-and-resolvers.md#unsaved-changes) version:

```typescript
export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = component =>
  !component.hasUnsavedChanges() || inject(ConfirmService).ask({
    title: 'Discard changes?',
    message: 'You have unsaved changes to this note.',
    confirmLabel: 'Discard',
    tone: 'danger',
  });
```

## Add Credential modal

```typescript
// features/credentials/credentials-page.ts (excerpt)
protected adding = signal(false);
private api = inject(CredentialsApi);
private toast = inject(ToastService);

async save(dto: CreateCredentialDto) {
  try {
    await firstValueFrom(this.api.create(dto));
    this.adding.set(false);
    this.toast.success('Credential saved');
    this.list.reload();
  } catch (e) {
    this.toast.error(toMessage(e));   // modal stays open with the form values intact
  }
}
```

```html
<app-page-header title="Credentials">
  <button actions type="button" class="rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white"
          (click)="adding.set(true)">Add credential</button>
</app-page-header>

<app-modal title="Add credential" [(open)]="adding">
  @if (adding()) {
    <app-credential-form [categories]="categories()" (submitted)="save($event)" (cancelled)="adding.set(false)" />
  }
</app-modal>
```

`@if (adding())` inside the modal recreates the form each time it opens, so it always starts empty.

`CreateCredentialDto` (proposed): `{ name, url, username, password, notes, categoryId }`. These are the fields of the Lab 3 form.

## `RevealedSecret`

```typescript
// features/credentials/components/revealed-secret.ts
import { Component, DestroyRef, computed, inject, input, signal } from '@angular/core';
import { firstValueFrom } from 'rxjs';
import { CredentialsApi } from '../data-access/credentials-api';
import { ToastService } from '../../../shared/ui/toast/toast-service';
import { toMessage } from '../../../core/http/error-messages';

const VISIBLE_SECONDS = 30;

@Component({
  selector: 'app-revealed-secret',
  template: `
    <div class="flex items-center gap-2">
      <code class="min-w-0 flex-1 truncate rounded bg-slate-100 px-2 py-1 font-mono text-sm dark:bg-slate-800"
            [attr.aria-label]="secret() ? 'Password' : 'Password hidden'">{{ secret() ?? '••••••••••' }}</code>
      @if (secret()) {
        <button type="button" (click)="copy()" class="text-sm">Copy</button>
        <button type="button" (click)="hide()" class="text-sm">Hide</button>
        <span class="text-xs text-slate-500" aria-live="off">{{ remaining() }}s</span>
      } @else {
        <button type="button" (click)="reveal()" [disabled]="loading()" class="text-sm">{{ loading() ? 'Revealing…' : 'Reveal' }}</button>
      }
    </div>
  `,
})
export class RevealedSecret {
  credentialId = input.required<string>();

  private api = inject(CredentialsApi);
  private toast = inject(ToastService);
  protected secret = signal<string | null>(null);
  protected loading = signal(false);
  private hideAt = signal(0);
  private now = signal(Date.now());
  protected remaining = computed(() => Math.max(0, Math.ceil((this.hideAt() - this.now()) / 1000)));
  private timer: ReturnType<typeof setInterval> | null = null;

  constructor() {
    inject(DestroyRef).onDestroy(() => this.hide());
  }

  async reveal() {
    this.loading.set(true);
    try {
      const { password } = await firstValueFrom(this.api.reveal(this.credentialId()));
      this.secret.set(password);
      this.hideAt.set(Date.now() + VISIBLE_SECONDS * 1000);
      this.timer = setInterval(() => {
        this.now.set(Date.now());
        if (this.remaining() === 0) this.hide();
      }, 1000);
    } catch (e) {
      this.toast.error(toMessage(e));
    } finally {
      this.loading.set(false);
    }
  }

  async copy() {
    const s = this.secret();
    if (!s) return;
    try {
      await navigator.clipboard.writeText(s);
      this.toast.success('Copied');            // never include the secret
    } catch {
      this.toast.error("Couldn't copy. Select the text and copy manually.");
    }
  }

  hide() {
    if (this.timer) clearInterval(this.timer);
    this.timer = null;
    this.secret.set(null);
  }
}
```

`CredentialsApi.reveal(id)` → `POST /credentials/:id/reveal` returns `{ password: string }`. The endpoint is planned; the response field name is **proposed**.

You could reuse the `CopyToClipboard` directive (outputs `copySucceeded`/`copyFailed`) from Module 05 instead of `copy()`.

## Mock routes

```typescript
const credReveal = path.match(/^\/credentials\/([\w-]+)\/reveal$/);
if (credReveal && req.method === 'POST') {
  if (Date.now() - lastReauthAt > 5 * 60_000) return fail(403, 'REAUTH_REQUIRED', req.url);   // used by the independent task
  const secret = MOCK_SECRETS[credReveal[1]!];
  return secret ? ok({ password: secret }) : fail(404, 'Credential not found', req.url);
}

const restore = path.match(/^\/notes\/([\w-]+)\/restore$/);
if (restore && req.method === 'POST') {
  const n = trash.find(x => x.id === restore[1]);
  if (!n) return fail(404, 'Not in Trash', req.url);
  trash = trash.filter(x => x !== n);
  notes = [n, ...notes];
  return ok(n);
}
```

Update the mock's `DELETE /notes/:id` to move the note into a `trash` array instead of dropping it. `lastReauthAt` starts at `0`; until the independent task is done, initialise it to `Date.now()` so reveal works.

## Independent task — re-auth dialog

```typescript
// core/auth/reauth-dialog.ts
@Service()
export class ReauthDialog {
  readonly pending = signal<((ok: boolean) => void) | null>(null);
  private inflight: Promise<boolean> | null = null;

  ask(): Promise<boolean> {
    return (this.inflight ??= new Promise<boolean>(resolve => this.pending.set(resolve))
      .finally(() => { this.inflight = null; this.pending.set(null); }));
  }
  answer(ok: boolean) { this.pending()?.(ok); }
}
```

`ask()` shares one dialog between concurrent requests, which is the same single-flight idea as token refresh.

```typescript
// core/auth/reauth-host.ts — placed once in Shell
@Component({
  selector: 'app-reauth-host',
  imports: [Modal, FormField, PasswordInput],
  template: `
    @if (dialog.pending()) {
      <app-modal title="Confirm it's you" [open]="true" (openChange)="!$event && dialog.answer(false)">
        <form id="reauth-form" (submit)="confirm($event)" class="space-y-3">
          <p class="text-sm text-slate-600 dark:text-slate-300">Enter your password to reveal this credential.</p>
          <label for="reauth-pw" class="text-sm font-medium">Password</label>
          <app-password-input inputId="reauth-pw" [formField]="pwForm.password" />
          @for (e of pwForm().errors(); track e.kind) { <p role="alert" class="text-sm text-rose-600">{{ e.message }}</p> }
        </form>
        <button actions type="button" (click)="dialog.answer(false)">Cancel</button>
        <button actions type="submit" form="reauth-form" class="rounded-lg bg-blue-600 px-3 py-2 text-white"
                [disabled]="pwForm().submitting()">Confirm</button>
      </app-modal>
    }
  `,
})
export class ReauthHost {
  protected dialog = inject(ReauthDialog);
  private http = inject(HttpClient);
  private api = inject(API_URL);
  protected model = signal({ password: '' });
  protected pwForm = form(this.model, p => required(p.password, { message: 'Enter your password.' }));

  async confirm(e: Event) {
    e.preventDefault();
    await submit(this.pwForm, async f => {
      try {
        await firstValueFrom(this.http.post(`${this.api}/auth/reauth`, { password: f().value().password },
          { context: new HttpContext().set(SKIP_ERROR_TOAST, true) }));
      } catch {
        return { kind: 'wrongPassword', message: 'Incorrect password.' };
      }
      this.model.set({ password: '' });
      this.dialog.answer(true);
      return undefined;
    });
  }
}
```

The interceptor is the Module 13 version (`reauthInterceptor`, registered **after** `authInterceptor`, before the mock). Mock `POST /auth/reauth`: on the correct password set `lastReauthAt = Date.now()` and return 204; otherwise return 401. Because `/auth/reauth` isn't marked `SKIP_AUTH`, a 401 here would trigger a token refresh. To avoid that, the mock returns **400** `"Incorrect password"` for a wrong password. Decide the real status code with the backend.
