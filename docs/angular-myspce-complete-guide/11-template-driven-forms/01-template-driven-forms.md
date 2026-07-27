# Template-Driven Forms

Back to [Module 11](README.md) · Next module: [12 Reactive and Signal Forms](../12-reactive-forms/README.md)

## 1. Learning objectives

- Understand form state: value, status, touched, dirty, pristine, submitted.
- Build forms with `FormsModule`, `ngModel` and `ngForm`.
- Validate with built-in and custom validator directives.
- Show accessible error messages at the right time.
- Submit, reset and handle dynamic fields.

## 2. Prerequisites

[Data binding](../02-components-and-templates/02-data-binding.md), [Directives](../05-directives/README.md).

## 3. What is it?

Forms where the **template** declares the controls. Each `ngModel` creates a `FormControl` behind the scenes; the `<form>` gets an `NgForm` directive that groups them.

## 4. Why does Angular need it?

Raw `<input>` elements don't track touched/dirty/valid state or aggregate validity. Angular's forms layer does, so you can show errors only after the user interacts and disable submit while invalid.

## 5. How it works internally

- `FormsModule` adds `NgForm` to every `<form>` automatically and provides `ngModel`.
- `[(ngModel)]="model.email"` with a `name` attribute registers a control named `email` in the parent `NgForm`.
- Validator attributes (`required`, `email`, `minlength`) are directives that add validator functions to the control.
- Control creation is **asynchronous** (next tick), which makes template-driven forms slightly harder to test than reactive ones.

### Form state vocabulary (same in all three form systems)

| State | Meaning |
|---|---|
| `value` | current value |
| `valid` / `invalid` | passes all validators? |
| `pending` | async validators running |
| `touched` / `untouched` | user focused and left the field |
| `dirty` / `pristine` | user changed the value |
| `submitted` (form) | submit was attempted |
| `errors` | `{ required: true, minlength: { requiredLength: 8, actualLength: 3 } }` |

Angular also adds CSS classes: `ng-valid`, `ng-invalid`, `ng-touched`, `ng-dirty`, `ng-pristine`, `ng-pending`.

## 6. Syntax and API reference

```html
<form #f="ngForm" (ngSubmit)="save(f)" novalidate>
  <input name="email" type="email" [(ngModel)]="model.email" required email #email="ngModel" />
  @if (email.invalid && (email.touched || f.submitted)) {
    @if (email.errors?.['required']) { <p>Email is required</p> }
    @if (email.errors?.['email']) { <p>Enter a valid email</p> }
  }
  <button type="submit" [disabled]="f.invalid || saving()">Save</button>
</form>
```

| Piece | Purpose |
|---|---|
| `#f="ngForm"` | reference to the form directive (`f.value`, `f.valid`, `f.submitted`, `f.resetForm()`) |
| `(ngSubmit)` | submit handler; prevents page reload |
| `name="…"` | **required** with `ngModel` inside a form — the control's key |
| `#x="ngModel"` | reference to one control |
| `[ngModelOptions]="{ updateOn: 'blur' }"` | update value/validation on blur or submit |
| `[ngModelOptions]="{ standalone: true }"` | not registered in the parent form |
| `ngModelGroup="address"` | nested group |

Built-in validator attributes: `required`, `email`, `minlength`, `maxlength`, `min`, `max`, `pattern`.

`novalidate` turns off native browser validation bubbles so Angular's messages are the only ones.

## 7. Smallest working example

```typescript
// features/settings/tabs/profile-tab.ts (template-driven version)
import { Component, signal } from '@angular/core';
import { FormsModule, NgForm } from '@angular/forms';

@Component({
  selector: 'app-profile-tab',
  imports: [FormsModule],
  templateUrl: './profile-tab.html',
})
export default class ProfileTab {
  protected model = { name: 'Pranjal Goyal', email: 'pranjal@example.com', bio: 'Keep learning, keep building, keep growing.' };
  protected saving = signal(false);

  async save(form: NgForm) {
    if (form.invalid) return;
    this.saving.set(true);
    try {
      // await firstValueFrom(this.api.updateProfile(form.value));
      form.resetForm(form.value);     // mark pristine with the saved values
    } finally {
      this.saving.set(false);
    }
  }
}
```

