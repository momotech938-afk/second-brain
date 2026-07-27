# Debugging Exercises

Back to [Module 22](README.md) · Previous: [Practical assessments](02-practical-assessments.md) · Next: [Implementation plan](04-implementation-plan.md)

For each exercise: read the symptom and code, **write your diagnosis and fix first**, then open the answer. Score one point per correct diagnosis. Target: **12 / 15**.

---

### D1 — The list that never updates

```typescript
toggleFavorite(id: string) {
  const note = this.notes().find(n => n.id === id)!;
  note.isFavorite = !note.isFavorite;
  this.notes.set(this.notes());
}
```

Symptom: the star doesn't change until something else re-renders.

<details><summary>Answer</summary>

The object was mutated and the same array reference set again. Signals compare with `Object.is`, so nothing is notified, and OnPush children receive the same input reference. Fix: `this.notes.update(list => list.map(n => n.id === id ? { ...n, isFavorite: !n.isFavorite } : n))`. ([Module 08 note 1](../08-signals-and-reactivity/01-signal-basics.md))
</details>

---

### D2 — Reload logs me out

```typescript
export const authGuard: CanActivateFn = () =>
  inject(AuthStore).isLoggedIn() || inject(Router).createUrlTree(['/login']);
```

Symptom: signed in, press F5 on `/notes` → login page.

<details><summary>Answer</summary>

