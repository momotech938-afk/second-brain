# Lab 12 — Solution

Back to [Lab 12](../lab-12-tests.md) · [Labs index](../README.md)

## `src/testing/fakes.ts`

```typescript
import { computed, signal } from '@angular/core';
import { of } from 'rxjs';
import type { User } from '../app/core/models';
import type { NotesApi } from '../app/features/notes/data-access/notes-api';
import { MOCK_NOTES, MOCK_USER } from './mock-data';

export function fakeAuthStore(user: User | null = MOCK_USER) {
  const status = signal<'unknown' | 'authenticated' | 'anonymous'>(user ? 'authenticated' : 'anonymous');
  return {
    user: signal(user),
    status,
    isLoggedIn: computed(() => status() === 'authenticated'),
    accessToken: signal(user ? 'test-token' : null),
    ensureSession: vi.fn(async () => {}),
    login: vi.fn(async () => {}),
    logout: vi.fn(async () => {}),
    refresh: vi.fn(() => of('new-token')),
    expire: vi.fn(),
  };
}

export const fakeConfirm = (answer: boolean) => ({ ask: vi.fn(async () => answer), answer: vi.fn(), current: signal(null) });

export const fakeToast = () => ({ success: vi.fn(), error: vi.fn(), info: vi.fn(), dismiss: vi.fn(), toasts: signal([]) });

export function fakeNotesApi(overrides: Partial<Record<keyof NotesApi, unknown>> = {}) {
  const data = structuredClone(MOCK_NOTES);
  return {
    list: vi.fn(() => of({ data, meta: { total: data.length, page: 1, limit: 8 } })),
    get: vi.fn((id: string) => of({ ...data.find(n => n.id === id)!, content: '' })),
    create: vi.fn(),
    update: vi.fn((id: string, dto: object) => of({ ...data.find(n => n.id === id)!, ...dto })),
    remove: vi.fn(() => of(undefined)),
    restore: vi.fn((id: string) => of(data.find(n => n.id === id)!)),
    togglePin: vi.fn((id: string) => of(data.find(n => n.id === id)!)),
    ...overrides,
  };
}
```

`vi` is a global in Angular's Vitest setup when `globals` is enabled; otherwise `import { vi } from 'vitest'`. Add `MOCK_USER` to `mock-data.ts` (`role: 'super_admin'`).

## `src/testing/dom.ts`

```typescript
import { ComponentFixture } from '@angular/core/testing';

export const byTest = <T extends HTMLElement = HTMLElement>(f: ComponentFixture<unknown>, id: string) =>
  f.nativeElement.querySelector(`[data-test="${id}"]`) as T | null;

export async function typeInto(f: ComponentFixture<unknown>, input: HTMLInputElement, value: string) {
  input.value = value;
  input.dispatchEvent(new Event('input'));
  input.dispatchEvent(new Event('blur'));
  await f.whenStable();
}

export async function click(f: ComponentFixture<unknown>, el: HTMLElement | null) {
  if (!el) throw new Error('click(): element not found');
  el.click();
  await f.whenStable();
}
```

## Store test — remove / undo / rollback

```typescript
// features/notes/data-access/notes-store.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpErrorResponse } from '@angular/common/http';
import { throwError } from 'rxjs';
import { NotesStore } from './notes-store';
import { NotesApi } from './notes-api';
import { ConfirmService } from '../../../shared/ui/confirm/confirm-service';
import { ToastService } from '../../../shared/ui/toast/toast-service';
import { AuthStore } from '../../../core/auth/auth-store';
import { fakeAuthStore, fakeConfirm, fakeNotesApi, fakeToast } from '../../../../testing/fakes';
import { provideTestDefaults } from '../../../../testing/test-providers';

function setup({ confirm = true, api = {} } = {}) {
  const toast = fakeToast();
  const notesApi = fakeNotesApi(api);
  TestBed.configureTestingModule({
    providers: provideTestDefaults(
      { provide: NotesApi, useValue: notesApi },
      { provide: ConfirmService, useValue: fakeConfirm(confirm) },
      { provide: ToastService, useValue: toast },
      { provide: AuthStore, useValue: fakeAuthStore() },
    ),
  });
  const store = TestBed.inject(NotesStore);
  return { store, toast, notesApi };
}

const loaded = async (store: NotesStore) => {
  await vi.waitFor(() => expect(store.list.hasValue()).toBe(true));
};

describe('NotesStore.remove', () => {
  it('does nothing when the user cancels', async () => {
    const { store, notesApi } = setup({ confirm: false });
    await loaded(store);
    await store.remove(store.notes()[0]!.id);
    expect(notesApi.remove).not.toHaveBeenCalled();
  });

  it('removes optimistically and offers Undo', async () => {
    const { store, toast } = setup();
    await loaded(store);
    const id = store.notes()[0]!.id;
    await store.remove(id);
    expect(store.notes().some(n => n.id === id)).toBe(false);
    expect(toast.success).toHaveBeenCalledWith('Moved to Trash', expect.objectContaining({ action: expect.objectContaining({ label: 'Undo' }) }));
  });

  it('Undo restores the note at its original index', async () => {
    const { store, toast, notesApi } = setup();
    await loaded(store);
    const target = store.notes()[1]!;
    await store.remove(target.id);
    const [, opts] = toast.success.mock.calls[0]!;
    opts.action.run();
    await vi.waitFor(() => expect(store.notes()[1]?.id).toBe(target.id));
    expect(notesApi.restore).toHaveBeenCalledWith(target.id);
  });

  it('rolls back when DELETE fails', async () => {
    const err = new HttpErrorResponse({ status: 500 });
    const { store, toast } = setup({ api: { remove: vi.fn(() => throwError(() => err)) } });
    await loaded(store);
    const before = store.notes().map(n => n.id);
    await store.remove(before[0]!);
    expect(store.notes().map(n => n.id)).toEqual(before);
    expect(toast.error).toHaveBeenCalled();
  });
});
```

