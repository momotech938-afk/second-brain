# Lab 9 — Solution

Back to [Lab 9](../lab-09-protected-routes.md) · [Labs index](../README.md)

## Guards

```typescript
// core/auth/auth-guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthStore } from './auth-store';

export const authGuard: CanActivateFn = async (_route, state) => {
  const auth = inject(AuthStore);
  const router = inject(Router);
  await auth.ensureSession();
  return auth.isLoggedIn() || router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

export const guestGuard: CanActivateFn = async () => {
  const auth = inject(AuthStore);
  await auth.ensureSession();
  return !auth.isLoggedIn() || inject(Router).createUrlTree(['/dashboard']);
};
```

```typescript
// core/auth/role-guard.ts
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { AuthStore } from './auth-store';
import type { User } from '../models';

/** canMatch: false → the router tries the next route (ends at the 404 page). */
export const roleGuard = (roles: User['role'][]): CanMatchFn => async () => {
  const auth = inject(AuthStore);
  await auth.ensureSession();
  const role = auth.user()?.role;
  return role !== undefined && roles.includes(role);
};
```

Module 10 note 4 shows a variant that redirects to `/dashboard`. In this lab we return `false` so the URL doesn't reveal the admin area. Both are valid. Pick one and be consistent.

```typescript
// core/feature-flags/feature-guard.ts
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { BootstrapStore } from '../bootstrap/bootstrap-store';

export const featureGuard = (flag: string): CanMatchFn => () =>
  inject(BootstrapStore).data()?.flags[flag] !== false;   // unknown flag → enabled
```

## Route table changes

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: 'login', title: 'Sign in', canActivate: [guestGuard], loadComponent: () => import('./features/auth/login-page') },
  { path: 'auth/callback', loadComponent: () => import('./features/auth/oauth-callback-page') },
  { path: 'lock', title: 'Locked', loadComponent: () => import('./features/auth/lock-page') },
  {
    path: '',
    component: Shell,
    canActivate: [authGuard],
    canActivateChild: [authGuard, lockGuard],
    children: [
      { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
      { path: 'dashboard', title: 'Dashboard', loadComponent: () => import('./features/dashboard/dashboard-page') },
      { path: 'notes', loadChildren: () => import('./features/notes/notes.routes') },
      { path: 'credentials', loadChildren: () => import('./features/credentials/credentials.routes') },
      { path: 'tasks', canMatch: [featureGuard('tasks')], loadChildren: () => import('./features/tasks/tasks.routes') },
      { path: 'settings', loadChildren: () => import('./features/settings/settings.routes') },
      { path: 'admin', canMatch: [roleGuard(['admin', 'super_admin'])], loadChildren: () => import('./features/admin/admin.routes') },
    ],
  },
  { path: '**', title: 'Not found', loadComponent: () => import('./shared/pages/not-found-page') },
];
```

## `BootstrapStore` + mock

Use `BootstrapStore` from [Module 01 note 3](../../01-angular-fundamentals/03-project-structure-and-bootstrap.md) (`core/bootstrap/bootstrap-store.ts`, `data` signal, `load()`), and add a `reset()`:

```typescript
reset() { this.data.set(null); }
```

Load it after a session exists:

```typescript
// app.config.ts
provideAppInitializer(async () => {
  const auth = inject(AuthStore);
  const boot = inject(BootstrapStore);
  await auth.ensureSession();
  if (auth.isLoggedIn()) await boot.load();
}),
```

And in `LoginPage` after `auth.login()`: `await inject(BootstrapStore).load()`. (Keep an injected field, `private boot = inject(BootstrapStore)`, rather than calling `inject` in the method.)

Mock route (planned `GET /bootstrap`; the flag names are **proposed**):

```typescript
if (req.method === 'GET' && path === '/bootstrap') {
  return ok({
    user: USER,
    settings: { theme: 'system' },
    nav: [],
    widgets: ['summary', 'today', 'week-progress'],
    flags: { notes: true, credentials: true, tasks: mockFlags.tasks },
  });
}
```

Expose `mockFlags` from the mock file so you can flip `tasks` from the browser console during development (`window.mockFlags = mockFlags` in dev mode only).

## Sidebar

```html
<!-- core/layout/sidebar.html (excerpt) -->
@for (item of visibleItems(); track item.route) {
  <app-nav-item [label]="item.label" [route]="item.route" [icon]="item.icon" />
}
<app-nav-item *appHasRole="['admin', 'super_admin']" label="Admin" route="/admin" icon="shield" />
```

```typescript
// sidebar.ts (excerpt)
private boot = inject(BootstrapStore);
protected visibleItems = computed(() => {
  const flags = this.boot.data()?.flags ?? {};
  return NAV_ITEMS.filter(i => !i.flag || flags[i.flag] !== false);
});
```

`NAV_ITEMS` gains an optional `flag` field: `{ label: 'Tasks', route: '/tasks', icon: 'check-square', flag: 'tasks' }`.

## Admin placeholder

```typescript
// features/admin/admin.routes.ts
export default [
  {
    path: '',
    loadComponent: () => import('./admin-layout'),
    children: [
      { path: '', pathMatch: 'full', redirectTo: 'appearance' },
      { path: 'appearance', title: 'Admin · Appearance', loadComponent: () => import('./pages/admin-placeholder') },
      { path: 'modules', title: 'Admin · Modules', loadComponent: () => import('./pages/admin-placeholder') },
      { path: 'navigation', title: 'Admin · Navigation', loadComponent: () => import('./pages/admin-placeholder') },
    ],
  },
] satisfies Routes;
```

`AdminLayout` is a copy of `SettingsLayout` (Lab 4) with these three tabs. The page contents are intentionally placeholders: the admin features are confirmed in principle but not specified yet.

## Reset on logout

```typescript
// features/notes/data-access/notes-store.ts (addition)
private auth = inject(AuthStore);

