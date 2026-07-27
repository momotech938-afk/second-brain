# Reactive Forms Core

Back to [Module 12](README.md) · Next: [Validation](02-validation.md)

## 1. Learning objectives

- Build typed forms with `FormControl`, `FormGroup`, `FormArray` and `NonNullableFormBuilder`.
- Bind them with `[formGroup]`, `formControlName`, `formGroupName`, `formArrayName`.
- Use `setValue`, `patchValue`, `reset`, `getRawValue`, `disable`/`enable`.
- React to `valueChanges`/`statusChanges` (and the unified `events` stream).
- Submit and reset cleanly.

## 2. Prerequisites

[Template-driven forms](../11-template-driven-forms/01-template-driven-forms.md) (form state vocabulary), [RxJS basics](../14-rxjs/01-observables.md) (helpful).

## 3. What is it?

Forms where the **class** owns the form model as objects (`FormGroup` of `FormControl`s). The template only binds to it. Since v14 they're strongly typed.

## 4. Why does Angular need it?

Complex forms (dynamic fields, cross-field rules, conditional validation, value streams) are easier to build and test when the model is explicit TypeScript, not scattered across the template.

## 5. How it works internally

- `FormControl<T>` holds `value`, `status`, `errors`, `touched`, `dirty`, and exposes `valueChanges`/`statusChanges` Observables.
- `FormGroup` aggregates children; its value is an object; its validity is the combination.
- Directives (`formControlName`) connect DOM elements through a `ControlValueAccessor` (built in for inputs, selects, checkboxes, radios).
- Updates are synchronous: `control.setValue(x)` updates value and validity immediately.

## 6. Syntax and API reference

```typescript
import { FormControl, FormGroup, FormArray, NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

// by hand
const name = new FormControl('', { nonNullable: true, validators: [Validators.required] });  // FormControl<string>
const group = new FormGroup({ name, url: new FormControl<string | null>(null) });

// with the builder (preferred)
const fb = inject(NonNullableFormBuilder);
const form = fb.group({
  name: ['', [Validators.required, Validators.maxLength(80)]],
  url: fb.control<string | null>(null),
  tags: fb.array<string>([]),
});
```

| API | Meaning |
|---|---|
| `form.value` | values of **enabled** controls (Partial type) |
| `form.getRawValue()` | values of **all** controls, including disabled (full type) |
| `setValue(v)` | must provide **every** key (strict) |
| `patchValue(v)` | any subset of keys |
| `reset(v?)` | back to initial value (non-nullable) or `null`; clears touched/dirty |
| `disable()` / `enable()` | disabled controls are excluded from `value` and validation |
| `markAllAsTouched()` | show all errors (use on submit) |
| `valueChanges` / `statusChanges` | Observables |
| `events` | unified stream of value/status/touched/pristine/submit/reset events (v18+) |
| `get('a.b')` / `controls.name` | access a control (prefer typed `controls`) |
| `updateOn: 'blur' \| 'submit'` | when value/validation update |

Template directives (import `ReactiveFormsModule`):

```html
<form [formGroup]="form" (ngSubmit)="save()">
  <input formControlName="name" />
  <div formGroupName="address"><input formControlName="city" /></div>
  <div formArrayName="tags">
    @for (c of tags.controls; track c; let i = $index) { <input [formControlName]="i" /> }
  </div>
</form>
<input [formControl]="searchControl" />   <!-- standalone control, no group -->
```

## 7. Smallest working example

```typescript
// shared/ui/search-control-demo.ts
import { Component } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-search-control-demo',
  imports: [ReactiveFormsModule],
  template: `<input [formControl]="q" placeholder="Search…" /> <p>You typed: {{ value() }}</p>`,
})
export class SearchControlDemo {
  q = new FormControl('', { nonNullable: true });
  value = toSignal(this.q.valueChanges, { initialValue: this.q.value });
}
```

## 8. Step-by-step

1. `nonNullable: true` makes the type `FormControl<string>` and `reset()` return to `''`, not `null`.
2. `[formControl]` binds the input.
3. `toSignal(valueChanges)` turns the stream into a signal so OnPush templates update.

## 9. Myspce real-world example — Add/Edit Credential form

