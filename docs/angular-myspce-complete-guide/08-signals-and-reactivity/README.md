# Module 08 — Signals and Reactivity

Signals are how modern Angular tracks state and updates the screen. This is the most important module in the guide; take your time.

Back to [README](../README.md) · Previous: [Module 07](../07-services-and-dependency-injection/README.md) · Next: [Module 09](../09-lifecycle-and-change-detection/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Signal basics](01-signal-basics.md) | reactivity fundamentals, `signal()`, `set`/`update`, immutability, equality, read-only signals, signals in templates and services |
| 2 | [`computed()` and `effect()`](02-computed-and-effect.md) | derived state, dependency tracking, glitch-free updates, `untracked`, what effects are (and aren't) for, cleanup |
| 3 | [`linkedSignal()`](03-linked-signal.md) | writable state that depends on other state: resetting selections, pagination, local drafts |
| 4 | [Resources](04-resources.md) | `resource()`, `rxResource()`, `httpResource()` (stable in v22), status, reload, chaining, `debounced()` |
| 5 | [Signals vs RxJS, patterns and testing](05-signals-vs-rxjs-and-patterns.md) | when to use which, interop, filtering/sorting/pagination, common mistakes, debugging, tests |

**Lab:** [Lab 5 — Signal-based UI state](../21-myspce-practice-labs/lab-05-signal-ui-state.md)

**Time:** ~5 h reading + 3 h practice.

## Self-assessment checklist

- [ ] I can explain the push/pull model: what happens when a signal is set and when a `computed` is read.
- [ ] I can update arrays/objects in signals immutably with `update()`.
- [ ] I can expose read-only state from a service with `asReadonly()` or `computed()`.
- [ ] I can write `computed()` values with conditional dependencies and explain why they're glitch-free.
- [ ] I can list three valid uses of `effect()` and three invalid ones.
- [ ] I can use `untracked()` and explain when it's needed.
- [ ] I can use `linkedSignal()` to reset page number when filters change.
- [ ] I can load data with `httpResource()` and show loading/error/empty/ready states.
- [ ] I can choose between `resource`, `rxResource` and `httpResource`.
- [ ] I can convert between Observables and signals with `toSignal`/`toObservable` and know the pitfalls.
- [ ] I can implement signal-based search, filter, sort and pagination for Notes.
- [ ] I can test signals, computed values, effects and resources.
