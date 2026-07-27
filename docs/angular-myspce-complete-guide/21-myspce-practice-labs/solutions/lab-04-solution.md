# Lab 4 — Solution

Back to [Lab 4](../lab-04-routing-and-nested-layouts.md) · [Labs index](../README.md)

## `app.routes.ts`

```typescript
import { Routes } from '@angular/router';
import { Shell } from './core/layout/shell';

export const routes: Routes = [
  { path: 'login', title: 'Sign in', loadComponent: () => import('./features/auth/login-page') },
  {
    path: '',
    component: Shell,
    children: [
      { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
      { path: 'dashboard', title: 'Dashboard', loadComponent: () => import('./features/dashboard/dashboard-page') },
      { path: 'notes', loadChildren: () => import('./features/notes/notes.routes') },
      { path: 'credentials', loadChildren: () => import('./features/credentials/credentials.routes') },
      { path: 'tasks', loadChildren: () => import('./features/tasks/tasks.routes') },
      { path: 'settings', loadChildren: () => import('./features/settings/settings.routes') },
    ],
  },
  { path: '**', title: 'Not found', loadComponent: () => import('./shared/pages/not-found-page') },
];
```

## Notes routes and editor

```typescript
// features/notes/notes.routes.ts
import { Routes } from '@angular/router';
import { unsavedChangesGuard } from '../../core/routing/unsaved-changes-guard';

export default [
  { path: '', title: 'Notes', loadComponent: () => import('./notes-page') },
  { path: 'new', title: 'New note', loadComponent: () => import('./note-editor-page'), canDeactivate: [unsavedChangesGuard] },
  { path: ':id', title: 'Note', loadComponent: () => import('./note-editor-page'), canDeactivate: [unsavedChangesGuard] },
] satisfies Routes;
```

```typescript
// features/notes/note-editor-page.ts
import { Component, computed, input, signal } from '@angular/core';
import { PageHeader } from '../../shared/ui/page-header';
import { HasUnsavedChanges } from '../../core/routing/unsaved-changes-guard';

@Component({
  selector: 'app-note-editor-page',
  imports: [PageHeader],
  template: `
    <app-page-header [title]="id() ? 'Edit note ' + id() : 'New note'" />
    <label for="title" class="text-sm font-medium">Title</label>
    <input id="title" class="mt-1 w-full rounded-lg border px-3 py-2" [value]="draft()" (input)="draft.set($any($event.target).value)" />
  `,
})
export default class NoteEditorPage implements HasUnsavedChanges {
  id = input<string>();
  protected draft = signal('');
  private dirty = computed(() => this.draft().trim().length > 0);
  hasUnsavedChanges() { return this.dirty(); }
}
```

## Guard

```typescript
// core/routing/unsaved-changes-guard.ts
import { CanDeactivateFn } from '@angular/router';

export interface HasUnsavedChanges { hasUnsavedChanges(): boolean; }

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = component =>
  !component.hasUnsavedChanges() || window.confirm('Discard changes?');   // replaced by ConfirmService in Lab 11
```

## Settings

```typescript
// features/settings/settings.routes.ts
import { Routes } from '@angular/router';

export default [
  {
    path: '',
    loadComponent: () => import('./settings-layout'),
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

```typescript
// features/settings/settings-layout.ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-settings-layout',
  imports: [RouterLink, RouterLinkActive, RouterOutlet],
  template: `
    <h1 class="mb-4 text-2xl font-bold">Settings</h1>
    <nav aria-label="Settings sections" class="mb-6 flex gap-2 overflow-x-auto">
      @for (t of tabs; track t.path) {
        <a [routerLink]="t.path" routerLinkActive="bg-blue-600 text-white" ariaCurrentWhenActive="page"
           class="shrink-0 rounded-lg px-3 py-1.5 text-sm">{{ t.label }}</a>
      }
    </nav>
    <router-outlet />
  `,
})
export default class SettingsLayout {
  protected readonly tabs = [
    { path: 'profile', label: 'Profile' },
    { path: 'appearance', label: 'Appearance' },
    { path: 'security', label: 'Security' },
    { path: 'customization', label: 'Customization' },
    { path: 'backup', label: 'Backup' },
  ];
}
```

## Title strategy

See [Module 10 note 1](../../10-routing/01-router-basics.md#consistent-titles-with-a-titlestrategy); provide `{ provide: TitleStrategy, useClass: MySpaceTitleStrategy }` in `app.config.ts`.

## 404 page

```typescript
// shared/pages/not-found-page.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-not-found-page',
  imports: [RouterLink],
  template: `
    <main class="grid min-h-dvh place-items-center p-4 text-center">
      <div><h1 class="text-4xl font-bold">404</h1><p class="mt-2 text-slate-500">This page doesn't exist.</p>
      <a routerLink="/dashboard" class="mt-4 inline-block rounded-lg bg-blue-600 px-4 py-2 text-white">Go to dashboard</a></div>
    </main>
  `,
})
export default class NotFoundPage {}
```

## Independent task — Tasks tabs

```typescript
// features/tasks/tasks.routes.ts
export default [
  {
    path: '',
    loadComponent: () => import('./tasks-layout'),
    children: [
      { path: '', title: 'Tasks', loadComponent: () => import('./pages/tasks-tab') },
      { path: 'goals', title: 'Goals', loadComponent: () => import('./pages/goals-tab') },
      { path: 'habits', title: 'Habits', loadComponent: () => import('./pages/habits-tab') },   // "Coming soon" placeholder
    ],
  },
] satisfies Routes;
```

`TasksLayout` mirrors `SettingsLayout` with tabs `[{ path: '.', label: 'Tasks', exact: true }, { path: 'goals', … }, { path: 'habits', … }]`; use `[routerLinkActiveOptions]="{ exact: t.exact ?? false }"` so "Tasks" isn't active on `/tasks/goals`.

Hint: `routerLink="."` points to the layout route itself (`/tasks`).