constructor() {
  effect(() => {
    if (this.auth.status() === 'anonymous') untracked(() => this.reset());
  });
}

reset() {
  this.category.set('all');
  this.query.set('');
  this.list.set(undefined);   // drop cached data from the previous user
}
```

Setting the value doesn't refetch by itself (the params didn't change). After the next login, call `notesStore.list.reload()`, or include `auth.user()?.id` in the resource `params` so a new user automatically triggers a fresh load. The second option is cleaner:

```typescript
readonly list = rxResource({
  params: () => this.auth.isLoggedIn()
    ? { user: this.auth.user()!.id, category: this.category(), q: this.query(), page: this.page(), limit: this.limit }
    : undefined,                                   // undefined → resource goes idle, no request
  stream: ({ params }) => this.api.list(params),
});
```

Do the same in `TasksStore`, `CredentialsUiStore` and `BootstrapStore`. Signed-out users can't open the pages anyway; the reset matters for the next sign-in.

## Independent task — idle lock

```typescript
// core/auth/idle-lock.ts
import { DestroyRef, Service, inject, signal } from '@angular/core';
import { DOCUMENT } from '@angular/common';
import { Router } from '@angular/router';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { debounceTime, fromEvent, merge, startWith } from 'rxjs';

const IDLE_MS = 10 * 60_000;

@Service()
export class IdleLock {
  private router = inject(Router);
  private doc = inject(DOCUMENT);
  private destroyRef = inject(DestroyRef);
  readonly locked = signal(false);
  private returnUrl = '/dashboard';

  start() {
    merge(fromEvent(this.doc, 'pointerdown'), fromEvent(this.doc, 'keydown'))
      .pipe(startWith(null), debounceTime(IDLE_MS), takeUntilDestroyed(this.destroyRef))
      .subscribe(() => this.lock());
  }

  lock() {
    if (this.locked()) return;
    this.returnUrl = this.router.url.startsWith('/lock') ? '/dashboard' : this.router.url;
    this.locked.set(true);
    this.router.navigateByUrl('/lock');
  }

  unlock() {
    this.locked.set(false);
    return this.router.navigateByUrl(this.returnUrl);
  }
}
```

```typescript
// core/auth/lock-guard.ts
export const lockGuard: CanActivateChildFn = () =>
  !inject(IdleLock).locked() || inject(Router).createUrlTree(['/lock']);
```

```typescript
// features/auth/lock-page.ts
@Component({
  selector: 'app-lock-page',
  imports: [FormField, PasswordInput],
  template: `
    <main class="grid min-h-dvh place-items-center p-4">
      <form (submit)="unlock($event)" class="w-full max-w-sm space-y-4 rounded-2xl bg-white p-6 text-center shadow-sm dark:bg-slate-900">
        <p class="text-sm text-slate-500">Locked after inactivity</p>
        <h1 class="text-xl font-semibold">{{ auth.user()?.name }}</h1>
        <label for="lock-pw" class="sr-only">Password</label>
        <app-password-input inputId="lock-pw" [formField]="pwForm.password" />
        @for (e of pwForm().errors(); track e.kind) { <p role="alert" class="text-sm text-rose-600">{{ e.message }}</p> }
        <button type="submit" class="w-full rounded-lg bg-blue-600 py-2 text-white" [disabled]="pwForm().submitting()">Unlock</button>
        <button type="button" class="text-sm underline" (click)="auth.logout()">Sign out instead</button>
      </form>
    </main>
  `,
})
export default class LockPage {
  protected auth = inject(AuthStore);
  private idle = inject(IdleLock);
  private http = inject(HttpClient);
  private api = inject(API_URL);
  protected model = signal({ password: '' });
  protected pwForm = form(this.model, p => required(p.password, { message: 'Enter your password.' }));

  async unlock(e: Event) {
    e.preventDefault();
    await submit(this.pwForm, async f => {
      try {
        await firstValueFrom(this.http.post(`${this.api}/auth/reauth`, { password: f().value().password },
          { context: new HttpContext().set(SKIP_ERROR_TOAST, true) }));
      } catch {
        return { kind: 'wrongPassword', message: 'Incorrect password.' };
      }
      this.model.set({ password: '' });
      await this.idle.unlock();
      return undefined;
    });
  }
}
```

Start it with `provideAppInitializer(() => inject(IdleLock).start())`. Mock `POST /auth/reauth` checks the password against `Password123!`.

Limitations to know:
- The lock is a client-side UX feature. The access token stays in memory while locked. A stronger version also drops the access token on lock and requires `/auth/reauth` to issue a new one. That is a backend decision, so it is listed for the NestJS design, not invented here.
- `/lock` requires a session: if `status` is `'anonymous'`, `LockPage` should redirect to `/login`.
