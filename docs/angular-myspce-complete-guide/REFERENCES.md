# References

The notes explain everything you need; these links are for checking details and going deeper. **Official docs always win** if a note and the docs disagree.

Back to [README](README.md).

## How the version baseline was verified (29 Oct 2026)

| Claim | Verified against |
|---|---|
| Angular 22.0 released June 2026; 22.1 in July 2026 | Ninja Squad, "What's new in Angular 22.0?" (3 Jun 2026) and its link to 22.1 (29 Jul 2026) |
| Angular 22 needs Node `^22.22.0 \|\| ^24.13.1 \|\| ^26.0.0`, TypeScript `>=6.0.0 <6.1.0`, RxJS `^6.5.3 \|\| ^7.4.0` | angular.dev version-compatibility table (next.angular.dev/reference/versions) |
| OnPush default, `ChangeDetectionStrategy.Eager`, migration | Ninja Squad 22.0; angular.love "Angular 22: Key Features and Changes"; InfoQ "Angular v22 released" |
| Signal Forms stable, `[formField]`, `form()`, `submit()`, `getError()`, `minDate`/`maxDate`, `when` option | Ninja Squad 22.0; angular.dev/essentials/signal-forms |
| `resource`, `rxResource`, `httpResource` stable; `chain()`; `debounced()` experimental | Ninja Squad 22.0 |
| `@Service()` decorator, `injectAsync()` | Ninja Squad 22.0 |
| HttpClient uses Fetch by default; `withFetch()` deprecated; `withXhr()`; `reportUploadProgress` | Ninja Squad 22.0 |
| `strictTemplates` default, comments in elements, `@default never(expr)` | Ninja Squad 22.0 |
| Router `paramsInheritanceStrategy: 'always'`, `CanMatchFn` third argument | Ninja Squad 22.0; InfoQ |
| Incremental hydration default | Ninja Squad 22.0 |
| Vitest default runner; `TestBed.getLastFixture()` | Ninja Squad 22.0 (Vitest default since v21) |

**Known limitation:** the angular.dev site itself was checked only for the version table and the Signal Forms essentials page. Other angular.dev links below are the standard documentation paths; if one moves, use the site search.

## Official Angular documentation

- Home and guides: https://angular.dev/
- Version compatibility: https://angular.dev/reference/versions
- Release schedule and support policy: https://angular.dev/reference/releases
- Update guide (step-by-step for any upgrade): https://angular.dev/update-guide
- Components: https://angular.dev/guide/components
- Templates and control flow: https://angular.dev/guide/templates
- Signals: https://angular.dev/guide/signals
- Resources: https://angular.dev/guide/signals/resource
- Dependency injection: https://angular.dev/guide/di
- Routing: https://angular.dev/guide/routing
- Forms overview: https://angular.dev/guide/forms
- Signal Forms essentials: https://angular.dev/essentials/signal-forms
- HTTP client: https://angular.dev/guide/http
- Testing: https://angular.dev/guide/testing
- Change detection / zoneless: https://angular.dev/guide/zoneless
- `@defer`: https://angular.dev/guide/templates/defer
- SSR and hydration: https://angular.dev/guide/ssr and https://angular.dev/guide/hydration
- Security: https://angular.dev/best-practices/security
- Style guide: https://angular.dev/style-guide
- CLI reference: https://angular.dev/cli
- API reference: https://angular.dev/api

## Release notes and changelogs

- Angular releases on GitHub: https://github.com/angular/angular/releases
- Angular CLI releases: https://github.com/angular/angular-cli/releases
- Ninja Squad "What's new" series (very precise per-release notes): https://blog.ninja-squad.com/

## Related libraries

- RxJS: https://rxjs.dev/ (operator decision tree: https://rxjs.dev/operator-decision-tree)
- TypeScript handbook: https://www.typescriptlang.org/docs/handbook/intro.html
- NgRx (Store, Signals): https://ngrx.io/
- Tailwind CSS with Angular: https://tailwindcss.com/docs/installation/framework-guides/angular
- Vitest: https://vitest.dev/
- Playwright: https://playwright.dev/
- Angular CDK: https://material.angular.dev/cdk/categories
- MDN Web Docs (HTML, CSS, DOM, Fetch, Web Crypto): https://developer.mozilla.org/
- OWASP Cheat Sheet Series (XSS, JWT, CSRF): https://cheatsheetseries.owasp.org/
- web.dev Core Web Vitals: https://web.dev/articles/vitals

## Articles used for v22 details

- Ninja Squad — What's new in Angular 22.0?: https://blog.ninja-squad.com/2026/06/03/what-is-new-angular-22.0
- angular.love — Angular 22: Key Features and Changes: https://angular.love/angular-22-key-features-and-changes
- InfoQ — Google Releases Angular v22: https://infoq.com/news/2026/08/angular-v22-released
- DEV — Angular v22: ChangeDetectionStrategy.Eager and OnPush by default: https://dev.to/lacolaco/angular-v22-introducing-changedetectionstrategyeager-and-onpush-by-default-4mof
- Ninja Squad — Angular Signal Forms part 1: https://blog.ninja-squad.com/2025/11/04/angular-signal-forms-part-1
