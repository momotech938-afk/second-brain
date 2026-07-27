# Angular 22 Cheat Sheet

One-page quick reference. Every line links back to a module if you need the full explanation.

Back to [README](README.md).

---

## CLI

```bash
npm install -g @angular/cli@22
ng new myspce --style=css --ssr=false     # zoneless + Vitest by default (v21+)
ng serve                                  # http://localhost:4200  (PORT=4300 ng serve works in v22)
ng build                                  # production build → dist/myspce/browser
ng test                                   # Vitest
ng generate component features/notes/note-card     # → note-card.ts/.html/.css, class NoteCard
ng generate service features/notes/data-access/notes-api   # @Service() by default in v22
ng generate service core/http/foo --injectable      # @Injectable() instead
ng generate guard core/auth/auth --functional
ng generate interceptor core/auth/auth
ng generate pipe shared/pipes/file-size
ng generate directive shared/directives/copy-to-clipboard
ng update @angular/cli @angular/core
```

## Component

```typescript
import { Component, ChangeDetectionStrategy, input, output, model, signal, computed } from '@angular/core';

@Component({
  selector: 'app-note-card',
  templateUrl: './note-card.html',
  imports: [DatePipe],
  changeDetection: ChangeDetectionStrategy.OnPush, // default in v22; optional
  host: { class: 'block', '[class.pinned]': 'note().pinned', '(click)': 'open.emit(note().id)' },
})
export class NoteCard {
  note = input.required<Note>();
  compact = input(false, { transform: booleanAttribute });
  open = output<string>();
  selected = model(false);
}
```

## Template syntax

| Syntax | Use |
|---|---|
| `{{ note().title }}` | interpolation |
| `[disabled]="saving()"` | property binding |
| `[attr.aria-label]="label()"` | attribute binding |
| `[class.active]="isActive()"` / `[class]="{ a: x, b: y }"` | class binding |
| `[style.width.%]="progress()"` | style binding |
| `(click)="save()"` / `(keyup.enter)="search()"` | event binding |
| `[(value)]="query"` | two-way binding (model / ngModel) |
| `#search` then `search.focus()` | template reference variable |
| `@let total = notes().length;` | local template variable |
| `@if (x) {} @else if (y) {} @else {}` | conditional |
| `@for (n of notes(); track n.id; let i = $index) {} @empty {}` | loop |
| `@switch (view()) { @case ('grid') {} @default {} }` | switch (`@default never;` for exhaustiveness) |
| `@defer (on viewport) {} @placeholder {} @loading {} @error {}` | lazy block |
| `<div /* comment */>` | comments inside elements (v22) |

## Signals

```typescript
const count = signal(0);           count(); count.set(1); count.update(n => n + 1);
const double = computed(() => count() * 2);
effect(() => localStorage.setItem('count', String(count())));
const page = linkedSignal(() => { filter(); return 1; });      // resets when filter changes
untracked(() => other());                                      // read without tracking
const ro = count.asReadonly();
```

## Resources (stable in v22)

```typescript
notes = httpResource<Paginated<Note>>(() => `/api/v1/notes?category=${this.category()}`);
notes.value(); notes.isLoading(); notes.error(); notes.hasValue(); notes.reload();

user = resource({ params: () => ({ id: this.id() }), loader: ({ params, abortSignal }) => fetchUser(params.id, abortSignal) });
tasks = rxResource({ params: () => this.view(), stream: ({ params }) => this.api.list(params) });
query = debounced(() => this.search(), 300);   // experimental in v22
```

## Dependency injection

```typescript
@Service()                                   // root singleton (v22)
export class NotesApi { private http = inject(HttpClient); }

export const API_URL = new InjectionToken<string>('API_URL');
providers: [{ provide: API_URL, useValue: '/api/v1' }]
// useClass | useValue | useFactory (deps via inject()) | useExisting
const report = injectAsync(() => import('./report').then(m => m.ReportService)); // v22
```

## Lifecycle and cleanup

`constructor` → `ngOnChanges` → `ngOnInit` → `ngDoCheck` → `ngAfterContentInit` → `ngAfterContentChecked` → `ngAfterViewInit` → `ngAfterViewChecked` → … → `ngOnDestroy`

```typescript
afterNextRender(() => initMap());           // browser only, once
afterEveryRender({ read: () => measure() });
inject(DestroyRef).onDestroy(() => clearInterval(id));
obs$.pipe(takeUntilDestroyed()).subscribe();
```

## Router

```typescript
provideRouter(routes, withComponentInputBinding(), withViewTransitions())
{ path: 'notes/:id', loadComponent: () => import('./note-detail'), canActivate: [authGuard], resolve: { note: noteResolver }, title: 'Note' }
{ path: 'admin', loadChildren: () => import('./admin/admin.routes'), canMatch: [roleGuard('admin')] }
{ path: '**', loadComponent: () => import('./not-found') }
inject(Router).navigate(['/notes', id], { queryParams: { tab: 'edit' } });
export const authGuard: CanActivateFn = () => inject(AuthStore).isLoggedIn() || inject(Router).parseUrl('/login');
```

v22: `paramsInheritanceStrategy` defaults to `'always'`; `CanMatchFn` gets a third `currentSnapshot` argument.

## Forms

```typescript
// Signal Forms (v22 stable)
model = signal({ email: '', password: '' });
loginForm = form(this.model, p => { required(p.email); email(p.email); minLength(p.password, 8); });
// <input [formField]="loginForm.email">   loginForm().invalid()   submit(this.loginForm, async v => …)

// Reactive Forms
fb = inject(NonNullableFormBuilder);
form = this.fb.group({ name: ['', [Validators.required]], tags: this.fb.array<string>([]) });
// <form [formGroup]="form"><input formControlName="name"></form>
```

## HTTP

```typescript
// HttpClient is available by default (v21+). Add provideHttpClient only to configure it:
provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))   // Fetch backend is default in v22
http.get<T>(url, { params: { page: 1 } }); http.post<T>(url, body); http.patch; http.delete;
export const authInterceptor: HttpInterceptorFn = (req, next) => next(req.clone({ setHeaders: { Authorization: `Bearer ${t}` } }));
```

## RxJS you'll use 90% of the time

`map` `filter` `tap` · `switchMap` (latest wins) `mergeMap` (parallel) `concatMap` (in order) `exhaustMap` (ignore while busy) · `debounceTime` `distinctUntilChanged` · `catchError` `retry` `finalize` · `combineLatest` `forkJoin` · `shareReplay({ bufferSize: 1, refCount: true })` · `toSignal(obs$)` `toObservable(sig)` `takeUntilDestroyed()`

## Testing (Vitest)

```typescript
TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
const fixture = TestBed.createComponent(NoteCard);
fixture.componentRef.setInput('note', mockNote);
await fixture.whenStable();
expect(fixture.nativeElement.textContent).toContain('Angular Notes');
TestBed.inject(HttpTestingController).expectOne('/api/v1/notes').flush(page);
```

## v22 changes to remember

| Change | Effect |
|---|---|
| OnPush is the default | `ChangeDetectionStrategy.Eager` = old default |
| `@Service()` | new root-service decorator, generated by default |
| Signal Forms, resources stable | use them in production |
| HttpClient uses Fetch | `withFetch()` deprecated; `withXhr()` for old behaviour; upload progress needs XHR |
| `strictTemplates` on by default | template type errors fail the build |
| Router `paramsInheritanceStrategy: 'always'` | child routes see parent params |
| TypeScript 6, Node 22+ | Node 20 dropped |
| Incremental hydration default (SSR) | `withIncrementalHydration()` deprecated |
