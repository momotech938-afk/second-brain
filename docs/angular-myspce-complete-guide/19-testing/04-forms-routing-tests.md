# Forms and Routing Tests

Back to [Module 19](README.md) · Previous: [Service and HTTP tests](03-service-http-tests.md) · Next: [E2E and CI](05-e2e-and-ci.md)

## 1. Learning objectives

- Test validators, Signal Forms and Reactive Forms without the DOM.
- Test form templates (error messages, submit button state) where it matters.
- Test guards, resolvers and routed components.
- Test navigation and redirects with `RouterTestingHarness`.

## 2. Validators — pure functions

```typescript
describe('strongPassword', () => {
  const v = strongPassword(12);
  it.each([
    ['', null],
    ['Short1!', { strongPassword: { missing: ['length'] } }],
    ['alllowercase123!', { strongPassword: { missing: ['uppercase'] } }],
    ['ValidPassw0rd!', null],
  ])('%s', (input, expected) => expect(v(new FormControl(input))).toEqual(expected));
});
```

## 3. Signal Forms

```typescript
describe('login form', () => {
  function setup() {
    const login = vi.fn().mockResolvedValue(undefined);
    TestBed.configureTestingModule({ providers: [provideRouter([]), { provide: AuthStore, useValue: { login } }] });
    const page = TestBed.createComponent(LoginPage).componentInstance as any;
    return { page, login };
  }

  it('requires a valid email', () => {
    const { page } = setup();
    page.model.set({ email: 'nope', password: 'S3cret!pass' });
    expect(page.loginForm.email().getError('email')).toBeTruthy();
    expect(page.loginForm().valid()).toBe(false);
  });

  it('does not call login when invalid, marks fields touched', async () => {
    const { page, login } = setup();
    await page.onSubmit(new Event('submit'));
    expect(login).not.toHaveBeenCalled();
    expect(page.loginForm.email().touched()).toBe(true);
  });

  it('shows a form-level error when the server rejects credentials', async () => {
    const { page, login } = setup();
    login.mockRejectedValueOnce(new Error('401'));
    page.model.set({ email: 'pranjal@example.com', password: 'wrongpass1' });
    await page.onSubmit(new Event('submit'));
    expect(page.loginForm().errors().map((e: { kind: string }) => e.kind)).toContain('invalidCredentials');
  });
});
```

## 4. Reactive Forms

```typescript
it('credential form requires name, username and password', () => {
  const f = TestBed.createComponent(CredentialForm);
  f.componentRef.setInput('categories', []);
  const form = (f.componentInstance as any).form as FormGroup;
  expect(form.controls['name']!.hasError('required')).toBe(true);
  form.patchValue({ name: 'GitHub', username: 'pg', password: 'S3cret!pass' });
  expect(form.valid).toBe(true);
});
```

## 5. Form templates (only what matters)

```typescript
it('shows the error message after blur', async () => {
  const fixture = TestBed.createComponent(LoginPage);
  await fixture.whenStable();
  const email: HTMLInputElement = fixture.nativeElement.querySelector('#email');
  email.value = 'bad';
  email.dispatchEvent(new Event('input'));
  email.dispatchEvent(new Event('blur'));
  await fixture.whenStable();
  expect(fixture.nativeElement.querySelector('#email-errors')?.textContent).toContain('valid email');
  expect(email.getAttribute('aria-invalid')).toBe('true');
});
```

## 6. Guards (functions)

```typescript
it('roleGuard blocks non-admins', async () => {
  TestBed.configureTestingModule({
    providers: [provideRouter([]), { provide: AuthStore, useValue: { ensureSession: () => Promise.resolve(), user: () => ({ role: 'user' }) } }],
  });
  const result = await TestBed.runInInjectionContext(() => roleGuard(['admin'])({} as Route, [], {} as never));
  expect(TestBed.inject(Router).serializeUrl(result as UrlTree)).toBe('/dashboard');
});
```

(v22 `CanMatchFn` receives a third `currentSnapshot` argument; tests can pass a dummy value.)

## 7. Routing integration with `RouterTestingHarness`

```typescript
import { RouterTestingHarness } from '@angular/router/testing';

describe('app routes', () => {
  function setup(loggedIn: boolean) {
    TestBed.configureTestingModule({
      providers: [
        provideRouter([
          { path: 'login', component: FakeLogin },
          { path: '', canActivate: [authGuard], children: [{ path: 'notes', component: FakeNotes }] },
        ], withComponentInputBinding()),
        { provide: AuthStore, useValue: { ensureSession: () => Promise.resolve(), isLoggedIn: () => loggedIn } },
      ],
    });
  }

  it('redirects anonymous users to login with returnUrl', async () => {
    setup(false);
    const harness = await RouterTestingHarness.create();
    await harness.navigateByUrl('/notes');
    expect(TestBed.inject(Router).url).toBe('/login?returnUrl=%2Fnotes');
  });

  it('renders notes for logged-in users', async () => {
    setup(true);
    const harness = await RouterTestingHarness.create();
    const page = await harness.navigateByUrl('/notes', FakeNotes);
    expect(page).toBeInstanceOf(FakeNotes);
  });
});

@Component({ template: 'login' }) class FakeLogin {}
@Component({ template: 'notes' }) class FakeNotes {}
```

`RouterTestingHarness` renders a real `RouterOutlet`, runs guards/resolvers, and gives you the activated component.

### Resolvers

```typescript
it('noteResolver redirects to /notes on 404', async () => {
  TestBed.configureTestingModule({ providers: [provideRouter([]), { provide: NotesApi, useValue: { get: () => throwError(() => new HttpErrorResponse({ status: 404 })) } }] });
  const route = { paramMap: convertToParamMap({ id: 'missing' }) } as ActivatedRouteSnapshot;
  const result = await firstValueFrom(TestBed.runInInjectionContext(() => noteResolver(route, {} as RouterStateSnapshot)) as Observable<unknown>);
  expect(result).toBeInstanceOf(RedirectCommand);
});
```

### `canDeactivate`

```typescript
it('asks before leaving with unsaved changes', async () => {
  const ask = vi.fn().mockResolvedValue(false);
  TestBed.configureTestingModule({ providers: [{ provide: ConfirmService, useValue: { ask } }] });
  const result = await TestBed.runInInjectionContext(() =>
    unsavedChangesGuard({ hasUnsavedChanges: () => true }, {} as never, {} as never, {} as never),
  );
  expect(ask).toHaveBeenCalled();
  expect(result).toBe(false);
});
```

## 8. Common mistakes

| Mistake | Fix |
|---|---|
| Testing forms only through the DOM | test models/validators directly; DOM tests for messages/a11y |
| Testing the whole real route table with real pages | fake page components; test guards separately |
| Forgetting `withComponentInputBinding()` in route tests | inputs stay undefined |
| Calling guards outside an injection context | `TestBed.runInInjectionContext` |

## 9. Practice

1. Test the change-password cross-field validation (Signal Forms version).
2. Test that `/admin` is not matched for non-admins (falls through to `**`).
3. Test that `/notes?category=work` binds `category` as an input on `NotesPage`.

## 10. Interview questions

- **Q: How do you test a functional guard?** Call it inside `TestBed.runInInjectionContext` with fake dependencies and assert the returned value/`UrlTree`.
- **Q: What does `RouterTestingHarness` give you?** A real router outlet in a test, so you can navigate and get the activated component.

## 11. Completion checklist

- [ ] Login and credential forms tested at model level.
- [ ] Guards, resolver and route redirects tested.
