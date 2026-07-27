# Signal Forms (stable in Angular 22)

Back to [Module 12](README.md) · Previous: [Dynamic and reusable forms](03-dynamic-and-reusable-forms.md) · Next module: [13 HTTP and API integration](../13-http-and-api-integration/README.md)

> **Version note:** Signal Forms arrived as experimental in **v21** and are **stable in v22** (`@angular/forms/signals`). v22 also added `getError()`, `minDate()`/`maxDate()`, a consistent `when` option for rules, `debounce(field, 'blur')`, `debounce` options on async validators, `reloadValidation()`, and changed custom controls' `touched` model into a `touched` input + `touch` output. Examples below use the v22 API; verify unusual options against the [API reference](https://angular.dev/api).

## 1. Learning objectives

- Create a form from a signal model with `form()` and bind fields with `[formField]`.
- Add validation with schema functions: built-ins, `validate`, cross-field, async/HTTP.
- Read field state (value, valid, touched, errors) as signals.
- Submit with `submit()` or `[formRoot]`, including server errors.
- Build custom controls with `FormValueControl`.
- Decide between Signal Forms and Reactive Forms.

## 2. Prerequisites

[Signals](../08-signals-and-reactivity/README.md), form state vocabulary from [Module 11](../11-template-driven-forms/01-template-driven-forms.md).

## 3. What is it?

A forms API where **your signal is the model**. `form(model, schema)` returns a **FieldTree** mirroring the model's shape; each field exposes its state as signals. Changing the input updates the signal; setting the signal updates the input.

## 4. Why does Angular need it?

Reactive Forms predate signals: their state lives in Observables and mutable control objects, which doesn't fit OnPush/zoneless templates well (remember the `FieldError` bridge in [note 2](02-validation.md)). Signal Forms make every piece of form state a signal, so templates, `computed()` values and resources react naturally — with less code and full typing.

## 5. How it works internally

```mermaid
flowchart LR
  M[model = signal({ email, password })] <--> F[form(model, schema)]
  F --> FE[loginForm.email: FieldTree]
  FE --> S[loginForm.email(): FieldState<br/>value · valid · touched · errors · pending · disabled]
  S <--> I["&lt;input [formField]='loginForm.email'&gt;"]
```

- The schema function runs once; rules inside it are **reactive**: a validator that reads `valueOf(other)` re-runs when `other` changes.
- `[formField]` is the `FormField` directive. It two-way binds the input and syncs `required`, `disabled`, `readonly` attributes from the field's rules.
- Async validators are built on resources, so they're cancelled when the value changes.

## 6. Syntax and API reference

```typescript
import {
  form, FormField, FormRoot, submit,
  required, email, minLength, maxLength, min, max, pattern, minDate, maxDate,
  validate, validateAsync, validateHttp, debounce,
  disabled, readonly, hidden, applyEach, applyWhen, schema, apply,
} from '@angular/forms/signals';
```

| Task | Code |
|---|---|
| Create | `f = form(this.model, p => { required(p.email); })` |
| Bind | `<input [formField]="f.email" />` (import `FormField`) |
| Field state | `f.email().value()`, `.valid()`, `.invalid()`, `.touched()`, `.dirty()`, `.errors()`, `.pending()`, `.disabled()` |
| Specific error (v22) | `f.email().getError('required')` |
| Form state | `f().valid()`, `f().submitting()`, `f().errorSummary()` |
| Built-in rule with message | `minLength(p.password, 8, { message: 'Use at least 8 characters' })` |
| Conditional rule (v22 `when`) | `required(p.notes, { when: ({ valueOf }) => valueOf(p.categoryId) === 'finance' })` |
| Custom rule | `validate(p.x, ({ value, valueOf }) => bad ? { kind: 'x', message: '…' } : undefined)` |
| Debounce updates | `debounce(p.q, 300)` or `debounce(p.password, 'blur')` |
| Arrays | `applyEach(p.tags, tag => maxLength(tag, 30))` |
| Reusable schema | `const urlRules = schema<string \| null>(u => pattern(u, /^https?:\/\//));` then `apply(p.url, urlRules)` |
| Submit | `await submit(f, async field => { … return errors or undefined })` |
| Mark touched | `f().markAsTouched()` (v22: also marks descendants) |

## 7. Smallest working example — Myspce login

