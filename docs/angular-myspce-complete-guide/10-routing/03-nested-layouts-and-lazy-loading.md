# Nested Layouts and Lazy Loading

Back to [Module 10](README.md) · Previous: [Params and navigation](02-params-and-navigation.md) · Next: [Guards and resolvers](04-guards-and-resolvers.md)

## 1. Learning objectives

- Build the Myspce shell (sidebar + topbar) as a layout route with children.
- Organise each feature's routes in its own file and lazy-load them.
- Use component-less routes, nested layouts and named outlets.
- Scope providers to routes and configure preloading.

## 2. Prerequisites

[Router basics](01-router-basics.md), [Injector hierarchy](../07-services-and-dependency-injection/03-injector-hierarchy.md).

## 3. What is it?

- A **layout route** has a component with its own `<router-outlet>`; its `children` render inside it.
- **Lazy loading** splits each feature into a separate JavaScript chunk that downloads only when the user navigates there.

## 4. Why does Angular need it?

Every Myspce page shares the sidebar/topbar; Login doesn't. Lazy loading keeps the first load small: the login page shouldn't download the credentials vault, calendar, or admin panel code.

## 5. How it works internally

`loadComponent`/`loadChildren` return dynamic `import()`s. The build tool creates a chunk per import. On first navigation, the router awaits the import, caches it, then continues matching. Routes with `providers` get an **environment injector** created when the route is first activated.

## 6. Syntax and API reference

```typescript
// Layout route
{ path: '', component: Shell, children: [ /* pages */ ] }

// Component-less grouping route (shared guard/providers/params, no extra component)
{ path: 'settings', canActivateChild: [authGuard], children: [ … ] }

// Lazy single page
{ path: 'trash', loadComponent: () => import('./features/trash/trash-page') }

// Lazy feature routes file
{ path: 'notes', loadChildren: () => import('./features/notes/notes.routes') }

// Named outlet (e.g. side drawer)
{ path: 'preview/:id', outlet: 'drawer', loadComponent: () => import('./features/notes/note-preview') }
// <router-outlet name="drawer" />   link: [routerLink]="[{ outlets: { drawer: ['preview', id] } }]"
```

Feature routes file:

```typescript
// features/notes/notes.routes.ts
import { Routes } from '@angular/router';
import { unsavedChangesGuard } from '../../core/routing/unsaved-changes-guard';
import { noteResolver, noteTitleResolver } from './data-access/note-resolver';

export default [
  { path: '', title: 'Notes', loadComponent: () => import('./pages/notes-page') },
  { path: 'new', title: 'New note', loadComponent: () => import('./pages/note-editor-page'), canDeactivate: [unsavedChangesGuard] },
  {
    path: ':id',
    resolve: { note: noteResolver },
    title: noteTitleResolver,            // a ResolveFn<string>; see note 4
    loadComponent: () => import('./pages/note-editor-page'),
    canDeactivate: [unsavedChangesGuard],
  },
] satisfies Routes;
```

> The `title` property accepts a string or a `ResolveFn<string>`. Title resolvers run alongside other resolvers, so don't rely on `route.data['note']` being ready inside them — fetch (or read from a cache) independently.

## 7. Smallest working example — the shell

```typescript
// core/layout/shell.ts
import { Component, inject, signal } from '@angular/core';
import { RouterOutlet, Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { Sidebar } from './sidebar';
import { Topbar } from './topbar';

@Component({
  selector: 'app-shell',
  imports: [RouterOutlet, Sidebar, Topbar],
  template: `
    <div class="flex min-h-dvh bg-slate-50 text-slate-900 dark:bg-slate-950 dark:text-slate-100">
      <app-sidebar [open]="sidebarOpen()" (closed)="sidebarOpen.set(false)" />
      <div class="flex min-w-0 flex-1 flex-col">
        <app-topbar (menuClicked)="sidebarOpen.set(true)" />
        <main id="main" class="flex-1 p-4 lg:p-8" tabindex="-1">
          <router-outlet />
        </main>
      </div>
    </div>
  `,
})
export class Shell {
  protected sidebarOpen = signal(false);

  constructor() {
    // close the mobile sidebar after each navigation
    inject(Router).events.pipe(filter(e => e instanceof NavigationEnd), takeUntilDestroyed())
      .subscribe(() => this.sidebarOpen.set(false));
  }
}
```

## 8. Step-by-step

1. `Shell` renders once for all authenticated pages; only the inner `<router-outlet>` changes.
2. The sidebar's own state (collapsed groups) survives navigation because `Shell` isn't destroyed.
3. `NavigationEnd` closes the mobile menu.

## 9. Myspce real-world example — Settings with nested tabs

The Settings design has tabs Profile / Appearance / Security / Customization / Backup. Make each tab a child route so it's linkable (`/settings/security`):

