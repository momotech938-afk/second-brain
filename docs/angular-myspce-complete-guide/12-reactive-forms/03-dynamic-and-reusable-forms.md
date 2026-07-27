# Dynamic and Reusable Forms

Back to [Module 12](README.md) · Previous: [Validation](02-validation.md) · Next: [Signal Forms](04-signal-forms.md)

## 1. Learning objectives

- Build nested groups and dynamic `FormArray`s.
- Write a custom form control with `ControlValueAccessor` (CVA).
- Build reusable form sections that plug into a parent form.
- Integrate forms with RxJS (autosave, dependent fields).

## 2. Prerequisites

[Reactive Forms core](01-reactive-forms-core.md), [Validation](02-validation.md).

## 3. What is it?

- **Dynamic forms:** the number of controls changes at runtime (tags, checklist items, multiple URLs).
- **Reusable controls:** components that behave like native inputs for any form system (a password input with show/hide and a strength meter).

## 4. Why does Angular need it?

Myspce needs tag lists on notes, a password input used in four places (login, credential, change password, re-auth), and an autosaving note editor.

## 5. How it works internally

- `FormArray<FormControl<string>>` is an ordered list of controls; its value is `string[]`.
- `ControlValueAccessor` is the adapter between a form control and a DOM/UI element: Angular calls `writeValue` to push values in, and you call the registered `onChange`/`onTouched` to push values out.
- `ControlContainer` lets a child component find the parent `FormGroupDirective` so it can bind `formControlName`s inside a parent form.

## 6. Syntax and API reference

```typescript
const tags = fb.array<string>([]);       // FormArray<FormControl<string>>
tags.push(fb.control('angular'));
tags.insert(0, fb.control('aws'));
tags.removeAt(1);
tags.clear();
tags.at(0).value;
tags.length;
tags.setValue(['a', 'b']);   // length must match
```

```typescript
interface ControlValueAccessor {
  writeValue(value: unknown): void;               // model → view
  registerOnChange(fn: (v: unknown) => void): void;   // view → model
  registerOnTouched(fn: () => void): void;
  setDisabledState?(disabled: boolean): void;
}
// provider: { provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => MyControl), multi: true }
```

## 7. Smallest working example — tags `FormArray`

```typescript
// features/notes/components/tag-editor.ts
import { Component, inject, input } from '@angular/core';
import { FormArray, FormControl, NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-tag-editor',
  imports: [ReactiveFormsModule],
  template: `
    <ul class="flex flex-wrap gap-2">
      @for (ctrl of tags().controls; track ctrl; let i = $index) {
        <li class="flex items-center gap-1 rounded-full bg-slate-100 px-2 dark:bg-slate-800">
          #<input [formControl]="ctrl" class="w-20 bg-transparent text-sm" [attr.aria-label]="'Tag ' + (i + 1)" />
          <button type="button" (click)="tags().removeAt(i)" [attr.aria-label]="'Remove tag ' + ctrl.value">×</button>
        </li>
      }
    </ul>
    <input #newTag class="mt-2 rounded border px-2 py-1 text-sm" placeholder="Add tag, press Enter"
           (keydown.enter)="$event.preventDefault(); add(newTag.value); newTag.value = ''" />
  `,
})
export class TagEditor {
  private fb = inject(NonNullableFormBuilder);
  tags = input.required<FormArray<FormControl<string>>>();

  add(raw: string) {
    const tag = raw.trim().toLowerCase();
    if (!tag || this.tags().value.includes(tag) || this.tags().length >= 20) return;
    this.tags().push(this.fb.control(tag, [Validators.maxLength(30)]));
  }
}
```

Parent: `<app-tag-editor [tags]="form.controls.tags" />`.

## 8. Step-by-step

1. The parent owns the `FormArray`; the editor receives it as an input.
2. `track ctrl` uses the control object identity (stable while it exists).
3. Adding/removing changes the array; validity and value update automatically.

