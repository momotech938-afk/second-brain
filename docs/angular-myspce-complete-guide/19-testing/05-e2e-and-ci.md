# End-to-End Tests and CI

Back to [Module 19](README.md) · Previous: [Forms and routing tests](04-forms-routing-tests.md) · Next module: [20 Security and deployment](../20-security-and-deployment/README.md)

## 1. Learning objectives

- Set up Playwright for an Angular app.
- Write E2E tests for Myspce's critical flows.
- Mock the API at the network level when the backend isn't available.
- Add accessibility scans.
- Run everything in CI and handle flaky tests.

## 2. Setup

```bash
npm init playwright@latest      # choose TypeScript, folder e2e/, add GitHub Actions workflow: yes
npm i -D @axe-core/playwright
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env['CI'] ? 2 : 0,
  reporter: process.env['CI'] ? [['html', { open: 'never' }], ['github']] : 'list',
  use: {
    baseURL: 'http://localhost:4200',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'desktop-chrome', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile', use: { ...devices['Pixel 7'] } },     // Myspce is mobile-first
  ],
  webServer: { command: 'npm run start', url: 'http://localhost:4200', reuseExistingServer: !process.env['CI'] },
});
```

## 3. Mocking the API (until NestJS exists)

```typescript
// e2e/fixtures.ts
import { test as base, Page } from '@playwright/test';

export async function mockApi(page: Page) {
  const notes = [{ id: 'n1', title: 'Angular Notes', preview: 'Components…', color: null, pinned: false, isFavorite: false, category: null, tags: [], createdAt: '2026-10-28T10:00:00Z', updatedAt: '2026-10-28T10:00:00Z' }];

  await page.route('**/api/v1/auth/refresh', r => r.fulfill({ status: 401, json: { statusCode: 401, message: 'Unauthorized' } }));
  await page.route('**/api/v1/auth/login', r =>
    r.fulfill({ json: { accessToken: 'test-token', user: { id: 'u1', name: 'Pranjal Goyal', email: 'pranjal@example.com', role: 'super_admin', avatarUrl: null, bio: null } } }),
  );
  await page.route('**/api/v1/bootstrap', r => r.fulfill({ json: { user: null, theme: {}, nav: [], flags: {}, widgets: [] } }));
  await page.route('**/api/v1/notes?*', r => r.fulfill({ json: { data: notes, meta: { total: notes.length, page: 1, limit: 12 } } }));
  await page.route('**/api/v1/notes', async r => {
    if (r.request().method() === 'POST') {
      const body = r.request().postDataJSON();
      const created = { ...notes[0], id: 'n2', title: body.title };
      notes.unshift(created);
      return r.fulfill({ status: 201, json: created });
    }
    return r.fulfill({ json: { data: notes, meta: { total: notes.length, page: 1, limit: 12 } } });
  });
}

export const test = base.extend<{ loggedIn: Page }>({
  loggedIn: async ({ page }, use) => {
    await mockApi(page);
    await page.goto('/login');
    await page.getByLabel('Email').fill('pranjal@example.com');
    await page.getByLabel('Password').fill('S3cret!pass');
    await page.getByRole('button', { name: 'Sign in' }).click();
    await page.waitForURL('**/dashboard');
    await use(page);
  },
});
export { expect } from '@playwright/test';
```

When the backend exists, run E2E against a seeded test database instead, and keep a smaller set of mocked tests for error scenarios.

## 4. Critical-path tests

```typescript
// e2e/auth.spec.ts
import { test, expect } from './fixtures';
import { mockApi } from './fixtures';

test('redirects to login and back after signing in', async ({ page }) => {
  await mockApi(page);
  await page.goto('/notes');
  await expect(page).toHaveURL(/\/login\?returnUrl=%2Fnotes/);
  await page.getByLabel('Email').fill('pranjal@example.com');
  await page.getByLabel('Password').fill('S3cret!pass');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL(/\/notes$/);
});

test('shows validation errors', async ({ page }) => {
  await mockApi(page);
  await page.goto('/login');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByText('Email is required.')).toBeVisible();
});
```

```typescript
// e2e/notes.spec.ts
import { test, expect } from './fixtures';

test('creates a note and finds it with search', async ({ loggedIn: page }) => {
  await page.getByRole('link', { name: 'Notes' }).click();
  await page.getByRole('button', { name: /new note/i }).click();
  await page.getByLabel('Title').fill('Read Angular documentation');
  await page.getByRole('button', { name: /save/i }).click();
  await expect(page.getByText('Note saved')).toBeVisible();

  await page.getByRole('link', { name: 'Notes' }).click();
  await page.getByPlaceholder('Search notes…').fill('angular doc');
  await expect(page.getByRole('heading', { name: 'Read Angular documentation' })).toBeVisible();
});
```

```typescript
// e2e/a11y.spec.ts
import AxeBuilder from '@axe-core/playwright';
import { test, expect } from './fixtures';

for (const path of ['/dashboard', '/notes', '/credentials', '/tasks', '/settings/profile']) {
  test(`no serious a11y violations on ${path}`, async ({ loggedIn: page }) => {
    await page.goto(path);
    const results = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa']).analyze();
    expect(results.violations.filter(v => ['serious', 'critical'].includes(v.impact ?? ''))).toEqual([]);
  });
}
```

Myspce's E2E list (v1): login/logout, session restore on reload, create/edit/delete/restore a note, filter notes by chip and URL, add credential + copy password (clipboard permission in Playwright: `context.grantPermissions(['clipboard-read', 'clipboard-write'])`), toggle a task and see dashboard progress update, switch theme and reload, admin hides a nav item.

## 5. CI pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - run: npx ng lint
      - run: npx ng test --no-watch --coverage
      - run: npx ng build
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test --project=desktop-chrome
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: playwright-report, path: playwright-report }
```

(Action versions shown are current majors at the time of writing; update as needed. Node 24 satisfies Angular 22's range.)

## 6. Flaky tests

| Cause | Fix |
|---|---|
| Fixed `waitForTimeout(1000)` | web-first assertions (`await expect(locator).toBeVisible()`) auto-wait |
| Shared state between tests | each test sets up its own data/mocks; `fullyParallel` exposes leaks |
| Animations/transitions | `reducedMotion: 'reduce'` in `use`, or disable animations in test builds |
| Real clock/time zones | `page.clock.setFixedTime(...)` and a fixed `timezoneId` in config |
| Network races | wait for the response: `page.waitForResponse('**/api/v1/notes*')` |
| Unit tests: real timers, leaking fake timers | `vi.useRealTimers()` in `afterEach` |

Quarantine a flaky test (`test.fixme`) only with a ticket to fix it; never ignore failures.

## 7. Interview questions

- **Q: What should E2E tests cover?** A small number of critical user journeys end-to-end; details belong in unit/integration tests.
- **Q: How do you make Playwright tests stable?** Role-based locators, web-first assertions, isolated data, network mocking or seeded backends, fixed time.

## 8. Completion checklist

- [ ] Playwright configured with desktop and mobile projects.
- [ ] Login, notes and a11y specs pass against mocks.
- [ ] CI runs lint, unit tests with coverage, build and E2E.
