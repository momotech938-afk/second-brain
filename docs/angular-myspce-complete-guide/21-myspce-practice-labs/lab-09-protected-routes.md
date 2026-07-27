# Lab 9 — Protected Routes, Roles and the Admin Area

Back to [Labs](README.md) · Solution: [lab-09-solution.md](solutions/lab-09-solution.md) · Previous: [Lab 8](lab-08-authentication.md) · Next: [Lab 10](lab-10-search-filter-sort-paginate.md)

## 1. Learning objective

Protect the app with guards that wait for session restore, keep signed-in users away from `/login`, gate the admin panel by role with `canMatch`, hide admin UI with `*appHasRole`, hide disabled modules with feature flags, and clear user data on logout.

## 2. Prerequisites

[Module 10 note 4](../10-routing/04-guards-and-resolvers.md), [Module 05 note 3](../05-directives/03-structural-and-host-directives.md), [Lab 8](lab-08-authentication.md).

## 3. Problem statement

Confirmed: the app is private and has an admin panel that controls the site. Assumed (A3): roles `super_admin | admin | user`. Guards are **UX only**. The API must still reject unauthorised calls (see [Module 20 note 2](../20-security-and-deployment/02-auth-and-tokens.md#4-guards-vs-backend-authorisation)).

## 4. Expected behaviour

| Situation | Result |
|---|---|
| Signed out, open `/notes/123` | → `/login?returnUrl=%2Fnotes%2F123`; after login → back to `/notes/123` |
| Signed in, open `/login` | → `/dashboard` |
| Reload on `/settings/security` while signed in | stays there (guard waits for `ensureSession()`) |
| Role `user` opens `/admin` | route doesn't match → 404 page (no hint the route exists) |
| Role `admin`/`super_admin` | sees "Admin" in the sidebar and can open `/admin` |
| Admin turns off the "Tasks" module flag | `/tasks` → 404; sidebar hides "Tasks" |
| Sign out, sign in again | no previous notes/tasks flash on screen |

## 5. Relevant concepts

`CanActivateFn`, `CanMatchFn`, `UrlTree`, `canActivateChild`, guard factories, structural directives, `effect` for reset on logout, `BootstrapStore` flags.

## 6. Step-by-step implementation

1. Write `authGuard` (waits for `ensureSession()`, returns a `UrlTree` with `returnUrl`) and `guestGuard` (the reverse).
2. Apply `canActivate: [authGuard]` and `canActivateChild: [authGuard]` on the shell route; `canActivate: [guestGuard]` on `/login`.
3. Write `roleGuard(roles)` as a **`canMatch`** factory. For `user`, return `false` so the router falls through to `**` (404). Don't redirect: that would reveal the route exists.
4. Create `features/admin/admin.routes.ts` with a placeholder `AdminLayout` (tabs: Appearance, Modules, Navigation — matching the confirmed "controlled by the admin panel" idea; their content is not specified yet).
5. Add `HasRole` (`*appHasRole`) from Module 05 to the sidebar's Admin item.
6. Add `BootstrapStore` (loads `GET /bootstrap` after login) and `featureGuard('tasks')`; extend the mock with `/bootstrap` returning flags.
7. Reset stores on logout: in each root store, `effect(() => { if (auth.status() === 'anonymous') this.reset(); })`.
8. Try every row of the table above.

## 7. Complete working code

Guards, route table changes, `BootstrapStore`, mock `/bootstrap`, sidebar and the logout reset are in the [solution](solutions/lab-09-solution.md).

## 8. Important lines explained

- `await auth.ensureSession()` inside the guard — without it, a reload sees `status === 'unknown'` and wrongly bounces to login.
- `router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } })` — return a `UrlTree` instead of calling `navigate()` inside a guard, so the router handles the redirect atomically.
- `canMatch: [roleGuard(['admin', 'super_admin'])]` — the lazy admin chunk isn't even downloaded for other roles.
- `return false` from `canMatch` → router tries the next route → `**`.
- `*appHasRole` hides UI, but **the API is the real lock**.

## 9. Expected output

Network tab for a `user`-role session: no `admin-*.js` chunk is ever requested, even after typing `/admin`.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Reload always goes to login | guard didn't `await ensureSession()` |
| Infinite redirect loop | `authGuard` on `/login`, or `guestGuard` on a shell child |
| Admin chunk still downloads for users | used `canActivate` instead of `canMatch` |
| Flags not applied after login | `BootstrapStore.load()` only ran at startup; call it after `login()` too |
| Old user's data visible after re-login | root stores not reset on `'anonymous'` |
| `returnUrl` contains `/login` | `authGuard` placed on the login route |

Enable `withDebugTracing()` temporarily to see guard results in the console.

## 11. Test cases

```typescript
describe('authGuard', () => {
  const setup = (status: 'authenticated' | 'anonymous') => {
    TestBed.configureTestingModule({
      providers: [
        provideRouter([
          { path: 'login', component: Dummy },
          { path: 'notes/:id', component: Dummy, canActivate: [authGuard] },
        ]),
        { provide: AuthStore, useValue: { ensureSession: async () => {}, isLoggedIn: () => status === 'authenticated' } },
      ],
    });
  };

  it('redirects anonymous users with returnUrl', async () => {
    setup('anonymous');
    await RouterTestingHarness.create('/notes/123');
    expect(TestBed.inject(Router).url).toBe('/login?returnUrl=%2Fnotes%2F123');
  });

  it('lets signed-in users through', async () => {
    setup('authenticated');
    await RouterTestingHarness.create('/notes/123');
    expect(TestBed.inject(Router).url).toBe('/notes/123');
  });
});

describe('roleGuard', () => {
  it('does not match for role user', async () => {
    const result = await TestBed.runInInjectionContext(() =>
      roleGuard(['admin'])({} as Route, [], {} as never));
    expect(result).toBe(false);
  });
});
```

(`Dummy` is an empty standalone component; `roleGuard` test needs an `AuthStore` stub with `user: () => ({ role: 'user' })`.)

## 12. Independent practice task

Add a **lock screen**. This is a *proposed* feature, not a confirmed requirement, but a credentials vault benefits from it: after 10 minutes without input, navigate to `/lock`, which shows the user's name and a password field. Correct password (`POST /auth/reauth` in the mock) returns to the previous URL. While locked, all shell routes redirect to `/lock` (`lockGuard`). Use an `IdleLock` service with `fromEvent` on `pointerdown`/`keydown`, `debounceTime(10 * 60_000)`, and `takeUntilDestroyed`.

## 13. Solution

See [solutions/lab-09-solution.md](solutions/lab-09-solution.md).

## 14. Completion checklist

- [ ] Every row in the behaviour table works.
- [ ] Admin chunk not downloaded for `user`.
- [ ] Feature flags hide routes and nav items.
- [ ] Stores reset on logout.
- [ ] Idle lock screen.
- [ ] Tests pass; tagged `lab-09`.
