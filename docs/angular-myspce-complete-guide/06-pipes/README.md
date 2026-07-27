# Module 06 — Pipes

Pipes format values in templates: dates, numbers, file sizes, "2 hours ago".

Back to [README](../README.md) · Previous: [Module 05](../05-directives/README.md) · Next: [Module 07](../07-services-and-dependency-injection/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Built-in pipes](01-built-in-pipes.md) | `date`, `number`/`decimal`, `currency`, `percent`, case pipes, `slice`, `json`, `keyvalue`, `async`, i18n pipes, locale setup, parameters and chaining |
| 2 | [Custom pipes](02-custom-pipes.md) | `@Pipe`, pure vs impure, typed transforms, null handling, pipes vs methods vs `computed`, performance, testing |

**Time:** ~2 h reading + 1 h practice.

## Self-assessment checklist

- [ ] I can format dates, numbers, currency (INR) and percentages with parameters.
- [ ] I can register the `en-IN` locale and explain what changes.
- [ ] I can chain pipes and pass multiple arguments.
- [ ] I can explain pure vs impure pipes and why most pipes should be pure.
- [ ] I can write a typed custom pipe that handles `null`/`undefined`.
- [ ] I can decide between a pipe, a component method and a `computed()`.
- [ ] I can unit-test a pipe without TestBed.
- [ ] I built `fileSize`, `timeAgo` and `initials` pipes for Myspce.