On reload, the in-memory token is gone and `status` is `'unknown'` until `/auth/refresh` completes. The guard checks too early. Fix: make it `async` and `await auth.ensureSession()` first, and/or restore the session in `provideAppInitializer`. ([Module 10 note 4](../10-routing/04-guards-and-resolvers.md#waiting-for-session-restore))
</details>

---

### D3 — Three refresh calls, then logged out

Symptom: after 15 minutes idle, opening the dashboard shows three `POST /auth/refresh`. The last two fail with 401 and the user is logged out.

<details><summary>Answer</summary>

Each 401 started its own refresh. With refresh-token rotation, the 2nd and 3rd used an already-rotated token. Fix: single-flight refresh — share one in-flight observable (`refresh$ ??= …shareReplay(1)` + `finalize` to reset). ([Module 13 note 2](../13-http-and-api-integration/02-interceptors-and-auth.md#why-the-refresh-is-single-flight))
</details>

---

### D4 — Search fires on every keystroke and shows wrong results

```typescript
onInput(q: string) {
  this.api.search(q).subscribe(r => this.results.set(r));
}
```

Symptom: typing "angular" fires 7 requests; sometimes results for "angul" appear last.

<details><summary>Answer</summary>

There's no debounce and no cancellation, so responses can arrive out of order. Fix: drive a signal, then `toObservable(q).pipe(debounceTime(300), distinctUntilChanged(), switchMap(q => api.search(q)))`, or use `httpResource`/`rxResource` keyed on a debounced signal (they cancel stale requests). ([Module 14 note 2](../14-rxjs/02-operators.md))
</details>

---

### D5 — `NG0203` in a store method

```typescript
@Service()
export class NotesStore {
  loadLater() {
    setTimeout(() => {
      const http = inject(HttpClient);
      http.get('/api/v1/notes').subscribe();
    });
  }
}
```

<details><summary>Answer</summary>

`inject()` only works in an injection context (constructor, field initialisers, factory functions, `runInInjectionContext`). Inside a `setTimeout` callback there is none. Fix: `private http = inject(HttpClient);` as a field. ([Module 07 note 1](../07-services-and-dependency-injection/01-services-and-inject.md))
</details>

---

### D6 — Effect loop

```typescript
count = signal(0);
constructor() {
  effect(() => this.count.set(this.count() + 1));
}
```

Symptom: the browser freezes, or Angular reports an effect loop.

<details><summary>Answer</summary>

The effect reads and writes the same signal, so it re-triggers itself. Derived values belong in `computed`. If you genuinely need to read without tracking, use `untracked()`. ([Module 08 note 2](../08-signals-and-reactivity/02-computed-and-effect.md))
</details>

---

### D7 — Blank page after deploy

Console: `Failed to load module script: Expected a JavaScript module script but the server responded with a MIME type of "text/html".`

<details><summary>Answer</summary>

The server returns `index.html` for JS URLs. Causes: deploying `dist/myspce` instead of `dist/myspce/browser`, a wrong `<base href>`, or a fallback rule that catches assets. Fix the deployed folder/base href and use `try_files $uri $uri/ /index.html` only for non-asset routes. ([Module 20 note 4](../20-security-and-deployment/04-deployment-and-ci.md#7-production-troubleshooting))
</details>

---

### D8 — Form never valid

```typescript
protected model = signal({ email: '', password: '' });
protected loginForm = form(this.model, p => {
  required(p.email);
  email(p.email);
  minLength(p.password, 8);
});
```

```html
<input [formField]="loginForm.email" />
<input type="password" [formField]="loginForm.email" />
```

<details><summary>Answer</summary>

The password input is bound to `loginForm.email`, so the password field is never touched or filled, and `minLength` on an empty password fails. Fix: `[formField]="loginForm.password"`. Signal Forms errors are per field, so log `loginForm.password().errors()` to see why it's invalid. ([Module 12 note 4](../12-reactive-forms/04-signal-forms.md))
</details>

---

### D9 — The admin chunk downloads for everyone

```typescript
{ path: 'admin', canActivate: [roleGuard(['admin'])], loadChildren: () => import('./features/admin/admin.routes') }
```

<details><summary>Answer</summary>

`canActivate` doesn't stop the lazy `loadChildren` from being fetched. The router loads the children to finish matching, and only then runs activation guards. Use `canMatch: [roleGuard([...])]` (a `CanMatchFn` factory): for other roles the route doesn't match at all, so the chunk is never requested and the URL falls through to the 404 page. ([Lab 9](../21-myspce-practice-labs/lab-09-protected-routes.md))
</details>

---

### D10 — Memory leak on the Tasks page

```typescript
ngOnInit() {
  interval(60_000).subscribe(() => this.store.reload());
}
```

Symptom: after visiting Tasks 5 times, reload runs 5 times a minute.

<details><summary>Answer</summary>

The subscription is never torn down. Fix: `interval(60_000).pipe(takeUntilDestroyed(this.destroyRef))`, or create it in a field initialiser with `takeUntilDestroyed()`. ([Module 09 note 3](../09-lifecycle-and-change-detection/03-cleanup-and-memory-leaks.md))
</details>

---

### D11 — `@defer` didn't split anything

```typescript
@Component({
  imports: [WeekProgressWidget],
  template: `
    @defer (on viewport) { <app-week-progress-widget /> } @placeholder { <div class="h-56"></div> }
  `,
})
export class DashboardPage {
  private w = viewChild(WeekProgressWidget);
}
```

<details><summary>Answer</summary>

`viewChild(WeekProgressWidget)` references the class outside the defer block, so the compiler must keep it in the eager bundle. Remove the eager reference (query it inside the deferred content, or communicate via a service). ([Module 17 note 2](../17-performance/02-loading-and-bundles.md))
</details>

---

### D12 — Error toast twice, inline error missing

Symptom: saving a note with a 400 error shows the same toast twice and no field error.

<details><summary>Answer</summary>

Both the global `errorInterceptor` and the component show a toast, and nobody maps the 400 to the form. Fix: the interceptor toasts only 0/5xx/429. Pass `SKIP_ERROR_TOAST` for requests whose errors the feature handles, and return the server message from the `submit` action as a field or form error. ([Module 13 note 3](../13-http-and-api-integration/03-error-handling-retry.md))
</details>

---

### D13 — Test passes alone, fails in the suite

```typescript
import { MOCK_NOTES } from '../../testing/mock-data';
it('removes a note', () => {
  MOCK_NOTES.splice(0, 1);
  /* … */
});
```

<details><summary>Answer</summary>

The test mutates a shared fixture module, so later tests see a shorter array. Clone per test: `const notes = structuredClone(MOCK_NOTES);`. Never mutate shared fixtures. ([Module 19 note 1](../19-testing/01-testing-setup.md#7-isolation-and-maintainability))
</details>

---

### D14 — Template doesn't update in a component test

```typescript
fixture.componentRef.setInput('note', updated);
expect(fixture.nativeElement.textContent).toContain(updated.title);
```

<details><summary>Answer</summary>

In zoneless tests, rendering is scheduled, not synchronous. `await fixture.whenStable()` (or call `fixture.detectChanges()`) before asserting. ([Module 19 note 1](../19-testing/01-testing-setup.md#zoneless-tests-the-default-for-angular-22-apps))
</details>

---

### D15 — Optional chaining changed behaviour after upgrading to v22

```html
@if (user()?.name === null) { <p>No name set</p> }
```

Symptom: after the v22 upgrade, "No name set" no longer appears when `user()` is `null`.

<details><summary>Answer</summary>

In v22, template `?.` returns `undefined` (like TypeScript), not `null`. The comparison `=== null` is now false. Fix the check (`!user()?.name` or `== null`). The migration may have wrapped old expressions with `$safeNavigationMigration(...)` to preserve the old behaviour; replace those deliberately. ([Module 03 note 2](../03-template-control-flow/02-expressions-and-loading-states.md))
</details>

---

## Score

| Score | Next step |
|---|---|
| 13–15 | Strong debugging instincts |
| 10–12 | Re-read the notes linked from the ones you missed |
| < 10 | Redo the related labs before starting the real project |
