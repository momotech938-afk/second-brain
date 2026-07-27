# Lab 2 — Reusable Layout and Navigation Components

Back to [Labs](README.md) · Solution: [lab-02-solution.md](solutions/lab-02-solution.md) · Previous: [Lab 1](lab-01-app-shell.md) · Next: [Lab 3](lab-03-forms-and-controls.md)

## 1. Learning objective

Build the Myspce shell from the design: sidebar (logo, user card, nav items, Settings at the bottom), top bar (search, theme toggle, bell, avatar), responsive off-canvas behaviour, and a reusable `PageHeader`.

## 2. Prerequisites

[Module 02](../02-components-and-templates/README.md), [Module 04](../04-component-communication/README.md), [Lab 1](lab-01-app-shell.md).

## 3. Problem statement

Every authenticated page shares the same layout. It must work from 360 px phones (drawer) to desktops (fixed sidebar), in light and dark mode.

## 4. Expected behaviour

- ≥ 1024 px: sidebar always visible on the left.
- < 1024 px: sidebar hidden; ☰ in the top bar opens it as a drawer; clicking a link or the backdrop closes it.
- The active nav item is highlighted and has `aria-current="page"`.
- Theme toggle switches light/dark and remembers the choice after reload.

## 5. Relevant concepts

Standalone components, `input()`, `output()`, `model()`, `host` bindings, class bindings, `routerLink`/`routerLinkActive`, a signal service (`ThemeStore`).

## 6. Step-by-step implementation

1. Generate components:
   ```bash
   ng g c core/layout/shell --skip-tests
   ng g c core/layout/sidebar
   ng g c core/layout/nav-item
   ng g c core/layout/topbar
   ng g c shared/ui/page-header
   ng g s core/theme/theme-store
   ```
2. Define the nav model (**static for now**; later it comes from `/bootstrap`):
   ```typescript
   // core/layout/nav.ts
   export interface NavLink { label: string; route: string; icon: string; }
   export const MAIN_NAV: NavLink[] = [
     { label: 'Dashboard', route: '/dashboard', icon: '▦' },
     { label: 'Notes', route: '/notes', icon: '✎' },
     { label: 'Credentials', route: '/credentials', icon: '🔑' },
     { label: 'Tasks & Goals', route: '/tasks', icon: '☑' },
     { label: 'Favorites', route: '/favorites', icon: '☆' },
     { label: 'Trash', route: '/trash', icon: '🗑' },
   ];
   ```
   (Text glyphs stand in for icons until you add an icon set in Lab 11. Memories, Files and Calendar are v1.1 and omitted.)
3. Build `NavItem` (see [Module 02 note 3](../02-components-and-templates/03-class-style-attribute-binding.md#9-myspce-real-world-example--sidebar-nav-item)) with inputs `label`, `route`, `icon`.
4. Build `Sidebar` with `open = input(false)` and `closed = output<void>()`. It renders the logo, a user card (hard-coded "Pranjal Goyal" for now), `MAIN_NAV`, and a Settings link at the bottom.
5. Build `Topbar` with `menuClicked = output<void>()`, a search input (visual only for now), a theme toggle using `ThemeStore`, a bell button and an avatar.
6. Build `ThemeStore` (see [Module 04 note 3](../04-component-communication/03-communication-patterns.md#6-shared-signal-service-preview)).
7. Build `Shell` with `sidebarOpen = signal(false)`, a backdrop on mobile, and `<router-outlet />`.
8. Use `Shell` as the root route with two placeholder pages (`dashboard`, `notes`) so links work (proper routing comes in Lab 4):
   ```typescript
   // app.routes.ts
   export const routes: Routes = [
     { path: '', component: Shell, children: [
       { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
       { path: 'dashboard', loadComponent: () => import('./features/dashboard/dashboard-page') },
       { path: 'notes', loadComponent: () => import('./features/notes/notes-page') },
     ] },
   ];
   ```
   Each placeholder page uses `<app-page-header title="…" />`.
9. Remove the Lab 1 card from `app.html` (leave only `<router-outlet />`).

## 7. Complete working code

Full code for `Sidebar`, `Topbar`, `Shell`, `PageHeader` and `ThemeStore` is in the [solution](solutions/lab-02-solution.md).

## 8. Important lines explained

- `[class.translate-x-0]="open()"` + base `-translate-x-full lg:translate-x-0` — one element serves as drawer (mobile) and static sidebar (desktop).
- `(closed)` output — the sidebar doesn't own `sidebarOpen`; the shell does (single source of truth).
- `routerLinkActive` + `ariaCurrentWhenActive="page"` — visual and accessible active state.
- `effect()` in `ThemeStore` — syncs the signal to `<html class="dark">` and localStorage (a legitimate effect).

## 9. Expected output

Matches the design's layout: dark or light sidebar with active item highlighted in blue, content area with page header.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Sidebar overlaps content on desktop | missing `lg:static lg:translate-x-0` |
| Drawer doesn't close after navigating | close on link click (`(click)` on nav) or on `NavigationEnd` in the shell |
| Theme resets on reload | `ThemeStore` not reading localStorage at startup |
| `NG8001 'app-nav-item' is not a known element` | add `NavItem` to `Sidebar.imports` |

## 11. Test cases

```typescript
// core/layout/sidebar.spec.ts
it('renders every main nav link', async () => {
  TestBed.configureTestingModule({ providers: [provideRouter([])] });
  const f = TestBed.createComponent(Sidebar);
  await f.whenStable();
  const labels = [...f.nativeElement.querySelectorAll('nav a')].map((a: HTMLElement) => a.textContent?.trim());
  expect(labels).toEqual(expect.arrayContaining(['Dashboard', 'Notes', 'Credentials', 'Tasks & Goals']));
});

it('emits closed when the backdrop is clicked (shell)', async () => {
  TestBed.configureTestingModule({ providers: [provideRouter([])] });
  const f = TestBed.createComponent(Shell);
  (f.componentInstance as any).sidebarOpen.set(true);
  await f.whenStable();
  f.nativeElement.querySelector('[data-testid="backdrop"]').click();
  await f.whenStable();
  expect((f.componentInstance as any).sidebarOpen()).toBe(false);
});
```

## 12. Independent practice task

Build the **Quick Overview** section of the dashboard: an `OverviewTile` component and a grid of the eight tiles from the design (Notes 124, Credentials 28, Memories 48, Tasks 12, Files 68, Events 6, Favorites 23, Trash 4) using hard-coded data. 2 columns on mobile, 4 on desktop. Each tile links to its module and has an accessible name.

## 13. Solution

See [solutions/lab-02-solution.md](solutions/lab-02-solution.md).

## 14. Completion checklist

- [ ] Shell, Sidebar, NavItem, Topbar, PageHeader, ThemeStore built.
- [ ] Responsive drawer works at 360 px and static sidebar at 1280 px.
- [ ] Active link highlighted with `aria-current`.
- [ ] Theme persists across reloads.
- [ ] Tests pass; tagged `lab-02`.
