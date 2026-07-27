# Testing Setup and TestBed Fundamentals

Back to [Module 19](README.md) · Next: [Component tests](02-component-tests.md)

## 1. Learning objectives

- Understand unit vs integration vs E2E tests and what to test at each level.
- Run and configure Vitest through the Angular CLI.
- Use `TestBed` to create components and services with DI.
- Handle async rendering in zoneless tests.
- Measure coverage and keep tests isolated.

## 2. The test pyramid for Myspce

```text
          ▲  E2E (Playwright)           ~10 tests: login, notes CRUD, credential reveal, settings theme
         ▲▲  Integration (TestBed+HTTP) ~dozens: pages with stores + HttpTestingController, guards with router
        ▲▲▲  Unit (plain/TestBed)       ~hundreds: pipes, validators, stores, utils, presentational components
```

| Level | Tests | Speed | Confidence |
|---|---|---|---|
| Unit | a function/class/component alone | ms | narrow |
| Integration | several units with real Angular wiring (DI, templates, router, HttpClient with test backend) | 10–100 ms | good |
| E2E | the real app in a real browser | seconds | highest |

## 3. Running tests

```bash
ng test                      # watch mode locally
ng test --no-watch           # single run (CI)
ng test --coverage           # coverage report (coverage/)
ng test --include "src/app/features/notes/**/*.spec.ts"
```

v22 `unit-test` builder options include `--quiet` (default `true` locally, `false` in CI) and `--isolate` (run tests in isolated workers).

`angular.json` (generated):

```json
"test": {
  "builder": "@angular/build:unit-test",
  "options": { "tsConfig": "tsconfig.spec.json", "buildTarget": "::development", "runner": "vitest" }
}
```

Exact option names may vary slightly between minor CLI versions — check `ng test --help`. You don't need a separate `vitest.config.ts` for typical setups; the builder configures Vitest. For **Vitest Browser Mode** (real Chromium instead of jsdom), install a provider (e.g. `@vitest/browser` + Playwright) and set the builder's browsers option.

## 4. TestBed fundamentals

`TestBed` creates a test **environment injector** (like `app.config.ts` for one test).

```typescript
import { TestBed } from '@angular/core/testing';

beforeEach(() => {
  TestBed.configureTestingModule({
    imports: [],                  // rarely needed for standalone components
    providers: [                  // like app.config.ts providers
      provideRouter([]),
      provideHttpClient(),
      provideHttpClientTesting(),
      { provide: API_URL, useValue: '/api/v1' },
      { provide: NotesApi, useValue: fakeNotesApi() },
    ],
  });
});

const service = TestBed.inject(NotesStore);          // services
const fixture = TestBed.createComponent(NotesPage);  // components
const fixture2 = TestBed.getLastFixture();           // v22: last created fixture (e.g. from beforeEach)
TestBed.runInInjectionContext(() => authGuard(route, state));   // functions that use inject()
TestBed.tick();                                      // flush effects + change detection synchronously
TestBed.overrideComponent(NotesPage, { set: { imports: [FakeNoteCard] } });   // swap child components
```

`TestBed` is reset automatically before each test.

### `ComponentFixture`

| API | Use |
|---|---|
| `fixture.componentInstance` | the component class instance |
| `fixture.componentRef.setInput('name', value)` | set inputs (goes through Angular) |
| `fixture.nativeElement` | host DOM element |
| `fixture.debugElement.query(By.css('…'))` | Angular-aware DOM querying |
| `await fixture.whenStable()` | wait for Angular's scheduler (rendering, effects, resources) |
| `fixture.detectChanges()` | run change detection now (also runs `checkNoChanges` in dev mode) |
| `fixture.destroy()` | destroy (test cleanup logic) |

### Zoneless tests (the default for Angular 22 apps)

There's no Zone.js to "auto-detect" async work. Rules:

