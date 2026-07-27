# Module 04 — Component Communication

How components pass data and events: signal inputs, outputs, `model()` two-way binding, and when to use a shared service instead.

Back to [README](../README.md) · Previous: [Module 03](../03-template-control-flow/README.md) · Next: [Module 05](../05-directives/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Inputs](01-inputs.md) | `input()`, `input.required()`, transforms, aliases, reacting to changes, `ngOnChanges`, legacy `@Input` |
| 2 | [Outputs and `model()`](02-outputs-and-model.md) | `output()`, payload design, aliases, `outputFromObservable`, `model()`, two-way binding |
| 3 | [Communication patterns](03-communication-patterns.md) | parent↔child, siblings, shared signal services, trade-offs, typed component APIs |

**Time:** ~3 h reading + 2 h practice.

## Self-assessment checklist

- [ ] I can declare required and optional inputs and read them in templates and `computed()`.
- [ ] I can use `booleanAttribute`/`numberAttribute` transforms and write my own transform.
- [ ] I can react to an input change without `ngOnChanges`.
- [ ] I can declare an output, emit a well-designed payload, and handle it in the parent.
- [ ] I can build a component with `model()` and use it with `[(value)]`.
- [ ] I can explain why a child must not mutate an input object.
- [ ] I can make two sibling components share state through a service.
- [ ] I can choose between inputs/outputs, `model()`, a shared service and a router param for a given case.
- [ ] I built `ChipFilter` (with `model()`) and `CredentialRow` (with outputs) for Myspce.
