# Concept Checklist

Back to [Module 22](README.md) · Next: [Practical assessments](02-practical-assessments.md)

Answer each item **out loud or in writing, without notes**. Mark ✅ / 🟡 / ❌. Each group links to the module to revisit.

## Foundations — [Module 00](../00-prerequisites/README.md), [Module 01](../01-angular-fundamentals/README.md)

- [ ] Explain `const`/`let`, closures, `this` in arrow vs regular functions, and spread/destructuring.
- [ ] Write a generic function with a constraint (`<T extends { id: string }>`).
- [ ] Explain `unknown` vs `any`, type narrowing, discriminated unions, `satisfies`.
- [ ] Use `Partial`, `Pick`, `Omit`, `Record`, `ReturnType` correctly.
- [ ] Explain the event loop: microtasks vs macrotasks; why `await` doesn't block.
- [ ] Trace the bootstrap: `main.ts → bootstrapApplication → app.config.ts → App`.
- [ ] Name what Angular 22 generates by default: standalone, OnPush default, zoneless, Vitest, strict templates.
- [ ] Explain the v20+ file naming (`note-card.ts`, class `NoteCard`).
- [ ] Explain build configurations, `fileReplacements` and budgets.
- [ ] Compare an Angular component with a React component (DI, templates, change detection).

## Components and templates — [Modules 02–06](../02-components-and-templates/README.md)

- [ ] Write a standalone component with `input()`, `input.required()`, `output()` and `model()`.
- [ ] Explain interpolation, property, attribute, class, style and event binding, and when to use `[attr.x]`.
- [ ] Use `@if … as`, `@for` with `track` and `@empty`, `@switch`, `@let`.
- [ ] Explain why `track $index` is wrong for editable lists.
- [ ] Choose between parent→child inputs, outputs, `model()`, a shared service, and a store.
- [ ] Write an attribute directive with `host` bindings and an input.
- [ ] Write a structural directive with `TemplateRef` + `ViewContainerRef`.
- [ ] Explain host directives and when to use `hostDirectives`.
- [ ] Write a pure custom pipe; explain why impure pipes are rare.
- [ ] Format dates, currency and numbers for `en-IN` with built-in pipes.

## Services and DI — [Module 07](../07-services-and-dependency-injection/README.md)

- [ ] Create a root service with `@Service()`; explain how it differs from `@Injectable({ providedIn: 'root' })`.
- [ ] Use `inject()` and explain where it can be called (injection context).
- [ ] Create and provide an `InjectionToken` (`API_URL`).
- [ ] Explain `useClass`, `useValue`, `useFactory`, `useExisting`.
- [ ] Explain the environment vs element injector hierarchy; when to provide in a component.
- [ ] Explain `injectAsync()` and when lazy service loading helps.
- [ ] Design the Myspce API layer: models → API services → stores → pages.

## Signals and reactivity — [Module 08](../08-signals-and-reactivity/README.md)

- [ ] Use `signal`, `set`, `update`; explain why you must not mutate signal values.
- [ ] Use `computed`; explain memoisation and dynamic dependency tracking.
- [ ] Explain when `effect` is appropriate (sync to non-Angular world) and when it is a smell.
- [ ] Use `linkedSignal` in both short and `{ source, computation }` forms.
- [ ] Use `resource`, `rxResource`, `httpResource`: params, status, `hasValue()`, `reload()`, `set()`.
- [ ] Explain how resources cancel stale requests.
- [ ] Choose signals vs RxJS for a given problem (state vs events over time).
- [ ] Use `toSignal` / `toObservable` and explain their injection-context requirements.

## Lifecycle and change detection — [Module 09](../09-lifecycle-and-change-detection/README.md)

- [ ] List the lifecycle hooks still worth using with signals.
- [ ] Use `afterNextRender` / `afterEveryRender` for DOM work.
- [ ] Explain OnPush (default in v22) and `ChangeDetectionStrategy.Eager`.
- [ ] Explain zoneless change detection: what schedules a render.
- [ ] Clean up with `DestroyRef`, `takeUntilDestroyed`; spot three common leaks.

## Routing — [Module 10](../10-routing/README.md)

- [ ] Configure routes with layouts, redirects, `**` and lazy `loadComponent`/`loadChildren`.
- [ ] Bind route params, query params and data to inputs with `withComponentInputBinding()`.
- [ ] Navigate with `routerLink`, `Router.navigate`, `queryParamsHandling`, `replaceUrl`.
- [ ] Write `CanActivateFn`, `CanMatchFn`, `CanDeactivateFn`; return `UrlTree`s.
- [ ] Explain `canMatch` vs `canActivate` for role-gated lazy routes.
- [ ] Explain `paramsInheritanceStrategy: 'always'` (the v22 default).
- [ ] Use a custom `TitleStrategy`.

## Forms — [Modules 11–12](../12-reactive-forms/README.md)

