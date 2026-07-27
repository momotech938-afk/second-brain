# Route Params, Query Params and Navigation

Back to [Module 10](README.md) · Previous: [Router basics](01-router-basics.md) · Next: [Nested layouts and lazy loading](03-nested-layouts-and-lazy-loading.md)

## 1. Learning objectives

- Read path params, query params, fragments and route data — with input binding and with `ActivatedRoute`.
- Navigate programmatically with `Router.navigate` / `navigateByUrl`.
- Merge, preserve and remove query params.
- Pass navigation state.
- Understand v22's `paramsInheritanceStrategy: 'always'` default.

## 2. Prerequisites

[Router basics](01-router-basics.md), [Inputs](../04-component-communication/01-inputs.md).

## 3. What is it?

| Part | Example | Typical use |
|---|---|---|
| Path param | `/notes/01JABC` → `id = '01JABC'` | which entity |
| Query param | `/notes?category=work&page=2` | filters, sort, tabs, pagination |
| Fragment | `/settings#security` | in-page anchor |
| Route data | `data: { breadcrumb: 'Notes' }` | static config per route |
| Resolved data | `resolve: { note: noteResolver }` | pre-fetched entity |
| Navigation state | `router.navigate([...], { state: { from: 'dashboard' } })` | transient info not in the URL |

## 4. Why does Angular need it?

Putting filter state in the URL makes Myspce views **shareable, bookmarkable and refresh-safe**, and makes back/forward behave naturally.

## 5. How it works internally

When a route activates, the router builds an `ActivatedRoute` tree with Observables/snapshots of params, query params, data. With `withComponentInputBinding()`, after each navigation the router sets the routed component's **inputs** whose names match param/query/data/resolve keys.

**v22:** `paramsInheritanceStrategy` now defaults to `'always'`: child routes see **all** parent params and data, not only when the parent is component-less. That makes nested layouts simpler but means a child input named `id` could receive a parent's `id`. There's **no migration** for this — set `'emptyOnly'` explicitly if you depended on the old behaviour.

## 6. Syntax and API reference

### Input binding (recommended)

```typescript
// route: { path: ':id', loadComponent: …, resolve: { note: noteResolver }, data: { mode: 'edit' } }
export class NoteDetailPage {
  id = input.required<string>();          // path param
  tab = input<'view' | 'edit'>('view');   // ?tab=edit
  note = input.required<Note>();          // resolver result
  mode = input<string>();                 // route data
}
```

v22 lets you configure the binding: `withComponentInputBinding({ queryParams: true, unmatchedInputBehavior: 'undefinedIfStale' })`. `'undefinedIfStale'` only resets inputs that the router had previously set, so other inputs aren't clobbered with `undefined`.

### `ActivatedRoute` (needed in services/children or for Observables)

```typescript
const route = inject(ActivatedRoute);
route.snapshot.paramMap.get('id');                    // once
route.paramMap.pipe(map(p => p.get('id')));            // Observable over time
route.queryParamMap; route.data; route.fragment;

id = toSignal(inject(ActivatedRoute).paramMap.pipe(map(p => p.get('id')!)), { requireSync: true });
```

The router **reuses** a component when only params change (`/notes/A` → `/notes/B`). So reading `snapshot` once in `ngOnInit` is a classic bug; inputs and Observables stay up to date.

### Navigating

```typescript
const router = inject(Router);

router.navigate(['/notes', note.id]);                                   // /notes/01JABC
router.navigate(['/notes'], { queryParams: { category: 'work' } });     // replaces all query params
router.navigate([], { queryParams: { page: 2 }, queryParamsHandling: 'merge' });  // keep others
router.navigate([], { queryParams: { category: null }, queryParamsHandling: 'merge' });  // remove one
router.navigate(['../'], { relativeTo: inject(ActivatedRoute) });         // relative
router.navigateByUrl('/login?returnUrl=%2Fnotes');                        // string URL
router.navigate(['/notes'], { replaceUrl: true });                       // don't add history entry
router.navigate(['/credentials'], { state: { highlight: id } });         // history.state
router.parseUrl('/login');                                               // UrlTree (for guards)
```

Reading navigation state: `inject(Router).currentNavigation()?.extras.state` during navigation, or `history.state` afterwards.

v22: `<a [routerLink]="['/users', id]" browserUrl="/my-profile">` navigates to one URL while showing another in the address bar (niche).

## 7. Smallest working example

```typescript
// features/notes/pages/note-detail-page.ts
@Component({
  selector: 'app-note-detail-page',
  imports: [RouterLink],
  template: `
    <a routerLink="/notes" class="text-sm text-blue-600">← All notes</a>
    <h1 class="text-2xl font-bold">Note {{ id() }}</h1>
    <nav class="mt-2 flex gap-2">
      <a [routerLink]="[]" [queryParams]="{ tab: 'view' }" [class.font-bold]="tab() === 'view'">View</a>
      <a [routerLink]="[]" [queryParams]="{ tab: 'edit' }" [class.font-bold]="tab() === 'edit'">Edit</a>
    </nav>
  `,
})
export default class NoteDetailPage {
  id = input.required<string>();
  tab = input<'view' | 'edit'>('view');
}
```

## 8. Step-by-step

