# Lab 2 — Solution

Back to [Lab 2](../lab-02-layout-and-navigation.md) · [Labs index](../README.md)

## `ThemeStore`

```typescript
// core/theme/theme-store.ts
import { Service, computed, effect, signal } from '@angular/core';

export type ThemeMode = 'light' | 'dark' | 'system';
const KEY = 'myspce.theme';

@Service()
export class ThemeStore {
  readonly mode = signal<ThemeMode>(read());
  private readonly systemDark = signal(matchMedia('(prefers-color-scheme: dark)').matches);
  readonly isDark = computed(() => this.mode() === 'dark' || (this.mode() === 'system' && this.systemDark()));

  constructor() {
    matchMedia('(prefers-color-scheme: dark)').addEventListener('change', e => this.systemDark.set(e.matches));
    effect(() => {
      document.documentElement.classList.toggle('dark', this.isDark());
      try { localStorage.setItem(KEY, this.mode()); } catch { /* storage unavailable */ }
    });
  }
  toggle() { this.mode.set(this.isDark() ? 'light' : 'dark'); }
}

function read(): ThemeMode {
  try {
    const v = localStorage.getItem(KEY);
    return v === 'light' || v === 'dark' || v === 'system' ? v : 'system';
  } catch { return 'system'; }
}
```

## `NavItem`

```typescript
// core/layout/nav-item.ts
import { Component, input } from '@angular/core';
import { RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-nav-item',
  imports: [RouterLink, RouterLinkActive],
  template: `
    <a [routerLink]="route()" routerLinkActive="bg-blue-600 !text-white shadow-sm" ariaCurrentWhenActive="page"
       class="flex items-center gap-3 rounded-lg px-3 py-2 text-sm text-slate-600 hover:bg-slate-100 dark:text-slate-300 dark:hover:bg-slate-800">
      <span aria-hidden="true" class="w-5 text-center">{{ icon() }}</span>
      <span>{{ label() }}</span>
    </a>
  `,
})
export class NavItem {
  label = input.required<string>();
  route = input.required<string>();
  icon = input('•');
}
```

## `Sidebar`

```typescript
// core/layout/sidebar.ts
import { Component, input, output } from '@angular/core';
import { NavItem } from './nav-item';
import { MAIN_NAV } from './nav';

@Component({
  selector: 'app-sidebar',
  imports: [NavItem],
  template: `
    <aside class="fixed inset-y-0 left-0 z-40 flex w-64 -translate-x-full flex-col border-r border-slate-200 bg-white p-4 transition-transform
                  lg:static lg:translate-x-0 dark:border-slate-800 dark:bg-slate-900"
           [class.translate-x-0]="open()" aria-label="Sidebar">
      <div class="mb-6 flex items-center gap-2 text-lg font-bold">
        <span class="grid size-8 place-items-center rounded-lg bg-blue-600 text-white" aria-hidden="true">M</span> MySpace
      </div>
      <div class="mb-6 flex items-center gap-3 rounded-lg bg-slate-100 p-3 dark:bg-slate-800">
        <span class="grid size-9 place-items-center rounded-full bg-blue-600 text-sm font-semibold text-white" aria-hidden="true">PG</span>
        <div class="min-w-0">
          <p class="truncate text-sm font-semibold">Pranjal Goyal</p>
          <p class="truncate text-xs text-slate-500">pranjal@example.com</p>
        </div>
      </div>
      <nav aria-label="Main" class="flex flex-1 flex-col gap-1" (click)="closed.emit()">
        @for (item of nav; track item.route) {
          <app-nav-item [label]="item.label" [route]="item.route" [icon]="item.icon" />
        }
        <div class="mt-auto"><app-nav-item label="Settings" route="/settings" icon="⚙" /></div>
      </nav>
    </aside>
  `,
})
export class Sidebar {
  open = input(false);
  closed = output<void>();
  protected readonly nav = MAIN_NAV;
}
```

## `Topbar`