## Interceptor test — refresh failure

```typescript
// core/auth/auth-interceptor.spec.ts (one more case beyond Lab 8)
it('calls expire() when the refresh fails', () => {
  const auth = fakeAuthStore();
  auth.refresh.mockReturnValue(throwError(() => new HttpErrorResponse({ status: 401 })));
  TestBed.configureTestingModule({
    providers: [
      provideHttpClient(withInterceptors([authInterceptor])), provideHttpClientTesting(),
      { provide: API_URL, useValue: '/api/v1' }, { provide: AuthStore, useValue: auth },
    ],
  });
  TestBed.inject(HttpClient).get('/api/v1/notes').subscribe({ error: () => {} });
  TestBed.inject(HttpTestingController).expectOne('/api/v1/notes').flush(null, { status: 401, statusText: 'Unauthorized' });
  expect(auth.expire).toHaveBeenCalledOnce();
});
```

## Component test — `NotesPage` state transitions

```typescript
// features/notes/notes-page.spec.ts
describe('NotesPage', () => {
  let ctrl: HttpTestingController;
  beforeEach(() => {
    TestBed.configureTestingModule({ providers: provideTestDefaults({ provide: AuthStore, useValue: fakeAuthStore() }) });
    ctrl = TestBed.inject(HttpTestingController);
  });
  afterEach(() => ctrl.verify());

  it('error → retry → empty → content', async () => {
    const harness = await RouterTestingHarness.create();
    await harness.navigateByUrl('/', NotesPage);          // route-less: render the page directly
    const el = () => harness.routeNativeElement!;

    ctrl.expectOne(r => r.url === '/api/v1/notes').flush({ message: 'boom' }, { status: 500, statusText: 'Server Error' });
    await harness.fixture.whenStable();
    expect(el().querySelector('[role=alert]')).not.toBeNull();

    (el().querySelector('[data-test=retry]') as HTMLButtonElement).click();
    ctrl.expectOne(r => r.url === '/api/v1/notes').flush({ data: [], meta: { total: 0, page: 1, limit: 8 } });
    await harness.fixture.whenStable();
    expect(el().textContent).toContain('No notes yet');
  });
});
```

`harness.navigateByUrl(url, Component)` requires a route for that URL. The simplest setup is `provideRouter([{ path: '', component: NotesPage }])` in this spec instead of the empty router from `provideTestDefaults()`. Because `provideTestDefaults` takes extra providers, add the route-specific `provideRouter` **after** it; the last provider wins.

## Routing test — returnUrl safety

```typescript
it('ignores external returnUrl values', async () => {
  // LoginPage with fakeAuthStore().login resolving
  const harness = await RouterTestingHarness.create();
  await harness.navigateByUrl('/login?returnUrl=%2F%2Fevil.example', LoginPage);
  /* fill valid email + password via typeInto, submit */
  await vi.waitFor(() => expect(TestBed.inject(Router).url).toBe('/dashboard'));
});
```

## Playwright fixtures

