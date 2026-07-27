# Lab 8 — Authentication: Login, Session Restore, Refresh, Logout

Back to [Labs](README.md) · Solution: [lab-08-solution.md](solutions/lab-08-solution.md) · Previous: [Lab 7](lab-07-ui-states.md) · Next: [Lab 9](lab-09-protected-routes.md)

## 1. Learning objective

Wire the confirmed login methods (email + password, Google) to an `AuthStore` with an in-memory access token, a cookie-based refresh, single-flight refresh on 401, session restore at startup, and logout — all against the mock backend.

## 2. Prerequisites

[Module 13 note 2](../13-http-and-api-integration/02-interceptors-and-auth.md), [Module 20 note 2](../20-security-and-deployment/02-auth-and-tokens.md), [Lab 3](lab-03-forms-and-controls.md) (login form), [Lab 6](lab-06-typed-api-services.md), [Lab 7](lab-07-ui-states.md).

## 3. Problem statement

Confirmed: single user, Google + email/password login. Assumed (A2): access token in memory (15 min), refresh token in an httpOnly cookie. The backend isn't built, so the mock must simulate tokens expiring so you can *see* refresh work.

## 4. Expected behaviour

- `/login` with `me@myspce.dev` / `Password123!` → dashboard. Wrong password → inline "Email or password is incorrect."
- Page reload while signed in → still signed in (session restored via `/auth/refresh` before first render).
- Access token "expires" after 30 s in the mock → next API call gets 401 → **one** refresh → original request retried transparently.
- Three parallel calls with an expired token → exactly **one** `/auth/refresh`.
- Refresh fails → redirected to `/login?reason=expired&returnUrl=…` and a notice is shown.
- Logout → `/auth/logout`, memory cleared, `/login`.
- Nothing is written to `localStorage`/`sessionStorage`.

## 5. Relevant concepts

`AuthStore` (signals), `provideAppInitializer`, `HttpContextToken` (`SKIP_AUTH`), `authInterceptor`, `shareReplay` single-flight, `firstValueFrom`, `returnUrl`, `afterNextRender` for the OAuth callback.

## 6. Step-by-step implementation