- [ ] Build a template-driven form with `ngModel` and explain when it's enough.
- [ ] Build a typed reactive form with `FormGroup`/`FormControl`/`NonNullableFormBuilder`.
- [ ] Write sync, cross-field and async validators.
- [ ] Build a `FormArray` UI (add/remove rows).
- [ ] Implement `ControlValueAccessor` for a custom control.
- [ ] Build a Signal Form: `form()`, `[formField]`, rules with messages, `validate`, `submit`.
- [ ] Use `FormRoot` with `submission: { action, onInvalid }`.
- [ ] Implement `FormValueControl` (`value` model, `touched` input, `touch` output).
- [ ] Return server errors from a submit action and show them on fields or the form.
- [ ] Make forms accessible: labels, `aria-invalid`, `aria-describedby`, focus on first error.

## HTTP — [Module 13](../13-http-and-api-integration/README.md)

- [ ] Call typed `get/post/patch/delete`; build `HttpParams` safely.
- [ ] Explain Fetch as the v22 default transport and when `withXhr()` is needed (upload progress).
- [ ] Write functional interceptors; explain their order.
- [ ] Use `HttpContextToken`s (`SKIP_AUTH`, `SKIP_ERROR_TOAST`).
- [ ] Implement single-flight token refresh and explain why it matters with rotation.
- [ ] Map errors to user messages (`toMessage`); decide toast vs inline.
- [ ] Decide which requests are safe to retry.
- [ ] Test HTTP with `HttpTestingController`.

## RxJS — [Module 14](../14-rxjs/README.md)

- [ ] Explain cold vs hot observables and unicast vs multicast.
- [ ] Choose between `switchMap`, `mergeMap`, `concatMap`, `exhaustMap` for a given scenario.
- [ ] Use `debounceTime`, `distinctUntilChanged`, `catchError`, `retry`, `shareReplay`, `finalize`.
- [ ] Explain `Subject`, `BehaviorSubject`, `ReplaySubject` and when a signal is better.
- [ ] Test an observable with fake timers.

## State management — [Module 15](../15-state-management/README.md)

- [ ] Classify state: server cache, UI state, URL state, form state, session state.
- [ ] Build a signal store service with private writable and public readonly signals.
- [ ] Implement optimistic updates with rollback.
- [ ] Explain when NgRx (Store or SignalStore) would be worth adding to Myspce — and why it probably isn't needed for v1.

## Advanced components — [Module 16](../16-advanced-components/README.md)

- [ ] Use `ng-content` with `select` and explain projection limits.
- [ ] Use `viewChild`, `viewChildren`, `contentChild`, `contentChildren` (signal queries).
- [ ] Create components dynamically (`ViewContainerRef.createComponent`, `NgComponentOutlet`).
- [ ] Build an accessible modal on native `<dialog>`.
- [ ] Build a promise-based confirm service and a toast service with Undo.

## Performance — [Module 17](../17-performance/README.md)

- [ ] Use `@defer` triggers, `prefetch`, `@placeholder`, `@loading`, `@error`.
- [ ] Read a bundle analysis and fix one real finding.
- [ ] Use `NgOptimizedImage` correctly.
- [ ] Profile with Angular DevTools and explain a slow render.
- [ ] Explain LCP, INP, CLS and how Angular code affects each.

## SSR — [Module 18](../18-ssr-and-hydration/README.md)

- [ ] Explain CSR vs SSR vs SSG and why Myspce v1 is a CSR SPA (assumption A1).
- [ ] Explain hydration and incremental hydration (default in v22 for SSR apps).
- [ ] Write platform-safe code (`afterNextRender`, `DOCUMENT`, no direct `window` at construction).

## Testing — [Module 19](../19-testing/README.md)

- [ ] Configure TestBed for a zoneless component test; use `whenStable()`.
- [ ] Test inputs, outputs and DOM by behaviour.
- [ ] Fake a store/API per test.
- [ ] Test guards and routes with `RouterTestingHarness`.
- [ ] Test `@defer` blocks with `DeferBlockBehavior.Manual`.
- [ ] Write a Playwright test with `page.route` API mocking.

## Security, a11y, deployment — [Module 20](../20-security-and-deployment/README.md)

- [ ] Explain Angular's XSS protection and the risk of `bypassSecurityTrust*`.
- [ ] Explain why nothing secret belongs in `environment.ts`.
- [ ] Justify access token in memory + refresh token in httpOnly cookie; explain CSRF implications.
- [ ] Explain why guards are not authorisation.
- [ ] Handle secrets in the UI safely (reveal timeout, no logs, no toasts with secrets).
- [ ] Make a page keyboard-operable with visible focus and correct ARIA.
- [ ] Configure nginx for SPA fallback, caching and security headers.
- [ ] Diagnose blank page, 404 on refresh, and `ChunkLoadError` in production.

## Scoring

| ✅ count (of 101) | Meaning |
|---|---|
| ≥ 90 and 0 ❌ | concepts solid → proceed |
| 75–89 | revisit 🟡 items, then proceed |
| < 75 or any ❌ in Signals, Routing, Forms or HTTP | repeat those modules' labs first |