```typescript
// features/settings/settings.routes.ts
export default [
  {
    path: '',
    loadComponent: () => import('./settings-layout'),      // page header + tab bar + <router-outlet>
    children: [
      { path: '', pathMatch: 'full', redirectTo: 'profile' },
      { path: 'profile', title: 'Profile', loadComponent: () => import('./tabs/profile-tab') },
      { path: 'appearance', title: 'Appearance', loadComponent: () => import('./tabs/appearance-tab') },
      { path: 'security', title: 'Security', loadComponent: () => import('./tabs/security-tab') },
      { path: 'customization', title: 'Customization', loadComponent: () => import('./tabs/customization-tab') },
      { path: 'backup', title: 'Backup', loadComponent: () => import('./tabs/backup-tab') },
    ],
  },
] satisfies Routes;
```

```html
<!-- features/settings/settings-layout.html -->
<h1 class="mb-4 text-2xl font-bold">Settings</h1>
<nav aria-label="Settings sections" class="mb-6 flex gap-2 overflow-x-auto">
  @for (t of tabs; track t.path) {
    <a [routerLink]="t.path" routerLinkActive="bg-blue-600 text-white" ariaCurrentWhenActive="page"
       class="rounded-lg px-3 py-1.5 text-sm">{{ t.label }}</a>
  }
</nav>
<router-outlet />
```

Three levels of layout: `App` → `Shell` → `SettingsLayout` → tab.

## 10. Advanced example — route providers and preloading

```typescript
// app.routes.ts (excerpt)
{
  path: 'admin',
  canMatch: [roleGuard(['admin', 'super_admin'])],
  providers: [AdminStore],                         // one AdminStore per admin route injector
  loadChildren: () => import('./admin/admin.routes'),
}
```

```typescript
// app.config.ts — preload lazy routes after the app is idle
import { PreloadAllModules, withPreloading } from '@angular/router';
provideRouter(routes, withPreloading(PreloadAllModules))
```

`PreloadAllModules` ignores `canMatch`-blocked routes (they aren't loaded unless matched), so non-admins never download admin code. For finer control, write a custom `PreloadingStrategy` that preloads only routes with `data: { preload: true }` (e.g. Notes and Tasks, the most used).

## 11. Visual explanation

```text
initial bundle: main (App, Shell, Sidebar, Topbar, AuthStore, router)
lazy chunks:    login · dashboard · notes · credentials · tasks · settings · trash · favorites
                memories* · files* · calendar*        (*behind feature flags)
                admin**                                 (**only if role matches)
```

## 12. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| Importing a lazy page component directly anywhere else | it's pulled into the main bundle | only reference it via `import()` in routes |
| Login inside the shell | sidebar flashes on the login page | put public routes outside the shell |
| Each settings tab as a `@switch` instead of child routes | not linkable, no back button | child routes |
| Huge shared module of components imported everywhere | bloated chunks | standalone imports per component |
| Feature routes file `export const routes` but `loadChildren` expects default | `undefined` children | `export default` or `.then(m => m.routes)` |

## 13. Debugging guide

- Check the build output: each lazy route should appear as a **Lazy chunk** with its name.
- Network tab: navigate and watch the chunk download; if it downloads at startup, something imports it eagerly.
- `ChunkLoadError` after deploy: old tab, new files. Reload on `NavigationError` with that error.

## 14. Best practices

- Shell layout route + public routes outside it.
- Every feature has `feature.routes.ts` with `export default`.
- `canMatch` for role/flag-gated features so their code isn't even downloaded.
- Selective preloading for the most-used features.

## 15. Angular vs React

Layout routes ≈ React Router layouts with `<Outlet />`. `loadChildren` ≈ lazy route modules. Route `providers` have no direct React equivalent.

## 16. Testing

```typescript
it('renders the settings profile tab by default', async () => {
  TestBed.configureTestingModule({ providers: [provideRouter([{ path: 'settings', loadChildren: () => import('./settings.routes') }])] });
  const harness = await RouterTestingHarness.create('/settings');
  expect(TestBed.inject(Router).url).toBe('/settings/profile');
  expect(harness.routeNativeElement?.textContent).toContain('Profile');
});
```

## 17. Practice questions

1. Build the Tasks feature with child routes `/tasks` (list), `/tasks/goals`, `/tasks/habits`.
2. Write a custom preloading strategy using `data.preload`.
3. Where would a "note preview drawer" named outlet live, and what would its URL look like?

## 18. Interview questions

- **Q: What's a component-less route?** A route with `children` but no component, used to group children under a shared path, guard or providers.
- **Q: How does lazy loading reduce the initial bundle?** Each `import()` in a route becomes a separate chunk loaded only on navigation.
- **Q: Why `canMatch` for admin routes?** If it fails, the route isn't matched, so its lazy code is never downloaded.

## 19. Revision notes

Shell layout + children · feature routes files (`export default`) · `loadComponent`/`loadChildren` · component-less grouping · named outlets · route `providers` · `withPreloading`.

## 20. Completion checklist

- [ ] Shell layout with nested routes works.
- [ ] Settings tabs are child routes.
- [ ] Build output shows one lazy chunk per feature.