```typescript
// features/auth/login-page.ts
import { Component, inject, signal } from '@angular/core';
import { Router } from '@angular/router';
import { form, FormField, required, email, minLength, submit } from '@angular/forms/signals';
import { AuthStore } from '../../core/auth/auth-store';

@Component({
  selector: 'app-login-page',
  imports: [FormField],
  templateUrl: './login-page.html',
})
export default class LoginPage {
  private auth = inject(AuthStore);
  private router = inject(Router);

  protected model = signal({ email: '', password: '' });
  protected loginForm = form(this.model, p => {
    required(p.email, { message: 'Email is required.' });
    email(p.email, { message: 'Enter a valid email address.' });
    required(p.password, { message: 'Password is required.' });
    minLength(p.password, 8, { message: 'Password is at least 8 characters.' });
  });

  async onSubmit(event: Event) {
    event.preventDefault();
    await submit(this.loginForm, async field => {
      try {
        await this.auth.login(field().value());
        await this.router.navigateByUrl('/dashboard');
        return undefined;
      } catch {
        return { kind: 'invalidCredentials', message: 'Email or password is incorrect.' };   // attaches to the whole form
      }
    });
  }
}
```

```html
<!-- features/auth/login-page.html -->
<main class="grid min-h-dvh place-items-center bg-slate-950 p-4">
  <form (submit)="onSubmit($event)" novalidate class="w-full max-w-sm space-y-4 rounded-2xl bg-slate-900 p-6 text-slate-100">
    <h1 class="text-xl font-semibold">Sign in to MySpace</h1>

    @let em = loginForm.email();
    <div>
      <label for="email" class="text-sm">Email</label>
      <input id="email" type="email" autocomplete="username" [formField]="loginForm.email"
             class="mt-1 w-full rounded-lg bg-slate-800 px-3 py-2"
             [attr.aria-invalid]="em.touched() && em.invalid()" aria-describedby="email-errors" />
      @if (em.touched() && em.invalid()) {
        <ul id="email-errors" class="mt-1 text-sm text-rose-400">
          @for (e of em.errors(); track e.kind) { <li>{{ e.message }}</li> }
        </ul>
      }
    </div>

    @let pw = loginForm.password();
    <div>
      <label for="password" class="text-sm">Password</label>
      <input id="password" type="password" autocomplete="current-password" [formField]="loginForm.password"
             class="mt-1 w-full rounded-lg bg-slate-800 px-3 py-2"
             [attr.aria-invalid]="pw.touched() && pw.invalid()" aria-describedby="password-errors" />
      @if (pw.touched() && pw.invalid()) {
        <ul id="password-errors" class="mt-1 text-sm text-rose-400">
          @for (e of pw.errors(); track e.kind) { <li>{{ e.message }}</li> }
        </ul>
      }
    </div>

    @for (e of loginForm().errors(); track e.kind) {
      <p role="alert" class="rounded-lg bg-rose-950 p-2 text-sm text-rose-200">{{ e.message }}</p>
    }

    <button type="submit" class="w-full rounded-lg bg-blue-600 py-2 font-medium disabled:opacity-50"
            [disabled]="loginForm().submitting()">
      {{ loginForm().submitting() ? 'Signing in…' : 'Sign in' }}
    </button>
    <a href="/api/v1/auth/google" class="block w-full rounded-lg border border-slate-700 py-2 text-center">Continue with Google</a>
  </form>
</main>
```

## 8. Step-by-step

1. `model` is a plain writable signal — the single source of truth.
2. The schema adds rules with messages; no template validator attributes.
3. `[formField]` binds each input and keeps the signal in sync.
4. `submit()` marks fields touched, runs validation, and only calls the action when valid (by default it still submits while async validators are pending; see `ignoreValidators`). It resolves `true` on success, `false` if invalid or the action returned errors.
5. Returning an error object without a field target attaches it to the submitted (root) field, shown via `loginForm().errors()`.
6. `submitting()` is `true` while the action runs.

The Google link goes to the backend's OAuth start endpoint (`GET /auth/google`) — a full page navigation, not an XHR.

## 9. Myspce real-world example — credential form with Signal Forms

Compare with the Reactive Forms version in [note 1](01-reactive-forms-core.md).