```typescript
// features/credentials/components/credential-form.ts
import { Component, computed, effect, inject, input, output, signal } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { CreateCredentialDto } from '../data-access/credentials-api';
import { Category, CredentialSummary } from '../../../core/models';

@Component({
  selector: 'app-credential-form',
  imports: [ReactiveFormsModule],
  templateUrl: './credential-form.html',
})
export class CredentialForm {
  private fb = inject(NonNullableFormBuilder);

  categories = input.required<Category[]>();
  /** When editing, the existing summary (no password). */
  initial = input<CredentialSummary | null>(null);
  saving = input(false);
  submitted = output<CreateCredentialDto>();
  cancelled = output<void>();

  protected form = this.fb.group({
    name: ['', [Validators.required, Validators.maxLength(80)]],
    url: this.fb.control<string | null>(null, [Validators.pattern(/^https?:\/\/\S+$/)]),
    username: ['', [Validators.required, Validators.maxLength(120)]],
    password: ['', [Validators.required, Validators.minLength(8)]],
    categoryId: this.fb.control<string | null>(null),
    notes: this.fb.control<string | null>(null, [Validators.maxLength(500)]),
  });

  protected isEdit = computed(() => !!this.initial());
  protected showPassword = signal(false);

  constructor() {
    // Fill the form when editing; password stays empty (unchanged unless the user types a new one)
    effect(() => {
      const c = this.initial();
      if (!c) return;
      this.form.reset({ name: c.name, url: c.url, username: c.username, password: '', categoryId: c.category?.id ?? null, notes: null });
      this.form.controls.password.clearValidators();               // optional when editing
      this.form.controls.password.addValidators(Validators.minLength(8));
      this.form.controls.password.updateValueAndValidity();
    });
  }

  protected save() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    const v = this.form.getRawValue();
    this.submitted.emit({ ...v, url: v.url || null, notes: v.notes || null });
  }
}
```

This effect is legitimate: it syncs an input into an imperative API — the `FormGroup`.

```html
<!-- features/credentials/components/credential-form.html -->
<form [formGroup]="form" (ngSubmit)="save()" novalidate class="space-y-4">
  <div>
    <label for="cred-name" class="text-sm font-medium">Name</label>
    <input id="cred-name" formControlName="name" class="mt-1 w-full rounded-lg border px-3 py-2" placeholder="GitHub" />
    @if (form.controls.name.touched && form.controls.name.hasError('required')) {
      <p class="text-sm text-rose-600">Name is required.</p>
    }
  </div>

  <div>
    <label for="cred-url" class="text-sm font-medium">Website URL</label>
    <input id="cred-url" type="url" formControlName="url" class="mt-1 w-full rounded-lg border px-3 py-2" placeholder="https://github.com" />
    @if (form.controls.url.touched && form.controls.url.hasError('pattern')) {
      <p class="text-sm text-rose-600">Start with http:// or https://</p>
    }
  </div>

  <div>
    <label for="cred-user" class="text-sm font-medium">Username or email</label>
    <input id="cred-user" formControlName="username" autocomplete="off" class="mt-1 w-full rounded-lg border px-3 py-2" />
  </div>

  <div>
    <label for="cred-pass" class="text-sm font-medium">{{ isEdit() ? 'New password (leave blank to keep)' : 'Password' }}</label>
    <div class="mt-1 flex gap-2">
      <input id="cred-pass" [type]="showPassword() ? 'text' : 'password'" formControlName="password" autocomplete="new-password"
             class="w-full rounded-lg border px-3 py-2" />
      <button type="button" (click)="showPassword.update(v => !v)" [attr.aria-pressed]="showPassword()">
        {{ showPassword() ? 'Hide' : 'Show' }}
      </button>
    </div>
  </div>

  <div>
    <label for="cred-cat" class="text-sm font-medium">Category</label>
    <select id="cred-cat" formControlName="categoryId" class="mt-1 w-full rounded-lg border px-3 py-2">
      <option [ngValue]="null">None</option>
      @for (c of categories(); track c.id) { <option [ngValue]="c.id">{{ c.name }}</option> }
    </select>
  </div>

  <div class="flex justify-end gap-2">
    <button type="button" (click)="cancelled.emit()">Cancel</button>
    <button type="submit" class="rounded-lg bg-blue-600 px-4 py-2 text-white" [disabled]="saving()">
      {{ saving() ? 'Saving…' : isEdit() ? 'Save changes' : 'Add credential' }}
    </button>
  </div>
</form>
```

Why this design:

- The form component is **presentational**: it emits a typed DTO; the page calls the API.
- The password is never pre-filled when editing — the list doesn't have it, by design.
- `autocomplete="new-password"` stops browsers from autofilling the user's own login into a vault entry.