## 9. Myspce real-world example — `PasswordInput` with `ControlValueAccessor`

```typescript
// shared/forms/password-input.ts
import { Component, computed, forwardRef, input, signal } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-password-input',
  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => PasswordInput), multi: true }],
  template: `
    <div class="flex items-center gap-2 rounded-lg border px-3 py-2 focus-within:ring-2 focus-within:ring-blue-500"
         [class.opacity-50]="disabled()">
      <input [id]="inputId()" class="min-w-0 flex-1 bg-transparent outline-none" [type]="visible() ? 'text' : 'password'"
             [value]="value()" [disabled]="disabled()" [attr.autocomplete]="autocomplete()"
             (input)="onInput($any($event.target).value)" (blur)="onTouched()" />
      <button type="button" class="text-sm text-slate-500" (click)="visible.update(v => !v)"
              [attr.aria-pressed]="visible()" [attr.aria-controls]="inputId()">
        {{ visible() ? 'Hide' : 'Show' }}
      </button>
    </div>
    @if (showStrength() && value()) {
      <div class="mt-1 h-1 rounded bg-slate-200" aria-hidden="true">
        <div class="h-1 rounded transition-all" [style.width.%]="strength() * 25"
             [class]="strength() < 2 ? 'bg-rose-500' : strength() < 4 ? 'bg-amber-500' : 'bg-emerald-500'"></div>
      </div>
      <p class="sr-only" aria-live="polite">Password strength {{ strength() }} of 4</p>
    }
  `,
})
export class PasswordInput implements ControlValueAccessor {
  inputId = input('password');
  autocomplete = input<'current-password' | 'new-password' | 'off'>('current-password');
  showStrength = input(false);

  protected value = signal('');
  protected disabled = signal(false);
  protected visible = signal(false);
  protected strength = computed(() => {
    const v = this.value();
    return [v.length >= 12, /[A-Z]/.test(v) && /[a-z]/.test(v), /\d/.test(v), /[^A-Za-z0-9]/.test(v)].filter(Boolean).length;
  });

  private onChange: (v: string) => void = () => {};
  protected onTouched: () => void = () => {};

  writeValue(v: string | null): void { this.value.set(v ?? ''); }
  registerOnChange(fn: (v: string) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  setDisabledState(d: boolean): void { this.disabled.set(d); }

  protected onInput(v: string) {
    this.value.set(v);
    this.onChange(v);
  }
}
```

Usage anywhere:

```html
<label for="cred-pass">Password</label>
<app-password-input inputId="cred-pass" formControlName="password" autocomplete="new-password" [showStrength]="true" />
```

Internal state is in **signals** (`value`, `disabled`), so the OnPush component updates when Angular calls `writeValue`/`setDisabledState`. In v22 this CVA also works unchanged inside **Signal Forms** with `[formField]` (CVA compatibility was added for migration).

## 10. Advanced example — reusable form section and autosave

### Section component bound into the parent form

```typescript
// shared/forms/url-fields.ts — renders into whatever parent [formGroup] contains it
import { Component, inject } from '@angular/core';
import { ControlContainer, ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-url-fields',
  imports: [ReactiveFormsModule],
  viewProviders: [{ provide: ControlContainer, useFactory: () => inject(ControlContainer, { skipSelf: true }) }],
  template: `
    <label class="block">Website URL <input formControlName="url" type="url" class="mt-1 w-full rounded-lg border px-3 py-2" /></label>
  `,
})
export class UrlFields {}
```

`viewProviders` re-exposes the parent's `ControlContainer` to this component's template, so `formControlName="url"` resolves to the parent form's `url` control.

### Autosave a note draft

```typescript
// features/notes/pages/note-editor-page.ts (Reactive Forms version)
constructor() {
  this.form.valueChanges.pipe(
    debounceTime(800),
    filter(() => this.form.valid && this.form.dirty),
    switchMap(v => this.api.update(this.note().id, v).pipe(
      tap(() => this.form.markAsPristine()),
      catchError(() => { this.saveError.set(true); return EMPTY; }),
    )),
    takeUntilDestroyed(),
  ).subscribe(() => this.lastSaved.set(new Date()));
}
```