```html
<!-- features/settings/tabs/profile-tab.html -->
<form #f="ngForm" (ngSubmit)="save(f)" novalidate class="max-w-lg space-y-4">
  <div>
    <label for="name" class="block text-sm font-medium">Name</label>
    <input id="name" name="name" [(ngModel)]="model.name" required maxlength="80" #name="ngModel"
           class="mt-1 w-full rounded-lg border px-3 py-2"
           [attr.aria-invalid]="name.invalid && (name.touched || f.submitted)"
           aria-describedby="name-err" />
    @if (name.invalid && (name.touched || f.submitted)) {
      <p id="name-err" class="mt-1 text-sm text-rose-600">Name is required.</p>
    }
  </div>

  <div>
    <label for="email" class="block text-sm font-medium">Email</label>
    <input id="email" name="email" type="email" [(ngModel)]="model.email" required email #email="ngModel"
           class="mt-1 w-full rounded-lg border px-3 py-2"
           [attr.aria-invalid]="email.invalid && (email.touched || f.submitted)" aria-describedby="email-err" />
    @if (email.invalid && (email.touched || f.submitted)) {
      <p id="email-err" class="mt-1 text-sm text-rose-600">
        {{ email.errors?.['required'] ? 'Email is required.' : 'Enter a valid email address.' }}
      </p>
    }
  </div>

  <div>
    <label for="bio" class="block text-sm font-medium">Bio</label>
    <textarea id="bio" name="bio" [(ngModel)]="model.bio" maxlength="200" rows="3" class="mt-1 w-full rounded-lg border px-3 py-2"></textarea>
    <p class="text-xs text-slate-500">{{ model.bio.length }}/200</p>
  </div>

  <button type="submit" class="rounded-lg bg-blue-600 px-4 py-2 text-white disabled:opacity-50" [disabled]="saving() || f.pristine">
    {{ saving() ? 'Saving…' : 'Save changes' }}
  </button>
</form>
```

## 8. Step-by-step

1. `FormsModule` enables `ngModel`/`ngForm`.
2. Each input has `name`, `[(ngModel)]` and validators.
3. Errors appear only after the user touched the field **or** tried to submit.
4. `aria-invalid` + `aria-describedby` connect errors to inputs for screen readers.
5. `resetForm(form.value)` marks the form pristine after saving, which disables the button until the next change.

Note: `model` is a plain object mutated by `ngModel`. With OnPush that's fine here because changes come from template events in this component. If other code must react to the model, prefer Signal Forms.

## 9. Myspce real-world example — custom validator directive (match password)

Used in a "change password" form (Security tab):

```typescript
// shared/forms/match-field.validator.ts
import { Directive, input } from '@angular/core';
import { AbstractControl, NG_VALIDATORS, ValidationErrors, Validator } from '@angular/forms';

@Directive({
  selector: '[appMatchField]',
  providers: [{ provide: NG_VALIDATORS, useExisting: MatchField, multi: true }],
})
export class MatchField implements Validator {
  appMatchField = input.required<string>();   // name of the other control
  validate(control: AbstractControl): ValidationErrors | null {
    const other = control.parent?.get(this.appMatchField());
    return other && control.value !== other.value ? { mismatch: true } : null;
  }
}
```

```html
<input name="newPassword" type="password" [(ngModel)]="m.newPassword" required minlength="12" #np="ngModel" />
<input name="confirm" type="password" [(ngModel)]="m.confirm" required appMatchField="newPassword" #c="ngModel" />
@if (c.errors?.['mismatch'] && c.touched) { <p class="text-rose-600">Passwords don't match.</p> }
```

