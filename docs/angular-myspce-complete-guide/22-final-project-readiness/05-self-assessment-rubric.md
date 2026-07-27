# Self-Assessment Rubric

Back to [Module 22](README.md) · Previous: [Implementation plan](04-implementation-plan.md)

Score each row honestly, using evidence from the [practical assessments](02-practical-assessments.md), the [debugging exercises](03-debugging-exercises.md) and your lab repo. **You are ready to build Myspce when at least 10 of the 12 rows are "Ready to build"**, and rows 4 (Signals), 6 (Routing & guards), 8 (HTTP & auth) and 12 (Security) are among them.

| # | Area | Beginner | Intermediate | Ready to build | Evidence |
|---|---|---|---|---|---|
| 1 | **TypeScript & tooling** | Writes types by copying examples; `any` appears often | Uses generics, unions and utility types; fixes compiler errors alone | Designs typed models/DTOs from an API contract; zero `any`; reads CLI/build errors and fixes config | Lab 6 models, A1 |
| 2 | **Components & templates** | Builds components from examples; confuses bindings | Uses inputs/outputs/`model()`, control flow and `track` correctly | Splits pages into presentational components with clear APIs; templates contain no business logic | Labs 1–2, 5 |
| 3 | **Dependency injection & services** | Uses `inject()` in components only | Creates services, tokens and providers; understands root vs component scope | Designs the API/store/service layers; uses tokens for config; never hits `NG0203` | Lab 6, D5 |
| 4 | **Signals & reactivity** | Uses `signal` and `set`; reaches for `effect` to sync state | Uses `computed`, `linkedSignal`, resources; knows when RxJS is better | Builds stores with readonly public state, resources keyed on params, optimistic updates, no state-syncing effects | Labs 5, 7, A2, D1, D6 |
| 5 | **Change detection & lifecycle** | Doesn't know why the view didn't update | Explains OnPush/zoneless; cleans up subscriptions | Diagnoses rendering issues with DevTools; uses `afterNextRender`/`DestroyRef` correctly | Lab 13, D10, D14 |
| 6 | **Routing & guards** | Sets up flat routes | Lazy routes, params as inputs, guards with `UrlTree` | Designs the full route tree with layouts, `canMatch` roles, session-aware guards, URL-synced state, unsaved-change protection | Labs 4, 9, 10, A3, D2, D9 |
| 7 | **Forms** | Builds a form with examples; validation messages inconsistent | Reactive or Signal Forms with custom/cross-field validators | Accessible Signal Forms with server errors mapped to fields, custom `FormValueControl`s, focus on first error | Lab 3, A1, D8 |
| 8 | **HTTP, auth & error handling** | Calls `HttpClient` in components | API services, interceptors, error messages, context tokens | Single-flight refresh, re-auth retry, safe retries, consistent error UX, no duplicate toasts | Labs 7, 8, 11, A4, D3, D12 |
| 9 | **RxJS** | Subscribes and nests subscribes | Uses `switchMap`, `debounceTime`, `catchError` appropriately | Picks flattening operators deliberately, handles cancellation and races, bridges to signals cleanly | Lab 10, A2, D4 |
| 10 | **UI kit, accessibility & UX states** | Visual-only components; mouse-only | Modals, menus, toasts that mostly work with a keyboard | Every overlay keyboard- and screen-reader-friendly; four UI states everywhere; axe clean | Labs 7, 11, A6 |
| 11 | **Testing & performance** | Few tests; doesn't measure | Unit + component tests; knows `@defer` and budgets | Layered suite with fakes and E2E in CI; measures before optimising; budgets enforced | Labs 12–13, A5, D11, D13 |
| 12 | **Security & deployment** | Unsure what's safe in the frontend | Knows XSS protection and token trade-offs; can build a Docker image | Handles secrets safely in the UI, no secrets in bundles, CSP/headers configured, diagnoses production failures, CI/CD with OIDC | Labs 9, 11, 14, A4, D7 |

## Your scores

| # | Area | Score (B / I / R) | Evidence link or note | Action if not R |
|---|---|---|---|---|
| 1 | TypeScript & tooling | | | |
| 2 | Components & templates | | | |
| 3 | DI & services | | | |
| 4 | Signals & reactivity | | | |
| 5 | Change detection & lifecycle | | | |
| 6 | Routing & guards | | | |
| 7 | Forms | | | |
| 8 | HTTP, auth & errors | | | |
| 9 | RxJS | | | |
| 10 | UI kit, a11y & UX states | | | |
| 11 | Testing & performance | | | |
| 12 | Security & deployment | | | |

**Total "Ready to build":** ___ / 12  → Ready? ☐ Yes ☐ Not yet (re-score on: ________)

## If you're not ready yet

| Row below R | Do this |
|---|---|
| 1 | [Module 00 notes 2–3](../00-prerequisites/README.md); type every lab model from scratch |
| 2 | Redo Lab 2 and Lab 5 without the solution |
| 3 | [Module 07](../07-services-and-dependency-injection/README.md) notes 2–4; rebuild the API layer from Lab 6 |
| 4 | [Module 08](../08-signals-and-reactivity/README.md) all notes; redo A2 |
| 5 | [Module 09](../09-lifecycle-and-change-detection/README.md); profile Lab 13 again |
| 6 | [Module 10](../10-routing/README.md) notes 3–4; redo Labs 9–10 |
| 7 | [Module 12 note 4](../12-reactive-forms/04-signal-forms.md); redo Lab 3 and A1 |
| 8 | [Module 13](../13-http-and-api-integration/README.md) notes 2–3; redo Lab 8 and A4 |
| 9 | [Module 14](../14-rxjs/README.md) note 2; write five marble-style tests |
| 10 | [Module 16 note 4](../16-advanced-components/04-reusable-ui-kit.md), [Module 20 note 3](../20-security-and-deployment/03-accessibility-i18n.md); keyboard-only run of every page |
| 11 | [Module 19](../19-testing/README.md), [Module 17](../17-performance/README.md); redo Labs 12–13 |
| 12 | [Module 20](../20-security-and-deployment/README.md) notes 1, 2 and 4; redo Lab 14 |

When you're ready, start [M0 in the implementation plan](04-implementation-plan.md#3-milestones) and mark Module 22 complete in [PROGRESS_TRACKER.md](../PROGRESS_TRACKER.md).
