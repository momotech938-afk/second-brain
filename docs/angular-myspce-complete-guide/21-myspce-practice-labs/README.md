# Module 21 — Myspce Practice Labs

Fourteen progressive labs that build a working Myspce frontend skeleton using **confirmed** requirements (screens, modules, auth methods, Tailwind, single user) and the **planned** API contract. Each lab ends with an independent task; solutions and hints are in [solutions/](solutions/).

Back to [README](../README.md) · Previous: [Module 20](../20-security-and-deployment/README.md) · Next: [Module 22](../22-final-project-readiness/README.md)

> **Not the final app.** These labs are practice. You'll build the real Myspce frontend after [Module 22](../22-final-project-readiness/README.md), reusing what you learned (and much of this code).

## How to work through a lab

1. Read the objective and prerequisites; review the linked modules if needed.
2. Try the step-by-step implementation **typing the code yourself** (no copy-paste for the first pass).
3. Read the "important lines" explanation and check the expected behaviour.
4. Run the test cases.
5. Do the independent practice task **without** looking at the solution.
6. Compare with `solutions/lab-XX-solution.md`.
7. Tick the completion checklist and update [PROGRESS_TRACKER.md](../PROGRESS_TRACKER.md).

All labs use one workspace (`myspce-practice`) and build on each other. Commit after each lab (`git tag lab-01` …) so you can compare later.

## Lab list

| Lab | Title | After module | Solution |
|---|---|---|---|
| 1 | [Create the Angular application shell](lab-01-app-shell.md) | 01 | [solution](solutions/lab-01-solution.md) |
| 2 | [Reusable layout and navigation components](lab-02-layout-and-navigation.md) | 02–04 | [solution](solutions/lab-02-solution.md) |
| 3 | [Forms and reusable form controls](lab-03-forms-and-controls.md) | 12 | [solution](solutions/lab-03-solution.md) |
| 4 | [Routing and nested layouts](lab-04-routing-and-nested-layouts.md) | 10 | [solution](solutions/lab-04-solution.md) |
| 5 | [Signal-based UI state](lab-05-signal-ui-state.md) | 08 | [solution](solutions/lab-05-solution.md) |
| 6 | [Typed API services](lab-06-typed-api-services.md) | 07, 13 | [solution](solutions/lab-06-solution.md) |
| 7 | [Loading, empty, success and error states](lab-07-ui-states.md) | 03, 08, 13 | [solution](solutions/lab-07-solution.md) |
| 8 | [Authentication](lab-08-authentication.md) | 13 | [solution](solutions/lab-08-solution.md) |
| 9 | [Protected routes and role-aware UI](lab-09-protected-routes.md) | 10, 20 | [solution](solutions/lab-09-solution.md) |
| 10 | [Search, filter, sort, paginate](lab-10-search-filter-sort-paginate.md) | 14, 15 | [solution](solutions/lab-10-solution.md) |
| 11 | [Dialogs, notifications and UI components](lab-11-dialogs-and-toasts.md) | 16 | [solution](solutions/lab-11-solution.md) |
| 12 | [Unit and integration tests](lab-12-tests.md) | 19 | [solution](solutions/lab-12-solution.md) |
| 13 | [Optimise loading and rendering](lab-13-performance.md) | 17 | [solution](solutions/lab-13-solution.md) |
| 14 | [Production build and deployment](lab-14-production-build.md) | 20 | [solution](solutions/lab-14-solution.md) |

## Ground rules

- No backend exists yet: labs 6+ use the **mock backend interceptor** from [Module 07](../07-services-and-dependency-injection/04-myspce-api-layer.md). When NestJS is ready, remove it from `app.config.ts`.
- Field names follow the proposed models in [MYSPCE_PROJECT_CONTEXT.md](../MYSPCE_PROJECT_CONTEXT.md#4-data-entities-used-in-examples). If you change a model, change it there too.
- Don't add features that aren't in the confirmed scope; when a lab needs an example feature, it's labelled **(proposed)**.

## Module self-assessment

- [ ] Labs 1–5 done (foundations).
- [ ] Labs 6–9 done (data + auth).
- [ ] Labs 10–11 done (state + UI kit).
- [ ] Labs 12–14 done (quality + release).
- [ ] Every independent task done without looking at the solution first.
