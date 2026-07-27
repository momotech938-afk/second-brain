# Validation (Reactive Forms)

Back to [Module 12](README.md) · Previous: [Reactive Forms core](01-reactive-forms-core.md) · Next: [Dynamic and reusable forms](03-dynamic-and-reusable-forms.md)

## 1. Learning objectives

- Use built-in validators and write custom sync validators.
- Write async validators (server checks) with debouncing.
- Validate across fields and conditionally.
- Show accessible, well-timed messages from one place.
- Map server errors onto controls.

## 2. Prerequisites

[Reactive Forms core](01-reactive-forms-core.md).

## 3. What is it?

A validator is a function `(control) => ValidationErrors | null`. Async validators return `Observable`/`Promise` of the same. Errors are a map: `{ minlength: { requiredLength: 8, actualLength: 3 } }`.

## 4. Why does Angular need it?

Validation gives immediate feedback and prevents useless API calls. **It does not replace server validation** — the NestJS DTOs must validate again.

## 5. How it works internally

On every value change, Angular runs sync validators; if all pass, it runs async validators (status becomes `PENDING`). A group's validators run after its children's. Status bubbles up: one invalid child makes the group invalid.

## 6. Syntax and API reference

| Built-in | Error key |
|---|---|
| `Validators.required` / `requiredTrue` | `required` |
| `Validators.email` | `email` |
| `Validators.minLength(n)` / `maxLength(n)` | `minlength` / `maxlength` |
| `Validators.min(n)` / `max(n)` | `min` / `max` |
| `Validators.pattern(re)` | `pattern` |
| `Validators.compose([...])` | combined |

```typescript
control.addValidators(v); control.removeValidators(v); control.setValidators([...]); control.clearValidators();
control.updateValueAndValidity();   // re-run after changing validators
control.setErrors({ server: 'Taken' });   // set manually (e.g. server errors)
control.hasError('required'); control.getError('minlength')?.requiredLength;
```

## 7. Smallest working example — a custom sync validator

```typescript
// shared/forms/validators.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export const noWhitespace: ValidatorFn = (c: AbstractControl<string | null>): ValidationErrors | null =>
  c.value && c.value.trim().length === 0 ? { whitespace: true } : null;

export function strongPassword(min = 12): ValidatorFn {
  return (c: AbstractControl<string | null>) => {
    const v = c.value ?? '';
    if (!v) return null;                                    // let `required` handle empties
    const missing = [
      v.length < min && 'length',
      !/[A-Z]/.test(v) && 'uppercase',
      !/[a-z]/.test(v) && 'lowercase',
      !/\d/.test(v) && 'number',
      !/[^A-Za-z0-9]/.test(v) && 'symbol',
    ].filter(Boolean) as string[];
    return missing.length ? { strongPassword: { missing } } : null;
  };
}
```

## 8. Step-by-step

1. Return `null` for valid, an object for invalid.
2. Ignore empty values so `required` controls the "empty" message (single responsibility).
3. Put details in the error object (`missing`) so the UI can explain precisely.

## 9. Myspce real-world example — change-password form with cross-field validation

```typescript
// features/settings/tabs/security-tab.ts
import { Component, inject, signal } from '@angular/core';
import { AbstractControl, NonNullableFormBuilder, ReactiveFormsModule, ValidationErrors, Validators } from '@angular/forms';
import { strongPassword } from '../../../shared/forms/validators';
import { FieldError } from '../../../shared/forms/field-error';

function passwordsMatch(group: AbstractControl): ValidationErrors | null {
  const next = group.get('next')?.value;
  const confirm = group.get('confirm')?.value;
  return next && confirm && next !== confirm ? { mismatch: true } : null;
}

@Component({
  selector: 'app-security-tab',
  imports: [ReactiveFormsModule, FieldError],
  templateUrl: './security-tab.html',
})
export default class SecurityTab {
  private fb = inject(NonNullableFormBuilder);
  protected saving = signal(false);

  protected form = this.fb.group(
    {
      current: ['', Validators.required],
      next: ['', [Validators.required, strongPassword(12)]],
      confirm: ['', Validators.required],
    },
    { validators: [passwordsMatch] },
  );

  async save() {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    this.saving.set(true);
    try {
      // await firstValueFrom(this.auth.changePassword(this.form.getRawValue()));
      this.form.reset();
    } catch (e) {
      applyServerErrors(this.form, e);     // see section 10
    } finally {
      this.saving.set(false);
    }
  }
}
```