1. After changing inputs or state, `await fixture.whenStable()` before asserting the DOM.
2. For effects/resources in services, `TestBed.tick()` (or `await TestBed.inject(ApplicationRef).whenStable()`).
3. Timers: `vi.useFakeTimers()` + `await vi.advanceTimersByTimeAsync(ms)`, then `await fixture.whenStable()`.
4. `fakeAsync`/`tick` from `@angular/core/testing` need Zone.js; with v22 you can enable them under Vitest by adding `zone.js/plugins/vitest-patch` to the test polyfills — only for legacy tests.

## 5. Smallest working example

```typescript
// shared/pipes/file-size.spec.ts — no TestBed needed
import { describe, it, expect } from 'vitest';
import { FileSizePipe } from './file-size';

describe('FileSizePipe', () => {
  it('formats megabytes', () => expect(new FileSizePipe().transform(1_258_291)).toBe('1.2 MB'));
});
```

Vitest globals (`describe`, `it`, `expect`, `vi`) are usually available without imports in Angular's setup; importing them explicitly is harmless.

## 6. Myspce example — a test helper file

```typescript
// src/testing/test-providers.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideHttpClientTesting } from '@angular/common/http/testing';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { API_URL } from '../app/core/http/api-url.token';

export function provideTestDefaults(...extra: unknown[]) {
  return [
    provideRouter([], withComponentInputBinding()),
    provideHttpClient(withInterceptors([])),
    provideHttpClientTesting(),
    { provide: API_URL, useValue: '/api/v1' },
    ...extra,
  ];
}
```

```typescript
// src/testing/mock-data.ts — shared fixtures that mirror MYSPCE_PROJECT_CONTEXT models
export const MOCK_NOTES: Note[] = [
  { id: 'n1', title: 'Angular Notes', preview: 'Components, Services, Routing, Best Practices…', color: '#fee2e2', pinned: false, isFavorite: true,
    category: { id: 'c1', module: 'note', name: 'Work', slug: 'work', color: '#3b82f6' }, tags: ['angular'], createdAt: '2026-10-25T10:00:00Z', updatedAt: '2026-10-28T10:00:00Z' },
  { id: 'n2', title: 'AWS Deployment', preview: 'ECS, RDS, CI/CD steps and commands', color: '#ffedd5', pinned: true, isFavorite: false,
    category: { id: 'c2', module: 'note', name: 'DevOps', slug: 'devops', color: '#f97316' }, tags: ['aws'], createdAt: '2026-10-25T10:00:00Z', updatedAt: '2026-10-25T10:00:00Z' },
];
```

## 7. Isolation and maintainability

- One behaviour per test; name it as a sentence: `it('shows the empty state when there are no notes')`.
- Arrange–Act–Assert structure.
- Fake **boundaries** (API services, `NOW`, storage), not internals.
- Assert on what users see (text, roles) rather than private fields where practical.
- Shared fixtures in `src/testing/`, never copy-pasted.

## 8. Coverage

```bash
ng test --no-watch --coverage
```

Targets that make sense for Myspce: stores, validators, pipes, guards, interceptors ≥ 90%; components ≥ 70%; don't chase 100% on templates. Coverage tells you what **isn't** tested, not that what's tested is correct.

## 9. Common mistakes

| Mistake | Fix |
|---|---|
| Asserting the DOM right after `setInput` | `await fixture.whenStable()` |
| Setting inputs via `componentInstance.note = …` | `componentRef.setInput('note', …)` (signal inputs are read-only) |
| Real HTTP in unit tests | `provideHttpClientTesting()` |
| Tests depending on the current date | inject a `NOW` token |
| Huge `beforeEach` setup shared by unrelated tests | small helpers, per-describe setup |

## 10. Interview questions

- **Q: What does `TestBed` do?** Creates an Angular testing environment with an injector and lets you create components/services with real DI.
- **Q: How do you wait for rendering in zoneless tests?** `await fixture.whenStable()`.
- **Q: What's the default unit test runner in new Angular projects?** Vitest (since v21).

## 11. Completion checklist

- [ ] `ng test` runs in watch and CI modes with coverage.
- [ ] `src/testing/` holds providers and mock data.
- [ ] I can explain each `ComponentFixture` method.
