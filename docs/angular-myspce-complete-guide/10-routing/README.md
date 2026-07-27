# Module 10 — Routing

URLs, pages, layouts, lazy loading and access control for every Myspce screen.

Back to [README](../README.md) · Previous: [Module 09](../09-lifecycle-and-change-detection/README.md) · Next: [Module 11](../11-template-driven-forms/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Router basics](01-router-basics.md) | `provideRouter`, route config, outlets, `routerLink`, redirects, wildcard/404, titles, matching rules |
| 2 | [Params and navigation](02-params-and-navigation.md) | route and query params, input binding, `ActivatedRoute`, programmatic navigation, navigation state, v22 param inheritance |
| 3 | [Nested layouts and lazy loading](03-nested-layouts-and-lazy-loading.md) | the app shell, child routes, `loadComponent`/`loadChildren`, feature route files, route providers, preloading |
| 4 | [Guards and resolvers](04-guards-and-resolvers.md) | `CanActivate`, `CanActivateChild`, `CanMatch`, `CanDeactivate`, resolvers, `RedirectCommand`, auth-aware navigation, testing |

**Lab:** [Lab 4 — Routing and nested layouts](../21-myspce-practice-labs/lab-04-routing-and-nested-layouts.md)

**Time:** ~4 h reading + 3 h practice.

> **Security reminder:** guards control what the **UI** shows. They are not security. The NestJS API must authorise every request on its own.

## Self-assessment checklist

- [ ] I can write Myspce's full route table with redirects, a 404 and titles.
- [ ] I can read route and query params as component inputs.
- [ ] I can navigate programmatically with query params and preserve/merge them.
- [ ] I can build the shell layout with nested routes and a login page outside it.
- [ ] I can lazy-load every feature and scope providers to a route.
- [ ] I can write functional `canActivate`, `canMatch` and `canDeactivate` guards.
- [ ] I can write a resolver and explain when not to use one.
- [ ] I can redirect unauthenticated users to `/login?returnUrl=…` and back after login.
- [ ] I can test guards and routed components with `RouterTestingHarness`.