```html
<!-- features/settings/tabs/security-tab.html -->
<form [formGroup]="form" (ngSubmit)="save()" novalidate class="max-w-md space-y-4">
  <label class="block">Current password
    <input type="password" formControlName="current" autocomplete="current-password" class="mt-1 w-full rounded-lg border px-3 py-2" />
  </label>
  <app-field-error [control]="form.controls.current" />

  <label class="block">New password
    <input type="password" formControlName="next" autocomplete="new-password" class="mt-1 w-full rounded-lg border px-3 py-2" />
  </label>
  <app-field-error [control]="form.controls.next" />

  <label class="block">Confirm new password
    <input type="password" formControlName="confirm" autocomplete="new-password" class="mt-1 w-full rounded-lg border px-3 py-2" />
  </label>
  @if (form.hasError('mismatch') && form.controls.confirm.touched) {
    <p class="text-sm text-rose-600" role="alert">Passwords don't match.</p>
  }

  <button type="submit" class="rounded-lg bg-blue-600 px-4 py-2 text-white" [disabled]="saving()">Change password</button>
</form>
```

### One component for all error messages

```typescript
// shared/forms/field-error.ts
import { Component, computed, effect, input, signal } from '@angular/core';
import { AbstractControl } from '@angular/forms';

const MESSAGES: Record<string, (e: any) => string> = {
  required: () => 'This field is required.',
  email: () => 'Enter a valid email address.',
  minlength: e => `Use at least ${e.requiredLength} characters.`,
  maxlength: e => `Use at most ${e.requiredLength} characters.`,
  pattern: () => 'The format is not valid.',
  whitespace: () => 'Cannot be only spaces.',
  strongPassword: e => `Add: ${e.missing.join(', ')}.`,
  taken: () => 'That name is already used.',
  server: e => String(e),
};

@Component({
  selector: 'app-field-error',
  template: `
    @if (message(); as m) {
      <p class="mt-1 text-sm text-rose-600" role="alert">{{ m }}</p>
    }
  `,
})
export class FieldError {
  control = input.required<AbstractControl>();

  /** Bumped on every control event (value, status, touched, pristine) so the computed re-runs. */
  private version = signal(0);

  constructor() {
    // Bridge: Reactive Forms state is not signal-based, so subscribe to the control's events.
    effect(onCleanup => {
      const sub = this.control().events.subscribe(() => this.version.update(v => v + 1));
      onCleanup(() => sub.unsubscribe());
    });
  }

  protected message = computed(() => {
    this.version();
    const c = this.control();
    if (!c.errors || !c.touched) return null;
    const [key, value] = Object.entries(c.errors)[0]!;
    return MESSAGES[key]?.(value) ?? 'Invalid value.';
  });
}
```

Why the bridge is needed: with OnPush (the v22 default), a child component only refreshes when its inputs, events inside it, or signals it reads change. Typing and blurring happen in the **parent's** template, and the `control` input keeps the same reference, so without the subscription `FieldError` would never update. This friction is exactly what Signal Forms remove — field state there is already signals ([note 4](04-signal-forms.md)).

## 10. Advanced example — async validator and server errors

### Async: "credential name already exists"

```typescript
// shared/forms/async-validators.ts
import { AsyncValidatorFn } from '@angular/forms';
import { of, timer, switchMap, map, catchError, first } from 'rxjs';

export function uniqueCredentialName(api: CredentialsApi, currentId?: string): AsyncValidatorFn {
  return control => {
    const name = (control.value ?? '').trim();
    if (!name) return of(null);
    return timer(400).pipe(                       // debounce: restart on each value change
      switchMap(() => api.list({ q: name })),
      map(page => page.data.some(c => c.name.toLowerCase() === name.toLowerCase() && c.id !== currentId) ? { taken: true } : null),
      catchError(() => of(null)),                 // don't block the form if the check fails
      first(),
    );
  };
}

// usage
name: ['', { validators: [Validators.required], asyncValidators: [uniqueCredentialName(inject(CredentialsApi))], updateOn: 'blur' }],
```

Status is `PENDING` while it runs — show "Checking…" and disable submit while `form.pending`.

### Mapping server validation errors

