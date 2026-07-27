# Module 11 — Template-Driven Forms

The simplest of Angular's three form systems. You'll mostly use Signal Forms or Reactive Forms in Myspce, but template-driven forms teach form fundamentals (state, validation, touched/dirty) and appear in many existing apps.

Back to [README](../README.md) · Previous: [Module 10](../10-routing/README.md) · Next: [Module 12](../12-reactive-forms/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Template-driven forms](01-template-driven-forms.md) | form fundamentals, `FormsModule`, `ngModel`, `ngForm`, validators, custom validator directives, error messages, submission, dynamic fields, reset, accessibility, when to use |

**Time:** ~2 h reading + 1 h practice.

## Which form system? (decide here, learn the others in Module 12)

| | Template-driven | Reactive Forms | Signal Forms (stable v22) |
|---|---|---|---|
| Model lives in | the template (`ngModel`) | the class (`FormGroup`) | a **signal** you own |
| Typing | weak | strong (typed forms) | strong |
| Validation | directives in HTML | functions in TS | schema functions in TS |
| Dynamic fields | awkward | `FormArray` | arrays in the model |
| Testability | needs DOM | class-only tests | class-only tests |
| Best for | tiny forms, legacy code | existing apps, complex RxJS-driven forms | **new Myspce forms** |

## Self-assessment checklist

- [ ] I can explain value, validity, touched, dirty, pristine and pending.
- [ ] I can build a form with `ngModel`, `ngForm` and `(ngSubmit)`.
- [ ] I can use `required`, `email`, `minlength`, `maxlength`, `pattern` and show messages only after touch/submit.
- [ ] I can write a custom validator directive.
- [ ] I can reset a form and disable submit while saving.
- [ ] I can make a form accessible (labels, `aria-invalid`, `aria-describedby`, focus on first error).
- [ ] I can say when template-driven forms are (and aren't) appropriate.