```typescript
// e2e/fixtures.ts
import { test as base, expect, Page } from '@playwright/test';
import notes from './data/notes.json' with { type: 'json' };

export async function mockApi(page: Page) {
  let session = false;
  let store = structuredClone(notes);
  let trash: typeof store = [];

  await page.route('**/api/v1/**', async route => {
    const req = route.request();
    const url = new URL(req.url());
    const path = url.pathname.replace('/api/v1', '');
    const json = (body: unknown, status = 200) => route.fulfill({ status, json: body });

    if (path === '/auth/login') { session = true; return json({ accessToken: 't', user: { id: 'u1', name: 'Pranjal', email: 'pranjal@example.com', avatarUrl: null, bio: null, role: 'super_admin' } }); }
    if (path === '/auth/refresh') return session ? json({ accessToken: 't', user: { id: 'u1', name: 'Pranjal', email: 'pranjal@example.com', avatarUrl: null, bio: null, role: 'super_admin' } }) : json({ message: 'No session' }, 401);
    if (path === '/auth/logout') { session = false; return json(null); }
    if (path === '/bootstrap') return json({ flags: { tasks: true }, widgets: [], nav: [], settings: {} });
    if (path === '/notes' && req.method() === 'GET') {
      const q = (url.searchParams.get('q') ?? '').toLowerCase();
      const data = store.filter(n => !q || n.title.toLowerCase().includes(q));
      return json({ data, meta: { total: data.length, page: 1, limit: 8 } });
    }
    if (path === '/notes' && req.method() === 'POST') {
      const body = req.postDataJSON();
      const note = { ...store[0], id: `n${Date.now()}`, title: body.title, preview: body.content ?? '', pinned: false };
      store = [note, ...store];
      return json(note, 201);
    }
    const del = path.match(/^\/notes\/(\w+)$/);
    if (del && req.method() === 'DELETE') { trash.push(...store.filter(n => n.id === del[1])); store = store.filter(n => n.id !== del[1]); return route.fulfill({ status: 204 }); }
    const res = path.match(/^\/notes\/(\w+)\/restore$/);
    if (res) { const n = trash.find(x => x.id === res[1])!; store = [n, ...store]; trash = trash.filter(x => x !== n); return json(n); }
    return json({ message: `No E2E mock for ${req.method()} ${path}` }, 404);
  });
}

export const test = base.extend<{ loggedIn: Page }>({
  loggedIn: async ({ page }, use) => {
    await mockApi(page);
    await page.goto('/login');
    await page.getByLabel('Email').fill('pranjal@example.com');
    await page.getByLabel('Password').fill('S3cret!pass');
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page).toHaveURL(/\/dashboard/);
    await use(page);
  },
});
export { expect };
```

Build the app with `useMockApi: false` for E2E (a `e2e` configuration in `angular.json` with its own environment file), so Playwright's `page.route` is the only mock.

```typescript
// e2e/notes-undo.spec.ts
import { test, expect } from './fixtures';

test('delete a note and undo', async ({ loggedIn: page }) => {
  await page.goto('/notes');
  const card = page.getByRole('heading', { name: 'Angular Notes' });
  await expect(card).toBeVisible();

  await page.getByRole('button', { name: 'Actions for Angular Notes' }).click();
  await page.getByRole('menuitem', { name: 'Delete' }).click();
  await page.getByRole('dialog').getByRole('button', { name: 'Delete' }).click();

  await expect(card).toBeHidden();
  await page.getByRole('button', { name: 'Undo' }).click();
  await expect(card).toBeVisible();
});
```

## Independent task — CI

Start from the workflow in [Module 19 note 5 §5](../../19-testing/05-e2e-and-ci.md#5-ci-pipeline-github-actions) and add:

```yaml
      - run: npx ng test --watch=false --coverage
      - name: Enforce core coverage
        run: node scripts/check-coverage.mjs coverage/coverage-summary.json src/app/core 80
      - run: npx ng build --configuration e2e
      - run: npx playwright install --with-deps chromium
      - run: npx serve -s dist/myspce/browser -l 4200 & npx wait-on http://localhost:4200
      - run: npx playwright test
      - if: failure()
        uses: actions/upload-artifact@v4
        with: { name: playwright-report, path: playwright-report }
```

```javascript
// scripts/check-coverage.mjs — fails if average statement coverage under a folder is below the threshold
import { readFileSync } from 'node:fs';
const [file, folder, min] = process.argv.slice(2);
const summary = JSON.parse(readFileSync(file, 'utf8'));
const rows = Object.entries(summary).filter(([k]) => k.includes(folder));
const covered = rows.reduce((s, [, v]) => s + v.statements.covered, 0);
const total = rows.reduce((s, [, v]) => s + v.statements.total, 0);
const pct = total ? (covered / total) * 100 : 100;
console.log(`${folder}: ${pct.toFixed(1)}% statements`);
if (pct < Number(min)) process.exit(1);
```

`coverage-summary.json` requires the `json-summary` reporter. Configure coverage reporters in your Vitest/Angular test options. The exact option name depends on your builder version, so check `ng test --help`.
