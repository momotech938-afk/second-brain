# Lab 7 — Loading, Empty, Error and Success States

Back to [Labs](README.md) · Solution: [lab-07-solution.md](solutions/lab-07-solution.md) · Previous: [Lab 6](lab-06-typed-api-services.md) · Next: [Lab 8](lab-08-authentication.md)

## 1. Learning objective

Give every data-driven screen the four states users actually see — **loading**, **empty**, **error (with retry)**, **content** — using resource status signals, skeletons, a reusable `ErrorState`, a global `errorInterceptor` + `toMessage()`, and the `ToastService`.

## 2. Prerequisites

[Module 03 note 2](../03-template-control-flow/02-expressions-and-loading-states.md), [Module 08 note 4](../08-signals-and-reactivity/04-resources.md), [Module 13 note 3](../13-http-and-api-integration/03-error-handling-retry.md), [Lab 6](lab-06-typed-api-services.md).

## 3. Problem statement

With a real (mock) network, screens flash blank, errors appear only in the console, and a failed save silently loses work. Fix it once with reusable pieces, then apply to Notes and the Dashboard.

## 4. Expected behaviour

| Situation | UI |
|---|---|
| First load | skeleton cards (same grid shape as real cards) |
| Reload with data already shown | data stays, a thin progress bar shows at top |
| 0 results, no filters | "No notes yet" + "New note" button |
| 0 results with search | "No notes match …" + "Clear search" |
| Request fails | `ErrorState` with message + "Try again" button (calls `reload()`) |
| Mutation fails (save, delete) | error toast; the form keeps its values |
| Mutation succeeds | success toast "Note saved" |
| Dashboard | each widget loads and fails **independently** |

## 5. Relevant concepts

`resource.status()`, `isLoading()`, `hasValue()`, `error()`, `reload()`, `@if/@else if`, `@defer` placeholders, `HttpContextToken`, functional interceptors, `aria-busy`, `role="status"`/`role="alert"`.

## 6. Step-by-step implementation

1. Create shared UI:
   - `Skeleton` (`width`, `height`, `rounded` inputs; `animate-pulse`)
   - `NoteCardSkeleton` (uses `Skeleton` to mimic a card)
   - `ErrorState` (`message` input, `retry` output, `role="alert"`)
   - `EmptyState` already exists; add a projected `[action]` slot
2. Add `core/http/error-messages.ts` (`toMessage`) from Module 13 note 3: friendly text per status; server `message` for 400/409.
3. Add `SKIP_ERROR_TOAST` (`core/http/http-context.ts`) and `errorInterceptor`: for errors **not** opted out, toast only status 0, 5xx and 429. Leave 401 to Lab 8 and 400/403/404/409 to the feature.
4. Build `ToastService` + `ToastHost` (or reuse the [Module 16 note 4](../16-advanced-components/04-reusable-ui-kit.md) version). Put `<app-toast-host />` in `App`.
5. Notes page: switch on resource state in this order — **error → first load → empty → content** — and show a top progress bar when `isLoading() && hasValue()`.
6. Note editor save: use `ToastService.success('Note saved')`; on error, don't navigate and keep the form; pass `SKIP_ERROR_TOAST` so the global toast doesn't duplicate the inline message.
7. Dashboard: wrap each widget in a small `WidgetFrame` that renders its own states.
8. Test by adding `?fail=1` (Lab 6 mock) and by throttling to "Slow 4G" in DevTools.

## 7. Complete working code

`ErrorState`, `Skeleton`, `toMessage`, `errorInterceptor`, the Notes page template and `WidgetFrame` are in the [solution](solutions/lab-07-solution.md).

## 8. Important lines explained

- `@if (list.error()) { … } @else if (!list.hasValue()) { skeleton } @else if (!notes().length) { empty } @else { grid }` — order matters: an error during reload must not be hidden behind old data forever.
- `[attr.aria-busy]="list.isLoading()"` — screen readers know the region is updating.
- `context: new HttpContext().set(SKIP_ERROR_TOAST, true)` — opt out per request instead of special-casing URLs in the interceptor.
- `catchError(err => { …; return throwError(() => err); })` — the interceptor *reports*, it doesn't swallow; callers still receive the error.

## 9. Expected output

- Slow network: grid of 8 grey skeleton cards, then real cards with no layout jump.
- `?fail=1`: "We couldn't load your notes. Try again." with a working button; one toast "Something went wrong on our side. Try again.".
- Saving a note titled "boom": inline error, toast not duplicated, form keeps the text.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Empty state flashes before data | checked `!notes().length` before `hasValue()` |
| Two toasts for one error | both interceptor and component toast; use `SKIP_ERROR_TOAST` |
| Retry button does nothing | called `reload` on a different resource instance, or the resource params are `undefined` (idle) |
| Skeleton causes layout shift | skeleton dimensions differ from real card; reuse the same grid classes |
| Error messages show raw JSON | always go through `toMessage(err)` |

## 11. Test cases

```typescript
it('shows ErrorState and retries', async () => {
  const fixture = TestBed.createComponent(NotesPage);
  const ctrl = TestBed.inject(HttpTestingController);
  await fixture.whenStable();
  ctrl.expectOne(r => r.url.endsWith('/notes')).flush({ message: 'x' }, { status: 500, statusText: 'Server Error' });
  await fixture.whenStable();
  expect(fixture.nativeElement.querySelector('[role=alert]')?.textContent).toContain("couldn't load");

  fixture.nativeElement.querySelector('[data-test=retry]').click();
  ctrl.expectOne(r => r.url.endsWith('/notes')).flush({ data: [], meta: { total: 0, page: 1, limit: 8 } });
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).toContain('No notes yet');
});

it('toMessage maps network errors', () => {
  expect(toMessage(new HttpErrorResponse({ status: 0 }))).toMatch(/offline|reach the server/i);
});
```

(Full TestBed setup with `provideTestDefaults()` is in [Module 19](../19-testing/02-component-tests.md).)

## 12. Independent practice task

Apply the four states to **Credentials** list and the **Tasks** "Today" list. Add an **offline banner**: a `NetworkStatus` service exposing `online = signal(navigator.onLine)` updated from `online`/`offline` window events; show a sticky banner "You're offline — changes won't be saved" and disable save buttons while offline.

## 13. Solution

See [solutions/lab-07-solution.md](solutions/lab-07-solution.md).

## 14. Completion checklist

- [ ] Notes and Dashboard show all four states correctly.
- [ ] One toast per failure; forms keep values on failure.
- [ ] `aria-busy`, `role=status`, `role=alert` in place.
- [ ] Credentials/Tasks states + offline banner.
- [ ] Tests pass; tagged `lab-07`.
