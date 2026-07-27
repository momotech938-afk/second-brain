# Lab 8 — Solution

Back to [Lab 8](../lab-08-authentication.md) · [Labs index](../README.md)

`AuthStore`, `SKIP_AUTH` and `authInterceptor` are used **exactly** as in [Module 13 note 2](../../13-http-and-api-integration/02-interceptors-and-auth.md). This solution adds the mock, the wiring and the UI.

## Mock auth routes

```typescript
// core/http/mock-auth.ts  (dev only — a simulation; real httpOnly cookies are set by the server)
import { HttpErrorResponse, HttpRequest, HttpResponse } from '@angular/common/http';
import { Observable, delay, dematerialize, materialize, of, throwError } from 'rxjs';
import type { User } from '../models';

const TOKEN_TTL_MS = 30_000;                        // short on purpose so you can watch refresh happen
const SESSION_FLAG = 'mock-refresh-cookie';         // simulates the httpOnly cookie surviving reloads (mock only!)
const USER: User = { id: '01JMOCKUSER0000000000000000', name: 'Pranjal', email: 'me@myspce.dev', avatarUrl: null, bio: null, role: 'super_admin' };

const issue = () => ({ accessToken: `mock.${Date.now() + TOKEN_TTL_MS}`, user: USER });
const ok = <T>(body: T) => of(new HttpResponse({ status: 200, body })).pipe(delay(250));
const err = (status: number, message: string, url: string): Observable<never> =>
  throwError(() => new HttpErrorResponse({ status, url, error: { statusCode: status, message, path: url, requestId: crypto.randomUUID(), timestamp: new Date().toISOString() } }))
    .pipe(materialize(), delay(250), dematerialize());

export const mockSession = {
  get alive() { return sessionStorage.getItem(SESSION_FLAG) === '1'; },
  set alive(v: boolean) { v ? sessionStorage.setItem(SESSION_FLAG, '1') : sessionStorage.removeItem(SESSION_FLAG); },
};

/** Returns a response for /auth routes, an error for unauthenticated calls, or null to continue. */
export function mockAuth(req: HttpRequest<unknown>, path: string): Observable<HttpResponse<unknown>> | null {
  if (path === '/auth/login' && req.method === 'POST') {
    const { email, password } = req.body as { email: string; password: string };
    if (email === USER.email && password === 'Password123!') { mockSession.alive = true; return ok(issue()); }
    return err(401, 'Invalid email or password', req.url);
  }
  if (path === '/auth/refresh' && req.method === 'POST') {
    return mockSession.alive ? ok(issue()) : err(401, 'No session', req.url);
  }
  if (path === '/auth/logout' && req.method === 'POST') { mockSession.alive = false; return ok(null); }

  // every other API route requires a valid, unexpired token
  const auth = req.headers.get('Authorization') ?? '';
  const expires = Number(auth.replace('Bearer mock.', ''));
  if (!auth.startsWith('Bearer mock.') || Number.isNaN(expires) || expires < Date.now()) {
    return err(401, 'Token expired', req.url);
  }
  return null;
}
```

In `mockBackendInterceptor` (Lab 6), call it first:

```typescript
export const mockBackendInterceptor: HttpInterceptorFn = (req, next) => {
  const api = inject(API_URL);
  if (!req.url.startsWith(api)) return next(req);
  const path = req.url.slice(api.length);
  if (isDevMode()) console.debug('[mock]', req.method, req.urlWithParams);
  return mockAuth(req, path) ?? route(req, path) ?? fail(404, `No mock for ${req.method} ${path}`, req.url);
};
```

> The `sessionStorage` flag exists **only inside the mock** to imitate a cookie. The real app never reads or writes tokens in Web Storage.

## Providers

```typescript
// app.config.ts (excerpt)
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([
      errorInterceptor,                                         // outermost: sees final errors
      authInterceptor,                                          // adds token, refreshes on 401
      ...(environment.useMockApi ? [mockBackendInterceptor] : []),   // innermost: the "server"
    ])),
    { provide: API_URL, useValue: environment.apiUrl },
    provideAppInitializer(() => inject(AuthStore).ensureSession()),
  ],
};
```

Why this order: a 401 from the mock passes through `authInterceptor` first (refresh + retry). Only if that fails does `errorInterceptor` see the error. 401 is not in its toast list, so `expire()` handles the UX.

## `LoginPage` (updated submit)

```typescript
// features/auth/login-page.ts (changes from Lab 3)
export default class LoginPage {
  private auth = inject(AuthStore);
  private router = inject(Router);
  returnUrl = input<string>();          // ?returnUrl=…
  reason = input<string>();             // ?reason=expired

  protected model = signal({ email: '', password: '' });
  protected loginForm = form(this.model, p => { /* same rules as Lab 3 */ });

  async onSubmit(e: Event) {
    e.preventDefault();
    await submit(this.loginForm, async f => {
      try {
        await this.auth.login(f().value());
      } catch (err) {
        if (err instanceof HttpErrorResponse && err.status === 401) {
          return { kind: 'invalidCredentials', message: 'Email or password is incorrect.' };
        }
        return { kind: 'server', message: toMessage(err) };
      }
      await this.router.navigateByUrl(safeReturnUrl(this.returnUrl()), { replaceUrl: true });
      return undefined;
    });
  }
}

function safeReturnUrl(url: string | undefined): string {
  return url && url.startsWith('/') && !url.startsWith('//') && !url.startsWith('/login') ? url : '/dashboard';
}
```

