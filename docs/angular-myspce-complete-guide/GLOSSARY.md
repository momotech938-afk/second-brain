# Glossary

Angular terms in plain English, alphabetically. Each entry links to the module that teaches it.

Back to [README](README.md).

| Term | Meaning | Module |
|---|---|---|
| **AOT (Ahead-of-Time) compilation** | Angular compiles templates to JavaScript at build time, not in the browser. Errors show up during `ng build`, and the app ships less code. | [01](01-angular-fundamentals/01-architecture-mental-model.md) |
| **Application config** | The `ApplicationConfig` object in `app.config.ts` that lists app-wide providers (router, HTTP, error listeners). | [01](01-angular-fundamentals/03-project-structure-and-bootstrap.md) |
| **Attribute directive** | A directive that changes how an existing element looks or behaves, e.g. a `copyToClipboard` directive. | [05](05-directives/02-custom-attribute-directives.md) |
| **Binding** | A link between the component class and the template: `{{ }}`, `[prop]`, `(event)`, `[(model)]`. | [02](02-components-and-templates/02-data-binding.md) |
| **Bootstrap** | Starting the app: `bootstrapApplication(App, appConfig)` in `main.ts`. | [01](01-angular-fundamentals/03-project-structure-and-bootstrap.md) |
| **Change detection** | How Angular decides which parts of the DOM to update after state changes. | [09](09-lifecycle-and-change-detection/02-change-detection.md) |
| **Component** | A class + template + styles that render one piece of UI. The basic building block. | [02](02-components-and-templates/01-components.md) |
| **`computed()`** | A read-only signal derived from other signals. Cached; recalculates only when its dependencies change. | [08](08-signals-and-reactivity/02-computed-and-effect.md) |
| **Content projection** | Passing markup from a parent into a child's `<ng-content>` slot (like React `children`). | [16](16-advanced-components/01-content-projection.md) |
| **ControlValueAccessor (CVA)** | The interface that lets a custom component work with Reactive/template-driven forms. | [12](12-reactive-forms/03-dynamic-and-reusable-forms.md) |
| **`@defer`** | Template block that lazy-loads its content on a trigger (viewport, idle, interaction). | [17](17-performance/02-loading-and-bundles.md) |
| **Dependency injection (DI)** | Angular creates objects for you and hands them to the code that asks for them with `inject()`. | [07](07-services-and-dependency-injection/01-services-and-inject.md) |
| **DestroyRef** | An injectable that lets you register cleanup code for when a component/service is destroyed. | [09](09-lifecycle-and-change-detection/03-cleanup-and-memory-leaks.md) |
| **Directive** | A class that adds behaviour to elements. Components are directives with templates. | [05](05-directives/README.md) |
| **`effect()`** | Runs a side effect (logging, localStorage, DOM sync) whenever signals it reads change. Not for deriving state. | [08](08-signals-and-reactivity/02-computed-and-effect.md) |
| **Eager (change detection)** | The old "check always" strategy, previously called `Default`. In v22 you must opt into it; OnPush is the default. | [09](09-lifecycle-and-change-detection/02-change-detection.md) |
| **Environment injector** | An injector created for the app or a lazy route that holds services not tied to a component. | [07](07-services-and-dependency-injection/03-injector-hierarchy.md) |
| **FieldTree** | The object returned by Signal Forms' `form()`. Each property is a field you can bind with `[formField]`. | [12](12-reactive-forms/04-signal-forms.md) |
| **FormField directive** | Signal Forms directive (`[formField]="loginForm.email"`) that binds an input to a field. | [12](12-reactive-forms/04-signal-forms.md) |
| **Guard** | A function the router calls to decide if navigation may continue (`CanActivateFn`, `CanMatchFn`…). UX only; the server must still check. | [10](10-routing/04-guards-and-resolvers.md) |
| **Host element** | The DOM element a component or directive is attached to, e.g. `<app-note-card>`. | [05](05-directives/02-custom-attribute-directives.md) |
| **Hydration** | Reusing server-rendered HTML on the client instead of re-creating it. | [18](18-ssr-and-hydration/01-ssr-concepts.md) |
| **`httpResource()`** | A resource that runs an HTTP GET whenever its reactive URL/request changes. Stable in v22. | [08](08-signals-and-reactivity/04-resources.md) |
| **Injection context** | A place where `inject()` is allowed: field initializers, constructors, factory functions, `runInInjectionContext`. | [07](07-services-and-dependency-injection/01-services-and-inject.md) |
| **InjectionToken** | A typed key for injecting values that are not classes (API URL, feature flags). | [07](07-services-and-dependency-injection/02-providers-and-tokens.md) |
| **`input()`** | Signal-based component input. `input.required<T>()` makes it mandatory. | [04](04-component-communication/01-inputs.md) |
| **Interceptor** | A function that sees every HTTP request/response: auth headers, errors, logging. | [13](13-http-and-api-integration/02-interceptors-and-auth.md) |
| **Lazy loading** | Loading a route's code only when the user navigates there (`loadComponent`, `loadChildren`). | [10](10-routing/03-nested-layouts-and-lazy-loading.md) |
| **Lifecycle hook** | A method Angular calls at a stage of a component's life (`ngOnInit`, `ngOnDestroy`…). | [09](09-lifecycle-and-change-detection/01-lifecycle-hooks.md) |
| **`linkedSignal()`** | A writable signal that resets itself when a source signal changes. | [08](08-signals-and-reactivity/03-linked-signal.md) |
| **`model()`** | A two-way bindable input: parent uses `[(value)]`, child calls `value.set()`. | [04](04-component-communication/02-outputs-and-model.md) |
| **NgModule** | The pre-standalone way of grouping components. Legacy; still found in older codebases. | [01](01-angular-fundamentals/01-architecture-mental-model.md) |
| **Observable** | A stream of values over time from RxJS. HttpClient returns Observables. | [14](14-rxjs/01-observables.md) |
| **OnPush** | Change-detection strategy where a component is checked only when inputs change, an event fires inside it, or a signal it reads changes. **Default in v22.** | [09](09-lifecycle-and-change-detection/02-change-detection.md) |
| **`output()`** | Declares a custom event a child emits to its parent. | [04](04-component-communication/02-outputs-and-model.md) |
| **Pipe** | A template function that formats a value: `{{ size \| fileSize }}`. | [06](06-pipes/README.md) |
| **Provider** | A rule telling DI how to create something: `useClass`, `useValue`, `useFactory`, `useExisting`. | [07](07-services-and-dependency-injection/02-providers-and-tokens.md) |
| **Resolver** | A function that loads data before a route activates. | [10](10-routing/04-guards-and-resolvers.md) |
| **Resource** | A signal-based wrapper around async work with `value()`, `isLoading()`, `error()`, `status()`. | [08](08-signals-and-reactivity/04-resources.md) |
| **Router outlet** | `<router-outlet />` — the spot where the active route's component renders. | [10](10-routing/01-router-basics.md) |
| **`@Service()`** | v22 decorator for a root-provided service. Equivalent to `@Injectable({ providedIn: 'root' })`; dependencies must use `inject()`. | [07](07-services-and-dependency-injection/01-services-and-inject.md) |
| **Signal** | A reactive value container. Read by calling it (`count()`); write with `set`/`update`. | [08](08-signals-and-reactivity/01-signal-basics.md) |
| **Signal Forms** | v21+ forms API built on signals (`form()`, `[formField]`, `submit()`). Stable in v22. | [12](12-reactive-forms/04-signal-forms.md) |
| **SSR** | Server-side rendering: HTML generated on the server per request. | [18](18-ssr-and-hydration/01-ssr-concepts.md) |
| **Standalone component** | A component that declares its own `imports` and needs no NgModule. The default since v19. | [02](02-components-and-templates/01-components.md) |
| **Strict templates** | Full type checking of templates. On by default in v22. | [01](01-angular-fundamentals/04-configuration-and-builds.md) |
| **Structural directive** | A directive that adds/removes DOM, using `<ng-template>` (`*appHasRole`). Mostly replaced by `@if`/`@for` for built-ins. | [05](05-directives/03-structural-and-host-directives.md) |
| **TestBed** | Angular's testing utility that creates components and services with DI in tests. | [19](19-testing/01-testing-setup.md) |
| **Template reference variable** | `#search` on an element, usable elsewhere in the template. | [02](02-components-and-templates/02-data-binding.md) |
| **`track`** | Required expression in `@for` that tells Angular how to identify each item (`track note.id`). | [03](03-template-control-flow/01-if-for-switch.md) |
| **View** | The rendered DOM for a component's template, managed by Angular. | [09](09-lifecycle-and-change-detection/02-change-detection.md) |
| **Vitest** | The default unit-test runner for new Angular projects since v21. | [19](19-testing/01-testing-setup.md) |
| **Zone.js** | Library that patches async APIs so Angular knows when to run change detection. New apps are zoneless. | [09](09-lifecycle-and-change-detection/02-change-detection.md) |
| **Zoneless** | Angular without Zone.js; signals, events and `markForCheck` schedule rendering. Default for new apps since v21. | [09](09-lifecycle-and-change-detection/02-change-detection.md) |
