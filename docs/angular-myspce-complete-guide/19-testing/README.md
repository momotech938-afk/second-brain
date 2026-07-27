# Module 19 — Testing

How to test Myspce with the Angular 22 toolchain: **Vitest** for unit/integration tests (default since v21), **TestBed** for Angular wiring, and **Playwright** for end-to-end flows.

Back to [README](../README.md) · Previous: [Module 18](../18-ssr-and-hydration/README.md) · Next: [Module 20](../20-security-and-deployment/README.md)

## Tools for Angular 22

| Purpose | Tool | Notes |
|---|---|---|
| Unit/integration runner | **Vitest** via `@angular/build:unit-test` (`ng test`) | default for new projects since v21; jsdom/happy-dom or **Vitest Browser Mode** |
| Angular test harness | `TestBed`, `ComponentFixture`, `RouterTestingHarness`, `HttpTestingController` | built in |
| DOM queries (optional) | `@testing-library/angular` | user-centric queries (`getByRole`) |
| Mocks / timers | `vi.fn()`, `vi.spyOn`, `vi.useFakeTimers()` | |
| E2E | **Playwright** | not bundled; `npm init playwright@latest` |
| Accessibility | `axe-core` (`@axe-core/playwright`) | |
| Legacy | Karma + Jasmine | legacy path; the `migrate-karma-to-vitest` and `refactor-jasmine-vitest` schematics (improved in v22) migrate setup and specs |

The Angular CLI's own Jest and Web Test Runner experimental builders were removed in v22.

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Testing setup](01-testing-setup.md) | test pyramid, Vitest config, TestBed fundamentals, zoneless testing, `whenStable`, `TestBed.tick`, `getLastFixture` (v22), coverage |
| 2 | [Component tests](02-component-tests.md) | rendering, inputs/outputs, DOM assertions, signals and computed, pipes and directives, Testing Library, a11y checks |
| 3 | [Service and HTTP tests](03-service-http-tests.md) | services with fakes, stores, resources, `HttpTestingController`, interceptors, async and Observables |
| 4 | [Forms and routing tests](04-forms-routing-tests.md) | Signal/Reactive forms, validators, guards, resolvers, `RouterTestingHarness` |
| 5 | [E2E and CI](05-e2e-and-ci.md) | Playwright flows for login/notes/credentials, API mocking, a11y, CI pipeline, flaky tests |

**Lab:** [Lab 12 — Unit and integration tests](../21-myspce-practice-labs/lab-12-tests.md)

**Time:** ~5 h reading + 4 h practice.

## Self-assessment checklist

- [ ] I can run, watch and get coverage for tests with `ng test`.
- [ ] I can render a component, set inputs with `setInput`, and assert the DOM after `whenStable()`.
- [ ] I can test outputs, signals, computed values and effects.
- [ ] I can test a service with fake dependencies and HTTP code with `HttpTestingController`.
- [ ] I can test Signal Forms and Reactive Forms without the DOM.
- [ ] I can test guards and routed components with `RouterTestingHarness`.
- [ ] I can write a Playwright test for login → create note → find it.
- [ ] I can mock the API in Playwright and run an axe accessibility scan.
- [ ] I can run tests in CI and diagnose a flaky test.