```typescript
// features/credentials/components/credential-form-signal.ts
import { Component, computed, input, linkedSignal, output } from '@angular/core';
import { form, FormField, FormRoot, required, maxLength, minLength, pattern, validateHttp } from '@angular/forms/signals';
import { CreateCredentialDto } from '../data-access/credentials-api';
import { Category, CredentialSummary } from '../../../core/models';
import { PasswordInput } from '../../../shared/forms/password-input';

type CredentialModel = Omit<CreateCredentialDto, 'url' | 'notes' | 'categoryId'> & { url: string; notes: string; categoryId: string };

const empty = (): CredentialModel => ({ name: '', url: '', username: '', password: '', categoryId: '', notes: '' });

@Component({
  selector: 'app-credential-form-signal',
  imports: [FormField, FormRoot, PasswordInput],
  templateUrl: './credential-form-signal.html',
})
export class CredentialFormSignal {
  categories = input.required<Category[]>();
  initial = input<CredentialSummary | null>(null);
  submitted = output<CreateCredentialDto>();

  protected isEdit = computed(() => !!this.initial());

  // the model resets whenever a different credential is opened
  protected model = linkedSignal<CredentialModel>(() => {
    const c = this.initial();
    return c ? { name: c.name, url: c.url ?? '', username: c.username, password: '', categoryId: c.category?.id ?? '', notes: '' } : empty();
  });

  protected credForm = form(this.model, p => {
    required(p.name, { message: 'Name is required.' });
    maxLength(p.name, 80);
    pattern(p.url, /^(https?:\/\/\S+)?$/, { message: 'Start with http:// or https://' });
    required(p.username, { message: 'Username is required.' });
    required(p.password, { message: 'Password is required.', when: () => !this.isEdit() });
    minLength(p.password, 8, { message: 'Use at least 8 characters.' });
    maxLength(p.notes, 500);

    // async uniqueness check against the planned API (debounced, cancelled on change)
    // (option names below follow the v21/v22 API — confirm `onSuccess`/`onError` in the API reference)
    validateHttp(p.name, {
      request: ({ value }) => (value().trim().length > 1 ? `/api/v1/credentials?q=${encodeURIComponent(value().trim())}&limit=5` : undefined),
      onSuccess: (res: { data: CredentialSummary[] }, { value }) =>
        res.data.some(c => c.name.toLowerCase() === value().trim().toLowerCase() && c.id !== this.initial()?.id)
          ? { kind: 'taken', message: 'You already have a credential with this name.' }
          : undefined,
      onError: () => undefined,          // don't block saving if the check fails
      debounce: 400,
    });
  }, {
    submission: {
      action: async field => {
        const v = field().value();
        this.submitted.emit({ ...v, url: v.url || null, notes: v.notes || null, categoryId: v.categoryId || null });
        return undefined;
      },
      onInvalid: field => field().errorSummary()[0]?.fieldTree().focusBoundControl(),
    },
  });
}
```

```html
<!-- features/credentials/components/credential-form-signal.html -->
<form [formRoot]="credForm" class="space-y-4">
  @let name = credForm.name();
  <label class="block text-sm">Name
    <input [formField]="credForm.name" class="mt-1 w-full rounded-lg border px-3 py-2" />
  </label>
  @if (name.pending()) { <p class="text-xs text-slate-500">Checking…</p> }
  @if (name.touched() && name.getError('taken'); as err) { <p class="text-sm text-rose-600">{{ err.message }}</p> }
  @else if (name.touched() && name.getError('required'); as err) { <p class="text-sm text-rose-600">{{ err.message }}</p> }

  <label class="block text-sm">Website URL
    <input type="url" [formField]="credForm.url" class="mt-1 w-full rounded-lg border px-3 py-2" />
  </label>

  <label class="block text-sm">Username
    <input [formField]="credForm.username" autocomplete="off" class="mt-1 w-full rounded-lg border px-3 py-2" />
  </label>

  <label class="block text-sm" for="cred-pass">{{ isEdit() ? 'New password (optional)' : 'Password' }}</label>
  <app-password-input inputId="cred-pass" [formField]="credForm.password" autocomplete="new-password" [showStrength]="true" />

  <label class="block text-sm">Category
    <select [formField]="credForm.categoryId" class="mt-1 w-full rounded-lg border px-3 py-2">
      <option value="">None</option>
      @for (c of categories(); track c.id) { <option [value]="c.id">{{ c.name }}</option> }
    </select>
  </label>

  <button type="submit" class="rounded-lg bg-blue-600 px-4 py-2 text-white" [disabled]="credForm().submitting()">
    {{ isEdit() ? 'Save changes' : 'Add credential' }}
  </button>
</form>
```