1. Add `AuthResponse { accessToken: string; user: User }` to models.
2. Copy `AuthStore`, `SKIP_AUTH` and `authInterceptor` from [Module 13 note 2](../13-http-and-api-integration/02-interceptors-and-auth.md). Don't change the public API: `user`, `status`, `isLoggedIn`, `accessToken`, `login()`, `ensureSession()`, `refresh()`, `logout()`, `expire()`.
3. Extend the mock backend with `/auth/login`, `/auth/refresh`, `/auth/logout`, and a token check on every other `/api/v1` route:
   - the mock "cookie" is a module variable `sessionAlive` (a browser can't fake httpOnly cookies from JS; this is a simulation)
   - tokens are `mock.<expiresAtMs>`; expired or missing → 401
4. Register interceptors in order: `errorInterceptor`, `authInterceptor`, then `mockBackendInterceptor` innermost.
5. Add `provideAppInitializer(() => inject(AuthStore).ensureSession())`.
6. Update `LoginPage.onSubmit` to call `auth.login()`; map 401 to a form-level error; navigate to `returnUrl` (only if it starts with `/`) or `/dashboard`.
7. Show a notice on `/login` when `reason=expired`.
8. Add a user menu in the top bar: name, avatar initials, "Sign out" → `auth.logout()`.
9. In DevTools → Network/Console, confirm one refresh for three parallel calls (open the Dashboard after waiting 30 s).

## 7. Complete working code

Mock auth routes, providers, the updated `LoginPage`, the expired notice and the user menu are in the [solution](solutions/lab-08-solution.md).

## 8. Important lines explained

- `this.refresh$ ??= http.post(...).pipe(..., finalize(() => this.refresh$ = null), shareReplay(...))` — concurrent 401s share one refresh; the next expiry starts a new one.
- `context: new HttpContext().set(SKIP_AUTH, true)` on login/refresh — prevents the interceptor from trying to refresh the refresh call (infinite loop).
- `req.url.startsWith(api)` in the interceptor — the token never leaks to third-party URLs (S3 upload URLs, Google).
- `returnUrl?.startsWith('/') && !returnUrl.startsWith('//')` — blocks open redirects such as `?returnUrl=https://evil.example`.
- `provideAppInitializer` — the first render waits for the session check, so guards and the shell don't flicker between "logged out" and "logged in".

## 9. Expected output

Console with the mock's debug log enabled, after 30 s idle on the Dashboard:

```text
[mock] GET /api/v1/dashboard/summary → 401
[mock] GET /api/v1/dashboard/today → 401
[mock] GET /api/v1/dashboard/week-progress → 401
[mock] POST /api/v1/auth/refresh → 200
[mock] GET /api/v1/dashboard/summary → 200
[mock] GET /api/v1/dashboard/today → 200
[mock] GET /api/v1/dashboard/week-progress → 200
```

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Infinite refresh loop | login/refresh requests missing `SKIP_AUTH` |
| Three refresh calls | `refresh()` creates a new observable each time; use the `??=` + `shareReplay` pattern |
| Signed out on every reload | no `ensureSession()` initializer, or the mock forgets `sessionAlive` (it resets on full reload — that's expected for the mock; use a `sessionStorage` flag **only inside the mock** to simulate the cookie) |
| Token visible in `localStorage` | never store it; the store keeps it in a signal |
| `returnUrl` to another site | validate it starts with a single `/` |
| Logout leaves old data on screen | root stores keep state; reset them when `status` becomes `'anonymous'` (Lab 9) |

## 11. Test cases

```typescript
describe('authInterceptor', () => {
  it('refreshes once for concurrent 401s and retries', () => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withInterceptors([authInterceptor])), provideHttpClientTesting(),
        provideRouter([]), { provide: API_URL, useValue: '/api/v1' },
      ],
    });
    const http = TestBed.inject(HttpClient);
    const ctrl = TestBed.inject(HttpTestingController);

    http.get('/api/v1/a').subscribe();
    http.get('/api/v1/b').subscribe();
    ctrl.match(r => ['/api/v1/a', '/api/v1/b'].includes(r.url)).forEach(r => r.flush(null, { status: 401, statusText: 'Unauthorized' }));

    const refresh = ctrl.match('/api/v1/auth/refresh');
    expect(refresh.length).toBe(1);
    refresh[0]!.flush({ accessToken: 'new', user: { id: '1', name: 'Me', email: 'me@myspce.dev', avatarUrl: null, bio: null, role: 'super_admin' } });

    const retried = ctrl.match(r => r.url === '/api/v1/a' || r.url === '/api/v1/b');
    expect(retried.every(r => r.request.headers.get('Authorization') === 'Bearer new')).toBe(true);
  });

  it('never attaches the token to non-API URLs', () => {
    /* same setup */
    TestBed.inject(HttpClient).get('https://s3.example/upload').subscribe();
    expect(TestBed.inject(HttpTestingController).expectOne('https://s3.example/upload').request.headers.has('Authorization')).toBe(false);
  });
});
```

## 12. Independent practice task

Implement the **Google login** path end-to-end against the mock:

1. "Continue with Google" links to `/api/v1/auth/google`. In mock mode that real navigation would 404, so in dev point it to `/auth/callback?mock=1` instead (via an environment flag).
2. Build `OAuthCallbackPage` at `/auth/callback` (outside the shell). It reads `error` and `mock` query params via input binding, sets the mock "cookie" when `mock=1`, calls `auth.ensureSession()`, and navigates to `/dashboard` (or back to `/login?reason=…` on error).
3. Show a full-screen "Signing you in…" with `role="status"` while it works.

## 13. Solution

See [solutions/lab-08-solution.md](solutions/lab-08-solution.md).

## 14. Completion checklist

- [ ] Email/password login, wrong-password message, logout.
- [ ] Session survives reload (mock-simulated).
- [ ] One refresh for concurrent 401s; retry works; expiry redirect with notice.
- [ ] No tokens in Web Storage; `returnUrl` validated.
- [ ] Google callback page.
- [ ] Tests pass; tagged `lab-08`.