Template addition (above the form fields):

```html
@if (reason() === 'expired') {
  <p role="status" class="rounded-lg bg-amber-50 p-2 text-sm text-amber-800 dark:bg-amber-950 dark:text-amber-200">
    Your session expired. Please sign in again.
  </p>
}
```

## User menu in the top bar

The shared `Menu` from [Module 16 note 4](../../16-advanced-components/04-reusable-ui-kit.md) has a fixed "⋯" trigger, so the account menu is a small disclosure of its own:

```typescript
// core/layout/user-menu.ts
import { Component, computed, inject, signal } from '@angular/core';
import { RouterLink } from '@angular/router';
import { AuthStore } from '../auth/auth-store';
import { ClickOutside } from '../../shared/directives/click-outside';

@Component({
  selector: 'app-user-menu',
  imports: [RouterLink, ClickOutside],
  template: `
    @if (auth.user(); as u) {
      <div class="relative" (appClickOutside)="open.set(false)" (keydown.escape)="open.set(false)">
        <button type="button" (click)="open.update(v => !v)" [attr.aria-expanded]="open()" aria-controls="user-menu"
                class="flex items-center gap-2 rounded-full p-1" [attr.aria-label]="'Account menu for ' + u.name">
          <span class="grid size-8 place-items-center rounded-full bg-blue-600 text-sm font-semibold text-white" aria-hidden="true">{{ initials() }}</span>
          <span class="hidden text-sm font-medium md:inline">{{ u.name }}</span>
        </button>
        @if (open()) {
          <div id="user-menu" class="absolute right-0 z-30 mt-2 w-44 rounded-lg bg-white p-1 shadow-lg dark:bg-slate-800">
            <a routerLink="/settings/profile" (click)="open.set(false)" class="block rounded px-3 py-2 text-sm hover:bg-slate-100 dark:hover:bg-slate-700">Profile</a>
            <button type="button" (click)="auth.logout()" class="block w-full rounded px-3 py-2 text-left text-sm hover:bg-slate-100 dark:hover:bg-slate-700">Sign out</button>
          </div>
        }
      </div>
    }
  `,
})
export class UserMenu {
  protected auth = inject(AuthStore);
  protected open = signal(false);
  protected initials = computed(() =>
    (this.auth.user()?.name ?? '').split(/\s+/).map(w => w[0]).join('').slice(0, 2).toUpperCase());
}
```

This is a disclosure (button + panel of links), not an ARIA `menu`, so it needs no arrow-key handling. Lab 11 revisits menus.

## Independent task — Google callback

```typescript
// environments/environment.ts
export const environment = { production: false, apiUrl: '/api/v1', useMockApi: true,
  googleLoginUrl: '/auth/callback?mock=1' };
// environments/environment.prod.ts → googleLoginUrl: '/api/v1/auth/google'
```

```html
<!-- login-page.html -->
<a [href]="googleLoginUrl" class="block rounded-lg border py-2 text-center text-sm">Continue with Google</a>
```

```typescript
// features/auth/oauth-callback-page.ts
import { Component, afterNextRender, inject, input } from '@angular/core';
import { Router } from '@angular/router';
import { AuthStore } from '../../core/auth/auth-store';
import { mockSession } from '../../core/http/mock-auth';
import { environment } from '../../../environments/environment';

@Component({
  selector: 'app-oauth-callback-page',
  template: `<main class="grid min-h-dvh place-items-center"><p role="status" class="text-slate-500">Signing you in…</p></main>`,
})
export default class OAuthCallbackPage {
  private auth = inject(AuthStore);
  private router = inject(Router);
  error = input<string>();
  mock = input<string>();

  constructor() {
    afterNextRender(async () => {
      if (this.error()) { await this.router.navigate(['/login'], { queryParams: { reason: this.error() } }); return; }
      if (environment.useMockApi && this.mock() === '1') mockSession.alive = true;
      this.auth.status.set('unknown');
      await this.auth.ensureSession();
      await this.router.navigateByUrl(this.auth.isLoggedIn() ? '/dashboard' : '/login', { replaceUrl: true });
    });
  }
}
```

Route: `{ path: 'auth/callback', loadComponent: () => import('./features/auth/oauth-callback-page') }` next to `/login`, outside the shell.

> Importing `mock-auth` into a feature page means the mock code ships in production unless tree-shaken. Guard it with `environment.useMockApi` (a build-time constant per environment file) as shown, or move the "set cookie" step into the mock interceptor by handling `GET /auth/callback-mock` there.

Caveat: `ensureSession()` in the Module 13 store caches the `restoring` promise. If it already ran at startup and failed, setting `status` back to `'unknown'` isn't enough — also reset the cached promise. Add a small `resetForReauth()` method that clears `restoring`, or make `ensureSession()` clear `restoring` in `clear()`.
