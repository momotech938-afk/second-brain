# Router Basics

Back to [Module 10](README.md) · Next: [Params and navigation](02-params-and-navigation.md)

## 1. Learning objectives

- Configure the router and its features.
- Write route definitions with paths, components, redirects, wildcards and titles.
- Navigate with `routerLink` and highlight the active link.
- Understand how Angular matches URLs.

## 2. Prerequisites

[Project structure and bootstrap](../01-angular-fundamentals/03-project-structure-and-bootstrap.md).

## 3. What is it?

The Angular Router maps the browser URL to a tree of components, renders them into `<router-outlet>`s, and updates the URL as users navigate — without full page reloads.

## 4. Why does Angular need it?

Myspce has many screens. Users expect the back button, bookmarks (`/notes?category=work`), deep links (`/notes/01J…`) and refresh to work. The router provides that, plus lazy loading and access control.

## 5. How it works internally

1. A navigation starts (link click, `router.navigate`, back button, initial load).
2. The router parses the URL into a `UrlTree`.
3. It **matches** the URL against the route config, depth-first, in order.
4. It runs guards (`canMatch` during matching, then `canActivate*`), resolvers.
5. It activates components into outlets and updates the browser URL.
6. It emits router events (`NavigationStart`, `NavigationEnd`, `NavigationError`, …).

Matching rules:

- Routes are tried **in order**; the first full match wins.
- `pathMatch: 'prefix'` (default) matches if the URL starts with the path; `'full'` requires the whole remaining URL to match (needed for empty-path redirects).
- `**` matches anything — put it **last**.

## 6. Syntax and API reference

```typescript
// app.config.ts
provideRouter(
  routes,
  withComponentInputBinding(),                         // params → component inputs
  withViewTransitions(),                               // animated route changes (browser View Transitions API)
  withInMemoryScrolling({ scrollPositionRestoration: 'top', anchorScrolling: 'enabled' }),
  withRouterConfig({ paramsInheritanceStrategy: 'always' }),   // the v22 default; shown for clarity
  // withHashLocation()                                // only if your server can't do SPA fallback
  // withPreloading(PreloadAllModules)                 // see note 3
)
```

```typescript
// Route fields you'll use
{
  path: 'notes/:id',
  component: NoteDetailPage,               // eager component
  loadComponent: () => import('…'),        // or lazy component (one of the two)
  loadChildren: () => import('…'),         // lazy child routes
  children: [ … ],                         // nested routes
  redirectTo: 'dashboard',                 // string or function
  pathMatch: 'full',
  title: 'Note',                           // string, ResolveFn<string>, or via TitleStrategy
  data: { breadcrumb: 'Note' },            // static data
  resolve: { note: noteResolver },
  canActivate: [authGuard], canActivateChild: [], canMatch: [], canDeactivate: [],
  providers: [ … ],                        // route-scoped environment injector
  outlet: 'modal',                         // named outlet (secondary route)
}
```

Template:

```html
<router-outlet />                                         <!-- import RouterOutlet -->
<a routerLink="/notes">Notes</a>                          <!-- import RouterLink -->
<a [routerLink]="['/notes', note.id]">Open</a>
<a routerLink="/notes" [queryParams]="{ category: 'work' }">Work notes</a>
<a routerLink="/notes" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">Notes</a>
```

`routerLink` values starting with `/` are absolute; without `/` they're relative to the current route (`['../', id]` goes up).

## 7. Smallest working example

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
  { path: 'dashboard', title: 'Dashboard · MySpace', loadComponent: () => import('./features/dashboard/pages/dashboard-page').then(m => m.DashboardPage) },
  { path: 'notes', title: 'Notes · MySpace', loadComponent: () => import('./features/notes/pages/notes-page').then(m => m.NotesPage) },
  { path: '**', title: 'Not found · MySpace', loadComponent: () => import('./shared/pages/not-found-page').then(m => m.NotFoundPage) },
];
```

```typescript
// app.ts
@Component({ selector: 'app-root', imports: [RouterOutlet], template: `<router-outlet />` })
export class App {}
```

## 8. Step-by-step

1. `/` → redirect to `/dashboard` (`pathMatch: 'full'` so it doesn't catch everything).
2. `loadComponent` lazy-loads each page.
3. `title` sets `document.title`.
4. `**` catches unknown URLs.

If a lazy file uses `export default class NotesPage`, you can write `loadComponent: () => import('./…/notes-page')` without `.then`.

## 9. Myspce real-world example — the full v1 route table

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard, guestGuard } from './core/auth/auth-guard';
import { roleGuard } from './core/auth/role-guard';
import { featureGuard } from './core/feature-flags/feature-guard';
import { Shell } from './core/layout/shell';

export const routes: Routes = [
  // Public
  { path: 'login', title: 'Sign in', canActivate: [guestGuard], loadComponent: () => import('./features/auth/login-page') },
  { path: 'auth/callback', loadComponent: () => import('./features/auth/oauth-callback-page') },  // Google redirect

  // Authenticated app inside the shell layout
  {
    path: '',
    component: Shell,
    canActivate: [authGuard],
    canActivateChild: [authGuard],
    children: [
      { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
      { path: 'dashboard', title: 'Dashboard', loadComponent: () => import('./features/dashboard/pages/dashboard-page') },
      { path: 'notes', loadChildren: () => import('./features/notes/notes.routes') },
      { path: 'credentials', loadChildren: () => import('./features/credentials/credentials.routes') },
      { path: 'tasks', loadChildren: () => import('./features/tasks/tasks.routes') },
      { path: 'favorites', title: 'Favorites', loadComponent: () => import('./features/favorites/favorites-page') },
      { path: 'trash', title: 'Trash', loadComponent: () => import('./features/trash/trash-page') },
      { path: 'settings', loadChildren: () => import('./features/settings/settings.routes') },

      // v1.1 modules behind admin feature flags
      { path: 'memories', canMatch: [featureGuard('memories')], loadChildren: () => import('./features/memories/memories.routes') },
      { path: 'files', canMatch: [featureGuard('files')], loadChildren: () => import('./features/files/files.routes') },
      { path: 'calendar', canMatch: [featureGuard('calendar')], loadChildren: () => import('./features/calendar/calendar.routes') },

      // Admin panel
      { path: 'admin', canMatch: [roleGuard(['admin', 'super_admin'])], loadChildren: () => import('./admin/admin.routes') },
    ],
  },

  { path: '**', title: 'Not found', loadComponent: () => import('./shared/pages/not-found-page') },
];
```

