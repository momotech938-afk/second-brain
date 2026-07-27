# Lab 4 — Routing and Nested Layouts

Back to [Labs](README.md) · Solution: [lab-04-solution.md](solutions/lab-04-solution.md) · Previous: [Lab 3](lab-03-forms-and-controls.md) · Next: [Lab 5](lab-05-signal-ui-state.md)

## 1. Learning objective

Implement Myspce's full v1 route table: login outside the shell, lazy feature routes inside it, Settings tabs as child routes, a 404 page, consistent titles, and an unsaved-changes guard.

## 2. Prerequisites

[Module 10](../10-routing/README.md), [Lab 2](lab-02-layout-and-navigation.md), [Lab 3](lab-03-forms-and-controls.md).

## 3. Problem statement

URLs must be shareable and refresh-safe; each feature should load only when visited.

## 4. Expected behaviour

| URL | Result |
|---|---|
| `/` | redirects to `/dashboard` |
| `/login` | login page, **no** sidebar |
| `/notes`, `/notes/new`, `/notes/:id` | notes list / editor (placeholder) inside the shell |
| `/settings` | redirects to `/settings/profile`; tabs Profile, Appearance, Security, Customization, Backup |
| `/anything-else` | 404 page with a link home |
| Browser tab title | "Notes · MySpace", "Security · MySpace", … |
| Leaving `/notes/new` with typed text | "Discard changes?" confirmation |

## 5. Relevant concepts

`provideRouter` features, layout routes, `loadChildren`/`loadComponent`, `export default`, child routes, `TitleStrategy`, `CanDeactivateFn`, input binding.

## 6. Step-by-step implementation

1. Create feature route files with `export default` arrays:
   - `features/notes/notes.routes.ts` → `''`, `'new'`, `':id'`
   - `features/settings/settings.routes.ts` → layout with 5 children + redirect
   - `features/credentials/credentials.routes.ts`, `features/tasks/tasks.routes.ts` → `''` only for now
2. Create placeholder pages (each `export default class …` with a `PageHeader`). `NoteEditorPage` takes `id = input<string>()` and shows "New note" or "Edit note {{ id() }}".
3. Create `SettingsLayout` with a tab nav (`routerLink` + `routerLinkActive`) and `<router-outlet />`.
4. Create `shared/pages/not-found-page.ts`.
5. Write `app.routes.ts` (login outside, shell with children, `**` last).
6. Add `MySpaceTitleStrategy` and provide it.
7. Add `unsavedChangesGuard` using `window.confirm` for now (Lab 11 replaces it with `ConfirmService`); `NoteEditorPage` implements `hasUnsavedChanges()` based on a `draft` signal.
8. Run `ng build` and confirm each feature is a separate lazy chunk in the output.

## 7. Complete working code

`app.routes.ts`, feature route files, `SettingsLayout`, the title strategy and the guard are in the [solution](solutions/lab-04-solution.md).

## 8. Important lines explained

- `{ path: '', component: Shell, children: [...] }` — every child renders inside the shell's outlet; `/login` sits outside it.
- `pathMatch: 'full'` on empty-path redirects — otherwise every URL would redirect.
- `export default` in lazy files — `loadComponent: () => import('./x')` without `.then`.
- `title: 'Notes'` + `TitleStrategy` — consistent "· MySpace" suffix in one place.
- `canDeactivate` — the router asks the page before leaving.

## 9. Expected output

Navigating with the sidebar changes only the content area; the Network tab shows a new chunk the first time you visit each feature.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Sidebar shows on `/login` | login must be a sibling of the shell route, not a child |
| `/settings` shows an empty area | missing child redirect to `profile` |
| Every route goes to 404 | `**` placed before other routes |
| `id` input undefined | `withComponentInputBinding()` missing |
| Settings tab "Profile" always active | links are relative (`profile`), check `routerLinkActive` options |

## 11. Test cases

```typescript
it('redirects /settings to /settings/profile', async () => {
  TestBed.configureTestingModule({ providers: [provideRouter(routes, withComponentInputBinding())] });
  await RouterTestingHarness.create('/settings');
  expect(TestBed.inject(Router).url).toBe('/settings/profile');
});

it('binds the note id', async () => {
  TestBed.configureTestingModule({ providers: [provideRouter(routes, withComponentInputBinding())] });
  const harness = await RouterTestingHarness.create();
  await harness.navigateByUrl('/notes/01JABC');
  expect(harness.routeNativeElement?.textContent).toContain('Edit note 01JABC');
});
```

## 12. Independent practice task

Turn **Tasks & Goals** into a feature with child routes matching the design's tabs: `/tasks` (Tasks), `/tasks/goals`, `/tasks/habits`, using a `TasksLayout` with a tab bar. Habits is v1.1, so show a "Coming soon" placeholder, but keep the route.

## 13. Solution

See [solutions/lab-04-solution.md](solutions/lab-04-solution.md).

## 14. Completion checklist

- [ ] All URLs in the table behave as expected.
- [ ] Each feature is a lazy chunk.
- [ ] Titles consistent; unsaved guard works.
- [ ] Tasks tabs as child routes.
- [ ] Tests pass; tagged `lab-04`.
