# Lab 3 — Forms and Reusable Form Controls

Back to [Labs](README.md) · Solution: [lab-03-solution.md](solutions/lab-03-solution.md) · Previous: [Lab 2](lab-02-layout-and-navigation.md) · Next: [Lab 4](lab-04-routing-and-nested-layouts.md)

## 1. Learning objective

Build Myspce's core forms with **Signal Forms** (v22 stable) and a reusable `PasswordInput` control: the login form and the add-credential form, with accessible validation and a mocked submit.

## 2. Prerequisites

[Module 12](../12-reactive-forms/README.md) (especially [Signal Forms](../12-reactive-forms/04-signal-forms.md)), [Lab 2](lab-02-layout-and-navigation.md).

## 3. Problem statement

The login page (confirmed: email + password, plus Google) and the "Add Credential" form (confirmed fields from the design: name, URL, username, password, category) need validation, good error timing, and a password input with show/hide.

## 4. Expected behaviour

- Errors appear only after a field is touched or on submit.
- Submit with invalid data focuses the first invalid field and shows all errors.
- Password input has Show/Hide and (on the credential form) a strength meter.
- Submitting valid data shows "Signing in…" / "Saving…" for ~600 ms (mock) and then a success message.

## 5. Relevant concepts

`form()`, `[formField]`, schema validators with messages, `submit()`, `[formRoot]` with `submission`, `FormValueControl` or `ControlValueAccessor`, accessible errors.

## 6. Step-by-step implementation

1. Generate:
   ```bash
   ng g c features/auth/login-page
   ng g c shared/forms/password-input
   ng g c features/credentials/components/credential-form
   ```
2. **PasswordInput** — implement as a `FormValueControl<string>` (signal-native, v22):
   - `value = model('')`, `disabled = input(false)`, `touched = input(false)`, `touch = output<void>()`
   - inputs `inputId`, `autocomplete`, `showStrength`
   - an internal `visible = signal(false)` and computed `strength` (0–4)
   - emit `touch` on blur
3. **LoginPage** (route `/login` outside the shell — add it to `app.routes.ts` now):
   - model `signal({ email: '', password: '' })`
   - rules: `required` + `email` on email; `required` + `minLength(8)` on password, each with a message
   - template with `[formField]`, errors list after `touched()`, a form-level error area, `submitting()` button state
   - action: wait 600 ms (`await new Promise(r => setTimeout(r, 600))`), then if email is `fail@example.com` return `{ kind: 'invalidCredentials', message: 'Email or password is incorrect.' }`, otherwise navigate to `/dashboard`
4. **CredentialForm** with `[formRoot]` and a `submission.action` that emits `submitted` with a DTO (`url`/`notes`/`categoryId` empty → `null`), `onInvalid` focusing the first error.
   - categories come from an input (hard-code `Work, Personal, Finance, Social, Other` in the parent for now)
5. Put `CredentialForm` on a temporary `/credentials` placeholder page and log the DTO.

## 7. Complete working code

`PasswordInput`, `LoginPage` and `CredentialForm` in full are in the [solution](solutions/lab-03-solution.md).

## 8. Important lines explained

- `required(p.email, { message })` — messages live with rules, so templates just loop `errors()`.
- `submit()` / `[formRoot]` — marks everything touched, validates, runs the action once, exposes `submitting()`.
- Returning `{ kind, message }` from the action — becomes a form-level error you render from `loginForm().errors()`.
- `FormValueControl` — no `NG_VALUE_ACCESSOR`, no `writeValue`: `[formField]` binds the `value` model directly.
- `autocomplete="current-password"` vs `"new-password"` — lets password managers do the right thing on login vs in the vault.

## 9. Expected output

Login: typing `abc` in email and tabbing away shows "Enter a valid email address."; submitting empty shows both required messages; `fail@example.com` shows the form-level error.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| `Can't bind to 'formField'` | import `FormField` from `@angular/forms/signals` |
| Errors visible on page load | gate with `touched()` |
| Custom control never marks touched | emit `touch` on blur |
| Model doesn't update | the model must be a writable `signal(...)` |
| Double submit on fast double-click | use `submitting()` to disable, or rely on `submit()` ignoring concurrent submissions |

## 11. Test cases

```typescript
it('login form rejects invalid email', () => {
  TestBed.configureTestingModule({ providers: [provideRouter([])] });
  const page = TestBed.createComponent(LoginPage).componentInstance as any;
  page.model.set({ email: 'abc', password: 'S3cret!pass' });
  expect(page.loginForm.email().getError('email')).toBeTruthy();
});

it('credential form emits a cleaned DTO', async () => {
  const f = TestBed.createComponent(CredentialForm);
  f.componentRef.setInput('categories', []);
  const emitted: unknown[] = [];
  f.componentInstance.submitted.subscribe(v => emitted.push(v));
  const cmp = f.componentInstance as any;
  cmp.model.set({ name: 'GitHub', url: '', username: 'pg', password: 'S3cret!pass', categoryId: '', notes: '' });
  await submit(cmp.credForm);
  expect(emitted[0]).toEqual({ name: 'GitHub', url: null, username: 'pg', password: 'S3cret!pass', categoryId: null, notes: null });
});
```

## 12. Independent practice task

Build the **Change password** form for Settings → Security: current, new, confirm. Rules: all required; new ≥ 12 chars with upper, lower, number and symbol (one `validate` rule returning a message that lists what's missing); confirm must equal new (cross-field, re-validates when either changes). Use `PasswordInput` for all three.

## 13. Solution

See [solutions/lab-03-solution.md](solutions/lab-03-solution.md).

## 14. Completion checklist

- [ ] `PasswordInput` works with `[formField]`.
- [ ] Login form with field and form-level errors.
- [ ] Credential form emits a clean DTO.
- [ ] Change-password form with cross-field rule.
- [ ] Tests pass; tagged `lab-03`.