Limitation: editing `newPassword` doesn't re-validate `confirm` automatically. That's one reason cross-field validation is easier in Reactive/Signal Forms ([Module 12](../12-reactive-forms/02-validation.md)).

## 10. Advanced example — dynamic fields

```html
@for (tag of model.tags; track $index; let i = $index) {
  <div class="flex gap-2">
    <input [name]="'tag' + i" [(ngModel)]="model.tags[i]" required />
    <button type="button" (click)="model.tags.splice(i, 1)" aria-label="Remove tag">✕</button>
  </div>
}
<button type="button" (click)="model.tags.push('')">+ Add tag</button>
```

It works, but names must be unique, tracking by index is fragile, and the value shape is implicit. For dynamic lists, use `FormArray` or Signal Forms arrays.

## 11. Visual explanation

```text
<form ngForm>                      NgForm (FormGroup)
  <input name="name" ngModel> ───► FormControl 'name'  {value, valid, touched, dirty, errors}
  <input name="email" ngModel> ──► FormControl 'email'
  (ngSubmit) ─────────────────────► save(f)   f.submitted = true
```

## 12. Common mistakes

| Mistake | Error / effect | Fix |
|---|---|---|
| `ngModel` inside a form without `name` | `NG01352: If ngModel is used within a form tag, either the name attribute must be set…` | add `name` |
| Forgetting `FormsModule` | `NG0303: Can't bind to 'ngModel'` | import it |
| Showing errors immediately on load | noisy | gate by `touched \|\| submitted` |
| `(submit)` instead of `(ngSubmit)` | `submitted` not set; page may reload | `(ngSubmit)` |
| Mixing `ngModel` and `formControlName` on one field | errors/undefined behaviour | pick one system |

## 13. Debugging guide

- `<pre>{{ f.value | json }} {{ f.valid }}</pre>` temporarily under the form.
- Check classes on the input in DevTools: `ng-invalid ng-touched` tells you exactly what Angular thinks.
- Tests: wait for `await fixture.whenStable()` after rendering because controls register asynchronously.

## 14. Best practices

- Use template-driven forms only for small, static forms (a single search box, a simple settings form).
- Always `novalidate`, labels, `aria-invalid`, `aria-describedby`.
- Show errors after touch or submit; disable submit while saving, not while invalid (users need to see why).

## 15. Angular vs React

`[(ngModel)]` ≈ a controlled input with `value` + `onChange`, plus automatic touched/dirty/validity tracking that React needs a library (React Hook Form/Formik) for.

## 16. Testing

```typescript
it('shows the required error after submit', async () => {
  const fixture = TestBed.createComponent(ProfileTab);
  await fixture.whenStable();
  const input: HTMLInputElement = fixture.nativeElement.querySelector('#name');
  input.value = '';
  input.dispatchEvent(new Event('input'));
  fixture.nativeElement.querySelector('form').dispatchEvent(new Event('submit'));
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).toContain('Name is required.');
});
```

## 17. Practice questions

1. Add `updateOn: 'blur'` to the email field. What changes for the user?
2. Write an `appNoWhitespace` validator directive.
3. Why does the match-password directive not revalidate automatically?

## 18. Interview questions

- **Q: Difference between touched and dirty?** Touched = the user focused and blurred the field; dirty = the user changed its value.
- **Q: Why is `name` required with `ngModel` in a form?** It's the key used to register the control in the parent `NgForm`.
- **Q: When would you choose template-driven forms?** Small, simple forms or existing code that already uses them.

## 19. Revision notes

`FormsModule` · `[(ngModel)]` + `name` · `#f="ngForm"` · `(ngSubmit)` · validator attributes · errors after touched/submitted · `NG_VALIDATORS` custom validators · `resetForm()`.

## 20. Completion checklist

- [ ] Profile form built with accessible errors.
- [ ] Custom validator directive written.
- [ ] I can explain why Myspce prefers Signal Forms for new forms.