## 10. Advanced example — reacting to value changes

Auto-fill the name from the URL host if the user hasn't typed a name:

```typescript
constructor() {
  this.form.controls.url.valueChanges.pipe(
    debounceTime(300),
    filter(() => this.form.controls.name.pristine),
    map(url => { try { return new URL(url ?? '').hostname.replace(/^www\./, ''); } catch { return null; } }),
    filter((host): host is string => !!host),
    takeUntilDestroyed(),
  ).subscribe(host => this.form.controls.name.setValue(host[0]!.toUpperCase() + host.slice(1).split('.')[0]));
}
```

Using the unified events stream (v18+):

```typescript
this.form.events.pipe(filter(e => e instanceof FormSubmittedEvent), takeUntilDestroyed()).subscribe(() => track('credential_submit'));
```

## 11. Visual explanation

```text
FormGroup (form)                 template
 ├─ name: FormControl<string>  ◄─► <input formControlName="name">
 ├─ url: FormControl<string|null>
 ├─ password: FormControl<string>
 └─ categoryId: FormControl<string|null> ◄─► <select formControlName="categoryId">
status = combination of children · value = { name, url, … }
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| `FormBuilder` without `nonNullable` → types full of `null` | `NonNullableFormBuilder` |
| `setValue({ name })` missing keys | throws — use `patchValue` |
| Reading `form.value` and losing disabled fields | `getRawValue()` |
| Subscribing to `valueChanges` without teardown | `takeUntilDestroyed()` |
| `formControlName` outside a `[formGroup]` | `NG01050` error — wrap in the group |
| Using `[disabled]` attribute with reactive forms | warning; call `control.disable()` |

## 13. Debugging guide

- `<pre>{{ form.getRawValue() | json }} {{ form.status }}</pre>` temporarily.
- `NG01050 formControlName must be used with a parent formGroup directive`.
- `Cannot find control with name: 'x'` → typo or the control isn't in the group.
- OnPush template not updating after `setValue` from a subscription → read form state through signals (`toSignal(form.statusChanges)`) or update via template events; reactive form state isn't signal-based.

## 14. Best practices

- `NonNullableFormBuilder` + typed `controls` access.
- Forms in presentational components emit DTOs; pages handle API calls.
- Never pre-fill secrets.
- Convert `valueChanges` to signals when the template depends on them.

## 15. Angular vs React

Reactive Forms ≈ React Hook Form's model-first approach, but built in, with explicit control objects and RxJS streams.

## 16. Testing

Reactive forms are testable without the DOM:

```typescript
it('is invalid without a name and emits a cleaned DTO when valid', () => {
  const fixture = TestBed.createComponent(CredentialForm);
  fixture.componentRef.setInput('categories', []);
  const cmp = fixture.componentInstance as any;
  expect(cmp.form.valid).toBe(false);
  cmp.form.setValue({ name: 'GitHub', url: '', username: 'pg', password: 'S3cret!pass', categoryId: null, notes: '' });
  const emitted: unknown[] = [];
  fixture.componentInstance.submitted.subscribe(v => emitted.push(v));
  cmp.save();
  expect(emitted[0]).toEqual({ name: 'GitHub', url: null, username: 'pg', password: 'S3cret!pass', categoryId: null, notes: null });
});
```

Empty strings pass `Validators.pattern` (only `required` rejects empties), so `url: ''` is valid and gets cleaned to `null`.

## 17. Practice questions

1. Build the Task form (title, notes, dueAt, priority radio, category select) with typed controls.
2. Disable `categoryId` until the user types a name. What happens to `form.value`?
3. When would `updateOn: 'blur'` help in the credential form?

## 18. Interview questions

- **Q: `setValue` vs `patchValue`?** `setValue` requires the full shape; `patchValue` accepts a partial object.
- **Q: `value` vs `getRawValue()`?** `value` excludes disabled controls; `getRawValue()` includes them.
- **Q: Why `NonNullableFormBuilder`?** Controls reset to their initial values and their types exclude `null`.

## 19. Revision notes

`NonNullableFormBuilder.group/control/array` · `[formGroup]` + `formControlName` · `setValue`/`patchValue`/`reset` · `getRawValue` · `markAllAsTouched` on submit · `valueChanges` + `takeUntilDestroyed` · `events` stream.

## 20. Completion checklist

- [ ] Credential form works for add and edit.
- [ ] Name auto-fills from URL.
- [ ] Form tested without the DOM.