Notes:

- `FormRoot` (`[formRoot]`) sets `novalidate`, prevents default submit and calls the form's `submission` options.
- `PasswordInput` is the **ControlValueAccessor** from [note 3](03-dynamic-and-reusable-forms.md); v22 lets CVAs work with `[formField]`, so you don't have to rewrite existing controls.
- Empty strings in the model (instead of `null`) keep native inputs simple; convert to `null` when building the DTO.
- `validateHttp` uses `httpResource`, so it goes through your interceptors (auth header included).

## 10. Advanced example — cross-field rules, arrays, conditional state, custom controls

### Cross-field: confirm password

```typescript
protected pwForm = form(signal({ current: '', next: '', confirm: '' }), p => {
  required(p.current);
  required(p.next);
  minLength(p.next, 12);
  validate(p.confirm, ({ value, valueOf }) =>
    value() && value() !== valueOf(p.next) ? { kind: 'mismatch', message: "Passwords don't match." } : undefined,
  );
});
```

Because the validator reads `valueOf(p.next)`, editing **either** field re-validates `confirm` — the problem we couldn't solve cleanly with template-driven forms.

### Arrays: note tags

```typescript
protected noteModel = signal({ title: '', tags: [] as string[] });
protected noteForm = form(this.noteModel, p => {
  required(p.title);
  maxLength(p.title, 200);
  applyEach(p.tags, tag => { required(tag); maxLength(tag, 30); });
});

addTag(t: string) { this.noteModel.update(m => ({ ...m, tags: [...m.tags, t] })); }
removeTag(i: number) { this.noteModel.update(m => ({ ...m, tags: m.tags.filter((_, j) => j !== i) })); }
```

```html
@for (tag of noteForm.tags; track $index; let i = $index) {
  <input [formField]="tag" /> <button type="button" (click)="removeTag(i)">×</button>
}
```

(Iterating a FieldTree array yields child field trees.)

### Conditional disabled/hidden

```typescript
disabled(p.reminderMinutes, { when: ({ valueOf }) => !valueOf(p.dueAt) });   // no due date → no reminder
hidden(p.goalId, { when: ({ valueOf }) => valueOf(p.kind) !== 'goal-task' });
```

### Custom control with `FormValueControl` (v22 shape)

```typescript
// shared/forms/priority-picker.ts
import { Component, input, model, output } from '@angular/core';
import type { FormValueControl } from '@angular/forms/signals';

type Priority = 'low' | 'medium' | 'high';

@Component({
  selector: 'app-priority-picker',
  template: `
    <div role="radiogroup" aria-label="Priority" class="flex gap-2">
      @for (p of options; track p) {
        <button type="button" role="radio" [attr.aria-checked]="value() === p" [disabled]="disabled()"
                class="rounded-lg px-3 py-1 text-sm" [class]="value() === p ? 'bg-blue-600 text-white' : 'bg-slate-100 dark:bg-slate-800'"
                (click)="value.set(p); touch.emit()">{{ p }}</button>
      }
    </div>
  `,
})
export class PriorityPicker implements FormValueControl<Priority> {
  readonly value = model<Priority>('medium');   // required by FormValueControl
  readonly disabled = input(false);              // optional state inputs Angular fills in
  readonly touched = input(false);               // v22: input…
  readonly touch = output<void>();               // …plus an output to mark touched
  protected readonly options: Priority[] = ['low', 'medium', 'high'];
}
```

`<app-priority-picker [formField]="taskForm.priority" />` — no providers, no `writeValue`. v22 also makes `FormValueControl` components usable with Reactive and template-driven forms.

## 11. Visual explanation

