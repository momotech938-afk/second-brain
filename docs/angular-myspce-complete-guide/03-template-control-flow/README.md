# Module 03 — Control Flow and Template Syntax

Showing, hiding and repeating UI: `@if`, `@for`, `@switch`, plus safe expressions and loading/empty/error states.

Back to [README](../README.md) · Previous: [Module 02](../02-components-and-templates/README.md) · Next: [Module 04](../04-component-communication/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [`@if`, `@for`, `@switch`](01-if-for-switch.md) | syntax, `track`, `$index` and friends, `@empty`, exhaustive `@default never`, legacy `*ngIf`/`*ngFor` |
| 2 | [Expressions and loading states](02-expressions-and-loading-states.md) | safe navigation, `??`, optional data, rendering vs CSS hiding, loading/empty/error patterns, template errors |

**Time:** ~2 h reading + 1 h practice.

## Self-assessment checklist

- [ ] I can write `@if / @else if / @else` and alias the result with `as`.
- [ ] I can write `@for` with `track`, `$index`, `$first`, `$last`, `$even`, `$count` and `@empty`.
- [ ] I can explain why `track` is required and what to track by.
- [ ] I can use `@switch` with `@default never;` to get an exhaustiveness check.
- [ ] I know when to remove an element with `@if` vs hide it with a class.
- [ ] I can render loading, empty, error and success states for a list without duplicated markup.
- [ ] I can convert `*ngIf`, `*ngFor` and `[ngSwitch]` to built-in control flow.
- [ ] I can read and fix the common template errors in this module.