```typescript
// core/layout/topbar.ts
import { Component, inject, output } from '@angular/core';
import { ThemeStore } from '../theme/theme-store';

@Component({
  selector: 'app-topbar',
  template: `
    <header class="sticky top-0 z-30 flex h-16 items-center gap-3 border-b border-slate-200 bg-white/80 px-4 backdrop-blur dark:border-slate-800 dark:bg-slate-900/80">
      <button type="button" class="rounded-lg p-2 lg:hidden" aria-label="Open menu" (click)="menuClicked.emit()">☰</button>
      <label for="global-search" class="sr-only">Search</label>
      <input id="global-search" type="search" placeholder="Search anything…"
             class="w-full max-w-md rounded-lg bg-slate-100 px-3 py-2 text-sm dark:bg-slate-800" />
      <div class="ml-auto flex items-center gap-2">
        <button type="button" class="rounded-lg p-2" (click)="theme.toggle()"
                [attr.aria-label]="theme.isDark() ? 'Switch to light theme' : 'Switch to dark theme'">
          {{ theme.isDark() ? '☀' : '☾' }}
        </button>
        <button type="button" class="rounded-lg p-2" aria-label="Notifications">🔔</button>
        <span class="grid size-8 place-items-center rounded-full bg-blue-600 text-xs font-semibold text-white" aria-label="Pranjal Goyal">PG</span>
      </div>
    </header>
  `,
})
export class Topbar {
  menuClicked = output<void>();
  protected theme = inject(ThemeStore);
}
```

## `Shell`

```typescript
// core/layout/shell.ts
import { Component, signal } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { Sidebar } from './sidebar';
import { Topbar } from './topbar';

@Component({
  selector: 'app-shell',
  imports: [RouterOutlet, Sidebar, Topbar],
  template: `
    <div class="flex min-h-dvh">
      <app-sidebar [open]="sidebarOpen()" (closed)="sidebarOpen.set(false)" />
      @if (sidebarOpen()) {
        <div data-testid="backdrop" class="fixed inset-0 z-30 bg-black/40 lg:hidden" (click)="sidebarOpen.set(false)"></div>
      }
      <div class="flex min-w-0 flex-1 flex-col">
        <app-topbar (menuClicked)="sidebarOpen.set(true)" />
        <main id="main" tabindex="-1" class="flex-1 p-4 lg:p-8"><router-outlet /></main>
      </div>
    </div>
  `,
})
export class Shell {
  protected sidebarOpen = signal(false);
}
```

## `PageHeader`

See [Module 16 note 1](../../16-advanced-components/01-content-projection.md#7-smallest-working-example--pageheader).

## Placeholder pages

```typescript
// features/dashboard/dashboard-page.ts
import { Component } from '@angular/core';
import { PageHeader } from '../../shared/ui/page-header';

@Component({ selector: 'app-dashboard-page', imports: [PageHeader], template: `<app-page-header title="Dashboard" />` })
export default class DashboardPage {}
```

## Independent task — overview tiles

```typescript
// features/dashboard/components/overview-tile.ts
import { Component, input } from '@angular/core';
import { RouterLink } from '@angular/router';
import { DecimalPipe } from '@angular/common';

@Component({
  selector: 'app-overview-tile',
  imports: [RouterLink, DecimalPipe],
  template: `
    <a [routerLink]="route()" [attr.aria-label]="label() + ': ' + count()"
       class="flex flex-col gap-2 rounded-xl border border-slate-200 bg-white p-4 hover:shadow-md dark:border-slate-800 dark:bg-slate-900">
      <span class="text-sm text-slate-500">{{ label() }}</span>
      <strong class="text-xl font-semibold">{{ count() | number }}</strong>
    </a>
  `,
})
export class OverviewTile {
  label = input.required<string>();
  count = input.required<number>();
  route = input.required<string>();
}
```

```typescript
// features/dashboard/dashboard-page.ts (updated)
@Component({
  selector: 'app-dashboard-page',
  imports: [PageHeader, OverviewTile],
  template: `
    <app-page-header title="Dashboard" />
    <h2 class="mb-3 font-semibold">Quick Overview</h2>
    <div class="grid grid-cols-2 gap-3 md:grid-cols-4">
      @for (t of tiles; track t.label) { <app-overview-tile [label]="t.label" [count]="t.count" [route]="t.route" /> }
    </div>
  `,
})
export default class DashboardPage {
  protected readonly tiles = [
    { label: 'Notes', count: 124, route: '/notes' },
    { label: 'Credentials', count: 28, route: '/credentials' },
    { label: 'Memories', count: 48, route: '/memories' },
    { label: 'Tasks', count: 12, route: '/tasks' },
    { label: 'Files', count: 68, route: '/files' },
    { label: 'Events', count: 6, route: '/calendar' },
    { label: 'Favorites', count: 23, route: '/favorites' },
    { label: 'Trash', count: 4, route: '/trash' },
  ];
}
```

Hints: the counts become `GET /dashboard/summary` in Lab 6; links to v1.1 modules will hit the 404 page until those routes exist (Lab 4).