```text
Reactive Forms                          Signal Forms
FormGroup (owns state) ─► template      signal (you own state) ─► form() ─► template
valueChanges (Observable)               model() / field().value() (signals)
validators on controls                  schema function with rules
status = 'VALID' | …                    valid(), invalid(), pending() signals
CVA for custom controls                 FormValueControl (model + inputs/outputs)
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Passing a value instead of a signal to `form()` | `form(signal({...}))` — the model must be writable |
| Forgetting to import `FormField` | `[formField]` isn't recognised |
| Mutating `model().tags.push(x)` | `model.update(m => ({ ...m, tags: [...m.tags, x] }))` |
| Expecting reactive conditions in plain `if` inside the schema | the schema runs once — use `when` / `applyWhen` |
| Showing errors before touch | `touched() && invalid()` |
| `<select multiple>` with `[formField]` | not supported yet — use a custom control |
| Old v21 `touched = model()` in custom controls | v22: `touched` input + `touch` output |

## 13. Debugging guide

- Print `{{ loginForm().value() | json }}` and `{{ loginForm().errorSummary() | json }}` temporarily.
- A rule doesn't re-run: it must read the other field via `valueOf`/`stateOf` inside the rule function.
- Async validator never resolves: check the `request` returns `undefined` for "skip"; check the Network tab.
- Template type errors on `[formField]`: the field's value type must match the control (e.g. `number` for `type="number"`).

## 14. Best practices

- Signal Forms for all new Myspce forms; keep Reactive Forms where they already work.
- Messages in the schema, rendering in templates (or one shared error component reading `errors()`).
- Use `linkedSignal` for "edit existing entity" models so the form resets when the entity changes.
- Put submission logic in `submission.action` with `[formRoot]` for consistent behaviour.

## 15. Angular vs React

Signal Forms ≈ React Hook Form with a schema resolver — but the state is reactive signals, not refs, so derived UI (submit button state, conditional fields) needs no `watch()` calls.

## 16. Testing

No DOM needed:

```typescript
it('validates and submits the login form', async () => {
  const login = vi.fn().mockResolvedValue(undefined);
  TestBed.configureTestingModule({ providers: [provideRouter([]), { provide: AuthStore, useValue: { login } }] });
  const page = TestBed.createComponent(LoginPage).componentInstance as any;

  expect(page.loginForm().valid()).toBe(false);
  page.model.set({ email: 'pranjal@example.com', password: 'S3cret!pass' });
  expect(page.loginForm().valid()).toBe(true);

  await page.onSubmit(new Event('submit'));
  expect(login).toHaveBeenCalledWith({ email: 'pranjal@example.com', password: 'S3cret!pass' });
});

it('flags mismatched passwords when either field changes', () => {
  const model = signal({ next: 'Abcdefgh123!', confirm: 'Abcdefgh123!' });
  const f = TestBed.runInInjectionContext(() =>
    form(model, p => validate(p.confirm, ({ value, valueOf }) => (value() !== valueOf(p.next) ? { kind: 'mismatch' } : undefined))),
  );
  expect(f.confirm().valid()).toBe(true);
  model.update(m => ({ ...m, next: 'Changed!123' }));
  expect(f.confirm().getError('mismatch')).toBeTruthy();
});
```

`form()` must be created in an injection context (it uses DI internally), hence `runInInjectionContext` outside components.

## 17. Practice questions

1. Build the Task form (title, notes, dueAt, priority via `PriorityPicker`, category) with Signal Forms.
2. Rebuild the Profile form from Module 11 with Signal Forms and `[formRoot]`.
3. Return a field-targeted server error (`fieldTree: field.email`) from a submit action.
4. When would you keep a form on Reactive Forms?

## 18. Interview questions

- **Beginner — What is the model in Signal Forms?** A writable signal you own; `form()` wraps it with field state and validation.
- **Intermediate — How are cross-field validators re-run?** Rules are reactive; a validator that reads another field with `valueOf` re-runs when that field changes.
- **Advanced — How do custom controls integrate?** Implement `FormValueControl<T>` with a `value` model (and optional state inputs/outputs); `[formField]` binds it. Existing CVAs keep working in v22.

## 19. Revision notes

`form(signal, schema, { submission })` · `[formField]` · field state signals · rules: `required`, `email`, `minLength`, `pattern`, `minDate`… with `{ message, when }` · `validate` / `validateHttp` · `applyEach` · `submit()` / `[formRoot]` · errors `{ kind, message }` · `getError()` · `FormValueControl`.

## 20. Completion checklist

- [ ] Login and credential forms built with Signal Forms.
- [ ] Cross-field and async validation work.
- [ ] `PriorityPicker` custom control works with `[formField]`.
- [ ] Forms tested without the DOM.
