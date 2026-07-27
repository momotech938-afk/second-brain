# Module 12 — Reactive Forms and Signal Forms

Reactive Forms (mature, typed, used in most existing Angular apps) and **Signal Forms** (signal-based, **stable in v22**, the recommended choice for new Myspce forms). You'll learn both, because you'll meet Reactive Forms at work and use Signal Forms in Myspce.

Back to [README](../README.md) · Previous: [Module 11](../11-template-driven-forms/README.md) · Next: [Module 13](../13-http-and-api-integration/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Reactive Forms core](01-reactive-forms-core.md) | `FormControl`, `FormGroup`, `FormArray`, `FormBuilder`, `NonNullableFormBuilder`, typed forms, `setValue` vs `patchValue`, disabled controls, `valueChanges`/`statusChanges`, submit/reset |
| 2 | [Validation](02-validation.md) | sync/async/custom validators, cross-field and conditional validation, accessible messages, server errors |
| 3 | [Dynamic and reusable forms](03-dynamic-and-reusable-forms.md) | nested groups, `FormArray`, `ControlValueAccessor`, reusable form components, RxJS integration |
| 4 | [Signal Forms](04-signal-forms.md) | `form()`, `[formField]`, schema validators, `validate`, async validation, `submit()`, `[formRoot]`, `FormValueControl`, migration |

**Lab:** [Lab 3 — Forms and reusable form controls](../21-myspce-practice-labs/lab-03-forms-and-controls.md)

**Time:** ~5 h reading + 3 h practice.

## Myspce forms map

| Form | Fields (from designs/plan) | Built with in this module |
|---|---|---|
| Login | email, password | Signal Forms |
| Profile (Settings) | name, email, bio, photo | Signal Forms |
| Add/edit credential | name, URL, username, password, category, notes | Reactive Forms (note 1–3) **and** Signal Forms (note 4) for comparison |
| Change password | current, new, confirm | cross-field validation |
| Task | title, notes, due date/time, priority, category | Signal Forms |
| Note tags | dynamic list | `FormArray` / Signal Forms arrays |

## Self-assessment checklist

- [ ] I can build a typed reactive form with `NonNullableFormBuilder` and bind it with `formControlName`.
- [ ] I can explain `setValue` vs `patchValue` and `value` vs `getRawValue()`.
- [ ] I can write sync, async and cross-field validators, and conditional validation.
- [ ] I can manage a `FormArray` of tags.
- [ ] I can write a `ControlValueAccessor` (password input) that works with `formControlName`.
- [ ] I can build the login form with Signal Forms, including `submit()` and server errors.
- [ ] I can write a custom Signal Forms control with `FormValueControl`.
- [ ] I can show accessible errors after touch or submit in both systems.
- [ ] I can test forms without the DOM.