The planned API returns `{ statusCode, message, ... }`. If NestJS returns field errors (e.g. `{ errors: { email: 'already in use' } }`), map them:

```typescript
// shared/forms/apply-server-errors.ts
import { HttpErrorResponse } from '@angular/common/http';
import { FormGroup } from '@angular/forms';

export function applyServerErrors(form: FormGroup, err: unknown): boolean {
  if (!(err instanceof HttpErrorResponse) || err.status !== 400) return false;
  const fieldErrors: Record<string, string> = err.error?.errors ?? {};
  let applied = false;
  for (const [field, msg] of Object.entries(fieldErrors)) {
    const c = form.get(field);
    if (c) { c.setErrors({ ...c.errors, server: msg }); c.markAsTouched(); applied = true; }
  }
  return applied;
}
```

(The exact error body shape depends on the final NestJS implementation — an unresolved question in the [project context](../MYSPCE_PROJECT_CONTEXT.md#7-unresolved-questions).)

### Conditional validation

```typescript
// "notes required when category is Finance"
this.form.controls.categoryId.valueChanges.pipe(takeUntilDestroyed()).subscribe(cat => {
  const notes = this.form.controls.notes;
  cat === financeId ? notes.addValidators(Validators.required) : notes.removeValidators(Validators.required);
  notes.updateValueAndValidity({ emitEvent: false });
});
```

`removeValidators` needs the **same function reference** that was added — `Validators.required` is a stable reference, custom factories are not (store them in a constant).

## 11. Visual explanation

```text
value change → sync validators → all pass? → async validators (PENDING) → VALID / INVALID
                                     │ no
                                     └─► INVALID (async skipped)
group validators run after children → group.errors (e.g. mismatch)
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Showing errors before interaction | gate by `touched` or form submitted |
| Cross-field validator on the child control | put it on the group |
| Async validator firing on every keystroke | `timer` debounce + `updateOn: 'blur'` |
| `removeValidators(strongPassword(12))` | new reference each call — keep one instance |
| Forgetting `updateValueAndValidity()` after changing validators | errors don't update |
| Client-only validation | always validate in NestJS DTOs too |

## 13. Debugging guide

- `form.errors` vs `control.errors`: group-level errors live on the group.
- Stuck in `PENDING`: async validator Observable never completes — add `first()`/`take(1)`.
- `ExpressionChangedAfterItHasBeenChecked` when setting errors in hooks — set them in event handlers.

## 14. Best practices

- One message map; one place to render errors.
- Validators are pure functions; test them alone.
- Prefer `updateOn: 'blur'` for expensive or async checks.
- Mirror server rules (lengths, patterns) but trust the server's verdict.

## 15. Angular vs React

Similar to React Hook Form's `validate` / resolver functions. Angular validators are attached to controls and can be added/removed at runtime.

## 16. Testing

```typescript
it.each([
  ['', null],
  ['short', { strongPassword: { missing: ['length', 'uppercase', 'number', 'symbol'] } }],
  ['Str0ng!Password', null],
])('strongPassword(%s)', (value, expected) => {
  expect(strongPassword(12)(new FormControl(value))).toEqual(expected);
});

it('flags mismatched passwords on the group', () => {
  const g = new FormGroup({ next: new FormControl('Abc123!xyzQW'), confirm: new FormControl('nope') }, { validators: passwordsMatch });
  expect(g.hasError('mismatch')).toBe(true);
});
```

## 17. Practice questions

1. Add an async "email already used" check to a registration form (if multi-user is ever added).
2. Write `dateNotInPast` for task due dates.
3. Show "Checking…" while the async validator runs.

## 18. Interview questions

- **Q: When do async validators run?** Only after all sync validators pass; the control is `PENDING` meanwhile.
- **Q: Where do cross-field validators go?** On the common parent `FormGroup`.
- **Q: Why still validate on the server?** Client validation can be bypassed; the server is the source of truth.

## 19. Revision notes

`ValidatorFn` returns errors or null · empties left to `required` · group validators for cross-field · async with debounce + `first()` · `setErrors` for server errors · `addValidators`/`removeValidators` + `updateValueAndValidity`.

## 20. Completion checklist

- [ ] Change-password form with strong-password and mismatch rules.
- [ ] Async unique-name validator with "Checking…".
- [ ] Validators unit-tested.
