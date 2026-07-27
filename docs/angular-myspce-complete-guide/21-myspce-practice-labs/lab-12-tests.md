# Lab 12 — A Real Test Suite (Unit, Component, Integration, E2E)

Back to [Labs](README.md) · Solution: [lab-12-solution.md](solutions/lab-12-solution.md) · Previous: [Lab 11](lab-11-dialogs-and-toasts.md) · Next: [Lab 13](lab-13-performance.md)

## 1. Learning objective

Turn the scattered tests from Labs 1–11 into a deliberate suite for Myspce: shared test helpers, fakes for each layer, focused tests where bugs are likely, Playwright E2E for the critical paths, and a coverage target you actually meet.

## 2. Prerequisites

All of [Module 19](../19-testing/README.md), Labs 1–11.

## 3. Problem statement

Ad-hoc tests are slow to write and break on refactors. Decide **what** to test at which level, build helpers once, and make `ng test` + `npx playwright test` the gate for every change.

## 4. Expected behaviour

| Level | Count (target) | Examples |
|---|---|---|
| Pure unit | ~25 | `parseNotesParams`, `toMessage`, `missingParts`, `toQueryParams`, pipes |
| Service/store | ~15 | `NotesStore` (remove/undo/rollback), `AuthStore` (single-flight refresh), `ConfirmService`, `ToastService` |
| HTTP/interceptor | ~8 | `NotesApi` params, `authInterceptor`, `errorInterceptor`, `reauthInterceptor` |
| Component | ~15 | `NoteCard`, `ChipFilter`, `PasswordInput`, `ErrorState`, `NotesPage` states, `LoginPage` |
| Routing | ~6 | guards, `/settings` redirect, `returnUrl`, 404 |
| E2E (Playwright) | ~8 | login/logout, session restore, note create→search→delete→undo, credential add/reveal/copy, a11y scan |

- `ng test` runs in < 30 s locally; no test depends on another's order.
- Statement coverage ≥ 80% on `core/` and `features/*/data-access`; UI components are judged by behaviour tests, not a number.
- E2E runs against the **production build** with the API mocked at the network layer (`page.route`), not the in-app mock interceptor.

## 5. Relevant concepts

Vitest (default runner since v21), `TestBed`, zoneless fixtures (`await fixture.whenStable()`), `TestBed.tick()`, `HttpTestingController`, `RouterTestingHarness`, fakes vs mocks, Playwright fixtures, `@axe-core/playwright`.

## 6. Step-by-step implementation

1. Create `src/testing/`:
   - `test-providers.ts` (`provideTestDefaults()`, from Module 19)
   - `mock-data.ts` (`MOCK_NOTES`, `MOCK_CREDENTIALS`, `MOCK_TASKS`, `MOCK_USER`)
   - `fakes.ts` (`fakeAuthStore()`, `fakeConfirm(answer)`, `fakeToast()`, `fakeNotesApi()`)
   - `dom.ts` (`byTest(fixture, id)`, `click(fixture, selector)`, `typeInto(input, value)`)
2. Add `data-test` attributes only where roles/labels can't identify an element.
3. Write the pure unit tests first (fastest feedback).
4. Store tests with fake APIs: success, failure and rollback paths.
5. Interceptor tests with `HttpTestingController`.
6. Component tests by **behaviour**: render inputs → assert DOM/ARIA → interact → assert outputs.
7. Routing tests with `RouterTestingHarness`.
8. Playwright: `fixtures.ts` with `mockApi(page)` and a `loggedIn` fixture. Write the E2E list from [Module 19 note 5](../19-testing/05-e2e-and-ci.md#4-critical-path-tests).
9. Run coverage: `ng test --coverage`. Read the report and add tests only where real logic is uncovered.

## 7. Complete working code

Helpers, fakes, and representative tests for each level are in the [solution](solutions/lab-12-solution.md).

## 8. Important lines explained

- `await fixture.whenStable()` — zoneless apps have no automatic change detection in tests; wait for scheduled rendering.
- `{ provide: NotesApi, useValue: fakeNotesApi({ remove: () => throwError(() => serverError) }) }` — override **one** behaviour per test.
- `ctrl.verify()` in `afterEach` — fails on unexpected or unflushed requests.
- `page.route('**/api/v1/**', handler)` — E2E controls the network without changing app code.
- `getByRole('button', { name: 'Delete' })` — tests that read like a user also check accessibility.

## 9. Expected output

```text
 ✓ src/app/features/notes/data-access/notes-params.spec.ts (6 tests)
 ✓ src/app/core/http/error-messages.spec.ts (9 tests)
 ✓ src/app/features/notes/data-access/notes-store.spec.ts (7 tests)
 ✓ src/app/core/auth/auth-interceptor.spec.ts (4 tests)
 …
 Test Files  31 passed (31)
      Tests  92 passed (92)
```

(Counts are an example target, not a requirement.)

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Template didn't update in the test | missing `await fixture.whenStable()` (zoneless) |
| "Expected no open requests" | a resource refetched after a signal change; flush it or assert it |
| Tests pass alone, fail together | shared mutable fixtures; clone mock data per test (`structuredClone`) |
| Fake timers break promises | use `vi.advanceTimersByTimeAsync` and restore real timers in `afterEach` |
| Snapshot tests churn on every class change | assert behaviour and ARIA, not markup |
| E2E flaky on toasts | assert with auto-waiting locators (`toBeVisible`), never fixed `waitForTimeout` |

## 11. Test cases

This lab *is* test cases. The minimum must-haves:

1. `NotesStore.remove`: confirm=false → no request; DELETE fails → item back at the same index; Undo → restore called and item reinserted.
2. `authInterceptor`: 2 concurrent 401s → 1 refresh; refresh fails → `expire()` called.
3. `NotesPage`: error → retry → empty → content transitions.
4. `LoginPage`: invalid fields block submit; 401 shows the form-level error; success navigates to a safe `returnUrl`.
5. `authGuard`: anonymous → `/login?returnUrl=…`.
6. E2E: note create → search → delete → undo.

## 12. Independent practice task

Add **CI**: a GitHub Actions workflow that runs lint, `ng test --watch=false --coverage`, `ng build`, and Playwright against `npx serve -s dist/myspce/browser`. Upload the Playwright report as an artifact on failure. Use [Module 19 note 5 §5](../19-testing/05-e2e-and-ci.md#5-ci-pipeline-github-actions) as the base, and make the job fail if coverage on `core/` drops below 80%.

## 13. Solution

See [solutions/lab-12-solution.md](solutions/lab-12-solution.md).

## 14. Completion checklist

- [ ] `src/testing/` helpers and fakes in place.
- [ ] The six must-have tests pass.
- [ ] Coverage target met for `core/` and data-access.
- [ ] Playwright critical paths + axe scan green.
- [ ] CI workflow running on push/PR.
- [ ] Tagged `lab-12`.
