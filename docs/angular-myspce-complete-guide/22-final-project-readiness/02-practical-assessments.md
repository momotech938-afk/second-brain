# Practical Assessments

Back to [Module 22](README.md) · Previous: [Concept checklist](01-concept-checklist.md) · Next: [Debugging exercises](03-debugging-exercises.md)

Six timed tasks. Work in a **fresh branch** of your lab project (`git switch -c assessment`), set a timer, and don't open the lab solutions. You may use the official docs ([angular.dev](https://angular.dev)) — real work allows it.

**Pass rule:** a task passes when every acceptance criterion is met within the time limit + 25%. Pass at least **5 of 6**.

---

## A1 — Category manager (components, signals, forms) · 60 min

Build `/settings/customization/categories` for **note categories** (the planned `Category` model: `id, module, name, slug, color`).

**Acceptance criteria**
1. Lists categories from `GET /categories?module=note` (add it to the mock) with loading/empty/error states.
2. "Add category" opens a modal with a Signal Form: name required (2–30 chars), color from 8 swatches (keyboard-selectable, `role="radiogroup"`), slug auto-derived from the name and editable.
3. Duplicate slug → server returns 409 → error shown **on the slug field**.
4. Edit and delete (with confirm + Undo).
5. One component test (form validation) and one store test (409 handling).

**Concepts:** `form()`, `submit` with field errors, `Modal`, `ConfirmService`, `ToastService`, `httpResource`/`rxResource`.

---

## A2 — Tasks "Today" with optimistic toggle (state, HTTP) · 60 min

**Acceptance criteria**
1. `/tasks?view=today` lists today's tasks grouped by priority (high → low), using the planned `PATCH /tasks/:id/toggle`.
2. Checking a task toggles **instantly** (optimistic) and rolls back with an error toast if the request fails (simulate with a task titled "boom").
3. A "Week progress" bar on the same page (`done / total` for the week) updates immediately on toggle without refetching.
4. Toggling twice quickly sends requests in order and ends in the correct state (hint: `concatMap` or per-id queueing).
5. Two store tests: rollback, and the double-toggle ordering.

---

## A3 — Notes deep linking (routing, URL state) · 45 min

**Acceptance criteria**
1. `/notes?tag=angular` filters by tag (add `tag` support to the mock). Tags render as chips on cards; clicking one sets the filter.
2. `/notes/:id` loads a note via a resource keyed on the `id` input; an invalid id shows a "Note not found" page state (404 from the mock), not a blank screen.
3. Navigating from `/notes/a` to `/notes/b` (same component) loads `b` without a full reload, and the previous request is cancelled.
4. Leaving the editor with unsaved changes asks via `ConfirmService`.
5. One `RouterTestingHarness` test for the 404 state.

---

## A4 — Credential detail with reveal and re-auth (security, interceptors) · 75 min

**Acceptance criteria**
1. `/credentials/:id` shows details (never the secret) from the list data or `GET /credentials/:id`.
2. "Reveal" calls `POST /credentials/:id/reveal`. If the mock returns `403 REAUTH_REQUIRED`, a re-auth modal appears; after the correct password the reveal **retries automatically**.
3. The secret auto-hides after 30 s, on navigation away, and when the tab is hidden (`visibilitychange`).
4. Copy works; the toast says "Copied" and never contains the secret; nothing secret is logged.
5. Interceptor test: 403 REAUTH → dialog → retry once; dialog cancelled → error propagates.

---

## A5 — Dashboard widget registry (dynamic components, performance) · 60 min

**Acceptance criteria**
1. Widgets come from `GET /bootstrap` → `widgets: string[]` (order matters). Unknown ids are skipped with a dev-mode console warning.
2. Each widget is lazy (`WIDGET_REGISTRY` of `() => import(...)`) and rendered with `NgComponentOutlet` or `ViewContainerRef`.
3. Widgets below the fold use `@defer (on viewport)` with same-size placeholders (no CLS).
4. A failing widget shows its own error + retry; others unaffected.
5. Production build shows separate chunks per widget; record before/after initial bundle size.

---

## A6 — Theme and preferences end-to-end (DI, effects, a11y, E2E) · 45 min

**Acceptance criteria**
1. `ThemeStore` with `mode: 'light' | 'dark' | 'system'`, `isDark` computed (respects `prefers-color-scheme` for `system`), and `toggle()`.
2. The `dark` class on `<html>` is applied by **one** `effect`. The choice persists via `KV_STORAGE` (an injectable wrapper around `localStorage` with try/catch).
3. No flash of the wrong theme on reload (an inline script in `index.html` sets the class before Angular boots — explain why that's acceptable for CSP or use a hashed script).
4. Appearance tab: a `role="radiogroup"` with three options, fully keyboard-operable, visible focus.
5. Playwright test: switch to dark → reload → still dark; axe scan has no serious violations in both themes.

---

## Recording results

| Task | Time taken | Criteria met (of 5) | Pass? | What slowed me down |
|---|---|---|---|---|
| A1 | | | | |
| A2 | | | | |
| A3 | | | | |
| A4 | | | | |
| A5 | | | | |
| A6 | | | | |

Use the "What slowed me down" column to pick which notes to re-read before starting the real project.
