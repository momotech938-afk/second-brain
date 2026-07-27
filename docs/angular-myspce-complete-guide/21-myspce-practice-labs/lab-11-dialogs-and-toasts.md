# Lab 11 — Dialogs, Confirmations, Menus and Toasts

Back to [Labs](README.md) · Solution: [lab-11-solution.md](solutions/lab-11-solution.md) · Previous: [Lab 10](lab-10-search-filter-sort-paginate.md) · Next: [Lab 12](lab-12-tests.md)

## 1. Learning objective

Add the interaction layer the designs need: an accessible `Modal` on native `<dialog>`, a promise-based `ConfirmService`, a `ToastService` with **Undo**, a card actions `Menu`, and a credential "reveal → copy → auto-hide" flow. Replace the `window.confirm` from Lab 4.

## 2. Prerequisites

[Module 16 note 4](../16-advanced-components/04-reusable-ui-kit.md), [Module 05 note 2](../05-directives/02-custom-attribute-directives.md) (`CopyToClipboard`, `ClickOutside`), [Lab 7](lab-07-ui-states.md), [Lab 9](lab-09-protected-routes.md).

## 3. Problem statement

The designs show "New Note" and "Add Credential" buttons, card "⋯" menus, and copy buttons on credentials. Deleting must be safe (planned API: soft delete to Trash, `POST /notes/:id/restore`) and every overlay must work with a keyboard and a screen reader.

## 4. Expected behaviour

- "Add Credential" opens a modal with the Lab 3 form. Escape or a backdrop click closes it; focus returns to the button. Saving shows "Credential saved".
- Note card "⋯" → Edit / Pin / Delete. Arrow keys move through the items; Escape closes and returns focus.
- Delete note → confirm dialog (danger tone) → note disappears immediately → toast "Moved to Trash" with **Undo** → Undo restores it in place.
- Leaving a dirty editor → styled confirm dialog instead of `window.confirm`.
- Credential "Reveal" → `POST /credentials/:id/reveal` → password shows for 30 s with a countdown, then hides. "Copy" copies and toasts "Copied" (the toast never contains the secret).
- At most 3 toasts. Errors use `role="alert"`, others `role="status"`.

## 5. Relevant concepts

`<dialog>.showModal()`, `model()` for `open`, content projection with `select`, signals-based services rendered by a host component, `Promise` APIs for dialogs, optimistic updates with rollback, `navigator.clipboard`, timers with cleanup.

## 6. Step-by-step implementation

1. Copy `Modal`, `ConfirmService` + `ConfirmHost`, `ToastService` + `ToastHost` and `Menu` from Module 16 note 4. Put `<app-confirm-host />` and `<app-toast-host />` once in `Shell`.
2. Credentials page: `adding = signal(false)`; "Add Credential" sets it; `<app-modal title="Add credential" [(open)]="adding">` wraps the Lab 3 `CredentialForm`. On submit, call `CredentialsApi.create`, close, toast.
3. Note card: add `<app-menu>` with Edit / Pin / Delete. Add `edit`, `pinToggled`, `remove` outputs.
4. `NotesStore.remove(id)`: confirm → optimistic remove from the list → `DELETE` → toast with Undo → Undo calls `POST /notes/:id/restore` and re-inserts. On `DELETE` failure, re-insert and show an error toast.
5. Rewrite `unsavedChangesGuard` with `ConfirmService` (Module 10 note 4 shows the final version).
6. Credential detail: `RevealedSecret` component — calls `reveal`, shows the value, counts down 30 s with an interval cleaned up in `DestroyRef`, then clears the secret signal.
7. Add the mock routes: `POST /credentials`, `POST /credentials/:id/reveal`, `POST /notes/:id/restore`.
8. Keyboard-only run-through: Tab to every control, open and close each overlay, and check where focus lands.

## 7. Complete working code

`NotesStore.remove` with Undo, the menu on `NoteCard`, the Add Credential modal, `RevealedSecret`, and the mock routes are in the [solution](solutions/lab-11-solution.md).

## 8. Important lines explained

- `if (!(await this.confirm.ask({...}))) return;` — dialogs as `async` functions keep the calling code linear.
- `this.notes.update(list => list.filter(n => n.id !== id))` **before** the request — optimistic UI; the snapshot `prev` allows rollback.
- `toast.success('Moved to Trash', { action: { label: 'Undo', run: () => this.restore(note, index) } })` — Undo is just a callback.
- `el.showModal()` — the browser makes the background inert and traps focus; you don't hand-roll a focus trap.
- `secret.set(null)` on timeout **and** on destroy — the secret doesn't linger in memory or the DOM.

## 9. Expected output

Delete flow with a screen reader: "Delete note? dialog", "Delete button"… then "Moved to Trash, Undo button" announced politely.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Modal opens but page behind still clickable | used `show()` instead of `showModal()` |
| Focus lost after closing | removed the `<dialog>` from the DOM before `close()`; keep it rendered and toggle `open` |
| Two confirm dialogs stacked | `ConfirmService.ask` must resolve the previous one (`false`) before opening another |
| Undo restores at the end of the list | remember the original index |
| Secret still visible after navigating away | clear it in `DestroyRef.onDestroy` |
| Toast says "Copied: hunter2" | never put secrets in toasts or logs |
| Menu closes when clicking an item before the handler runs | `ClickOutside` listener on `pointerdown` of the document — ensure the item is inside the host |

## 11. Test cases

```typescript
describe('ConfirmService', () => {
  it('resolves true/false', async () => {
    const svc = TestBed.inject(ConfirmService);
    const p = svc.ask({ title: 'Delete?' });
    svc.answer(true);
    await expect(p).resolves.toBe(true);
  });
  it('cancels the previous dialog when a new one opens', async () => {
    const svc = TestBed.inject(ConfirmService);
    const first = svc.ask({ title: 'A' });
    svc.ask({ title: 'B' });
    await expect(first).resolves.toBe(false);
  });
});

describe('NotesStore.remove', () => {
  it('rolls back when DELETE fails', async () => {
    // provide ConfirmService stub → ask resolves true; NotesApi stub → remove throws
    const store = TestBed.inject(NotesStore);
    const before = store.notes().length;
    await store.remove(store.notes()[0]!.id);
    expect(store.notes().length).toBe(before);
  });
});
```

## 12. Independent practice task

Build the **re-auth before reveal** flow from [Module 13 note 2 §10](../13-http-and-api-integration/02-interceptors-and-auth.md#10-advanced-example--re-auth-before-revealing-a-credential):

- The mock returns `403 { message: 'REAUTH_REQUIRED' }` for `/reveal` unless re-auth happened within the last 5 minutes.
- A `ReauthDialog` service (same pattern as `ConfirmService`) shows a modal with a password field and returns `Promise<boolean>`.
- `reauthInterceptor` catches that 403, awaits the dialog, and retries the request once.

## 13. Solution

See [solutions/lab-11-solution.md](solutions/lab-11-solution.md).

## 14. Completion checklist

- [ ] Modal, confirm, toast hosts in the shell; all keyboard-accessible.
- [ ] Delete with confirm + Undo + rollback on failure.
- [ ] Unsaved-changes guard uses `ConfirmService`.
- [ ] Reveal → copy → auto-hide, no secret in toasts/logs.
- [ ] Re-auth dialog + interceptor.
- [ ] Tests pass; tagged `lab-11`.