1. `/notes/01JABC?tab=edit` sets `id = '01JABC'`, `tab = 'edit'`.
2. `[routerLink]="[]"` + `queryParams` changes only the query string on the current route.
3. The component instance is reused; inputs update; template re-renders.

## 9. Myspce real-world example — Notes filters in the URL

```typescript
// features/notes/pages/notes-page.ts
export default class NotesPage {
  private router = inject(Router);

  category = input('all');
  q = input('');
  sort = input<'-updatedAt' | 'title'>('-updatedAt');
  page = input(1, { transform: numberAttribute });

  setFilter(changes: Partial<{ category: string; q: string; sort: string; page: number }>) {
    const resetsPage = 'category' in changes || 'q' in changes || 'sort' in changes;
    this.router.navigate([], {
      queryParams: {
        ...changes,
        ...(changes.category === 'all' ? { category: null } : {}),
        ...(changes.q === '' ? { q: null } : {}),
        ...(resetsPage ? { page: null } : {}),
      },
      queryParamsHandling: 'merge',
      replaceUrl: 'q' in changes,     // typing shouldn't create 20 history entries
    });
  }
}
```

The page's `httpResource` reads these inputs ([Module 08](../08-signals-and-reactivity/04-resources.md)), so the URL **is** the state. With URL-driven page numbers you no longer need `linkedSignal` for resetting — `setFilter` resets `page` in the URL.

For the search box, debounce before navigating (RxJS `debounceTime` or v22 `debounced()`), otherwise every keystroke is a navigation.

## 10. Advanced example — return to where you came from after login

```typescript
// core/auth/auth-guard.ts (excerpt)
export const authGuard: CanActivateFn = (_route, state) =>
  inject(AuthStore).isLoggedIn() || inject(Router).createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });

// features/auth/login-page.ts (excerpt)
returnUrl = input<string>('/dashboard');
async onLoggedIn() {
  const target = this.returnUrl().startsWith('/') && !this.returnUrl().startsWith('//') ? this.returnUrl() : '/dashboard';
  await this.router.navigateByUrl(target, { replaceUrl: true });
}
```

The `startsWith('/')` check prevents **open redirects** (`?returnUrl=https://evil.com` or `//evil.com`).

## 11. Visual explanation

```text
/notes?category=work&page=2
        │                │
        └─ input category └─ input page
setFilter({ category: 'finance' }) → /notes?category=finance   (page removed → defaults to 1)
setFilter({ page: 3 })             → /notes?category=finance&page=3
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `route.snapshot.paramMap` read once in `ngOnInit` | inputs or `paramMap` Observable |
| `navigate([], { queryParams })` wiping other params | `queryParamsHandling: 'merge'` |
| Query param numbers arriving as strings | `numberAttribute` transform |
| Trusting `returnUrl` blindly | validate it's a relative path |
| Every keystroke pushes history | `replaceUrl: true` and debounce |
| Child input unexpectedly receiving a parent param (v22) | rename, or set `paramsInheritanceStrategy: 'emptyOnly'` |

## 13. Debugging guide

- Input stays `undefined` → `withComponentInputBinding()` missing, or names don't match, or the component isn't the routed one (input binding only applies to routed components).
- Query params are strings: always transform/validate.
- `withDebugTracing()` shows the resolved params for each navigation.

## 14. Best practices

- URL for anything shareable or navigational; signals for ephemeral UI.
- Input binding for routed components; `ActivatedRoute` elsewhere.
- Validate and transform all params; they're user input.

## 15. Angular vs React

`input()` from route params ≈ `useParams()`/`useSearchParams()` but typed and reactive without hooks. `queryParamsHandling: 'merge'` ≈ `setSearchParams(prev => …)`.

## 16. Testing

```typescript
it('binds the id param as an input', async () => {
  TestBed.configureTestingModule({
    providers: [provideRouter([{ path: 'notes/:id', component: NoteDetailPage }], withComponentInputBinding())],
  });
  const harness = await RouterTestingHarness.create();
  const page = await harness.navigateByUrl('/notes/01JABC', NoteDetailPage);
  expect(page.id()).toBe('01JABC');
});
```

## 17. Practice questions

1. Add `sort` and `favorite` query params to Notes with defaults.
2. Implement Credentials `?open=:id` that opens a detail drawer.
3. Why is reading `snapshot` in `ngOnInit` a bug when navigating between two notes?

## 18. Interview questions

- **Q: How do you read query params in a routed component?** With `withComponentInputBinding()`, declare an `input()` with the same name; otherwise use `ActivatedRoute.queryParamMap`.
- **Q: What does `queryParamsHandling: 'merge'` do?** Keeps existing query params and applies the new ones (null removes a key).
- **Q: What changed for params in v22?** `paramsInheritanceStrategy` defaults to `'always'`, so children inherit all parent params and data.

## 19. Revision notes

path params `:id` · query params for filters · input binding · `ActivatedRoute` Observables · `navigate([], { queryParams, queryParamsHandling: 'merge' })` · `null` removes · validate `returnUrl` · v22 inheritance `'always'`.

## 20. Completion checklist

- [ ] Notes filters live in the URL.
- [ ] Login returns to the original page safely.
- [ ] I tested param binding with `RouterTestingHarness`.