(These examples assume lazy files use `export default`. Guards are built in [note 4](04-guards-and-resolvers.md).)

### Consistent titles with a `TitleStrategy`

```typescript
// core/routing/myspace-title-strategy.ts
import { Injectable, inject } from '@angular/core';
import { Title } from '@angular/platform-browser';
import { RouterStateSnapshot, TitleStrategy } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class MySpaceTitleStrategy extends TitleStrategy {
  private title = inject(Title);
  override updateTitle(snapshot: RouterStateSnapshot) {
    const t = this.buildTitle(snapshot);
    this.title.setTitle(t ? `${t} · MySpace` : 'MySpace');
  }
}
// app.config.ts: { provide: TitleStrategy, useClass: MySpaceTitleStrategy }
```

Now routes just say `title: 'Notes'`.

## 10. Advanced example — redirect functions

```typescript
{
  path: 'n/:id',                                       // short share link
  redirectTo: ({ params }) => `/notes/${params['id']}`,
},
{
  path: 'home',
  redirectTo: () => (inject(AuthStore).isLoggedIn() ? '/dashboard' : '/login'),
},
```

Redirect functions run in an injection context and can return a string or a `UrlTree`.

## 11. Visual explanation

```text
URL: /notes/01JABC?tab=edit
routes:
  ''      (Shell)            ✔ prefix match ''
   └─ 'notes' (lazy)         ✔ matches 'notes'
        └─ ':id'             ✔ id = 01JABC
Rendered: App <router-outlet> → Shell <router-outlet> → NoteDetailPage
```

## 12. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| `{ path: '', redirectTo: 'dashboard' }` without `pathMatch: 'full'` | redirect loop / everything redirects | add `pathMatch: 'full'` |
| `**` not last | swallows later routes | move it to the end |
| Forgetting `RouterOutlet`/`RouterLink` in `imports` | nothing renders / links don't navigate | import them |
| `href="/notes"` instead of `routerLink` | full page reload | `routerLink` |
| Exact active matching on parent links | Dashboard link always active | `routerLinkActiveOptions: { exact: true }` where needed |

## 13. Debugging guide

- `withDebugTracing()` in `provideRouter` (dev only) logs every router event.
- `NG04002: Cannot match any routes. URL Segment: 'x'` → no route matched (and no `**`).
- Lazy chunk fails to load after a deploy (`ChunkLoadError`) → old tab requesting deleted files; handle `NavigationError` by reloading ([Module 20](../20-security-and-deployment/04-deployment-and-ci.md)).

## 14. Best practices

- Lazy-load every page.
- Public pages outside the shell; authenticated pages as children of it.
- One `TitleStrategy` for consistent titles.
- Keep the root routes file short; each feature has its own routes file.

## 15. Angular vs React

`Routes` config ≈ React Router's `createBrowserRouter([...])`. `<router-outlet>` ≈ `<Outlet />`. `routerLink` ≈ `<Link to>`. `loadComponent` ≈ `lazy: () => import()`. Guards ≈ loaders/redirects in route config.

## 16. Testing

```typescript
import { RouterTestingHarness } from '@angular/router/testing';

it('redirects / to /dashboard', async () => {
  TestBed.configureTestingModule({ providers: [provideRouter(routes)] });
  const harness = await RouterTestingHarness.create('/');
  expect(TestBed.inject(Router).url).toBe('/dashboard');
});
```

(Provide fake guards/stores when testing the real route table.)

## 17. Practice questions

1. Add `/settings/profile`, `/settings/appearance`, `/settings/security` with `/settings` redirecting to profile.
2. Why must the empty-path redirect use `pathMatch: 'full'`?
3. Add a short link `/c/:id` that redirects to `/credentials?open=:id`.

## 18. Interview questions

- **Q: How does route matching work?** Depth-first, in config order, first complete match wins; `pathMatch` controls prefix vs full matching.
- **Q: `loadComponent` vs `loadChildren`?** The first lazy-loads a single standalone component; the second lazy-loads a routes array (a feature with child routes).

## 19. Revision notes

`provideRouter(routes, with…())` · order matters · `pathMatch: 'full'` for empty redirects · `**` last · `title` + `TitleStrategy` · `routerLink` / `routerLinkActive` · redirect functions.

## 20. Completion checklist

- [ ] Myspce route table written with redirects, titles, 404.
- [ ] Sidebar uses `routerLink` + `routerLinkActive`.
- [ ] `withDebugTracing()` used once to watch a navigation.