`switchMap` cancels an in-flight save when a newer change arrives — only the latest content is saved. For "never drop a save", use `concatMap` ([Module 14](../14-rxjs/02-operators.md)).

## 11. Visual explanation

```text
FormGroup ──► formControlName="password" ──► NG_VALUE_ACCESSOR (PasswordInput)
   setValue('x') ───────► writeValue('x') ──► value signal ──► <input [value]>
   control.value ◄─────── onChange('xy')  ◄── (input) event
   touched ◄───────────── onTouched()     ◄── (blur) event
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Forgetting `multi: true` on `NG_VALUE_ACCESSOR` | `NG01203: No value accessor for form control` |
| Calling `onChange` inside `writeValue` | causes loops and marks the form dirty — don't |
| Plain fields in a CVA under OnPush | UI doesn't reflect `writeValue`; use signals |
| `track $index` on a removable `FormArray` list | wrong inputs get focus/state; track the control |
| Autosave with `mergeMap` | out-of-order saves; use `switchMap`/`concatMap` |

## 13. Debugging guide

- `NG01203 No value accessor for form control name: 'password'` → the element/component has no CVA, or the CVA isn't provided correctly.
- Section component: `Cannot find control with name` → missing `viewProviders` `ControlContainer` trick, or the parent group lacks that control.

## 14. Best practices

- One reusable control per input type (password, tags, date-time, category picker).
- Keep CVA state in signals.
- Parent owns `FormArray`s; children get them as inputs.
- Use RxJS for autosave/dependent fields, with explicit operator choice.

## 15. Angular vs React

A CVA ≈ a React component implementing `value` + `onChange` + `onBlur` to integrate with React Hook Form's `Controller`. Angular's contract is formalised by the interface and DI token.

## 16. Testing

```typescript
it('propagates typed values and touch', () => {
  const ctrl = new FormControl('');
  @Component({ imports: [PasswordInput, ReactiveFormsModule], template: `<app-password-input [formControl]="ctrl" />` })
  class Host { ctrl = ctrl; }

  const fixture = TestBed.createComponent(Host);
  fixture.detectChanges();
  const input: HTMLInputElement = fixture.nativeElement.querySelector('input');
  input.value = 'S3cret!';
  input.dispatchEvent(new Event('input'));
  input.dispatchEvent(new Event('blur'));
  expect(ctrl.value).toBe('S3cret!');
  expect(ctrl.touched).toBe(true);
});
```

## 17. Practice questions

1. Build a `FormArray` of "useful links" (title + URL groups) for a note.
2. Add a "Generate password" button to `PasswordInput` that calls `onChange` with a random strong password (use `crypto.getRandomValues`).
3. Make autosave show "Saving… / Saved 2 min ago / Couldn't save".

## 18. Interview questions

- **Q: What's a `ControlValueAccessor`?** The bridge between Angular forms and a custom input component: `writeValue` in, `onChange`/`onTouched` out.
- **Q: How do you let a child component use `formControlName` from the parent form?** Re-provide the parent's `ControlContainer` via `viewProviders`.
- **Q: `switchMap` vs `concatMap` for autosave?** `switchMap` keeps only the latest save (cancels earlier); `concatMap` queues every save in order.

## 19. Revision notes

`FormArray` push/removeAt/at · track controls · CVA = `writeValue` + `registerOnChange` + `registerOnTouched` + `setDisabledState` + `NG_VALUE_ACCESSOR` multi · `ControlContainer` in `viewProviders` · autosave with `debounceTime` + `switchMap`.

## 20. Completion checklist

- [ ] `TagEditor` and `PasswordInput` built and reused.
- [ ] A reusable section component works inside a parent form.
- [ ] Autosave works with error state.
