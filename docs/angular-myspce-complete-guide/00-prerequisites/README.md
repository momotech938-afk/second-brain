# Module 00 — Prerequisites

Angular is a TypeScript framework. Most "Angular is hard" moments are actually TypeScript, async or tooling gaps. This module closes them, using only what Angular really needs.

Back to [README](../README.md) · Next: [Module 01](../01-angular-fundamentals/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [JavaScript for Angular](01-javascript-for-angular.md) | `this`, arrow functions, modules, destructuring, spread, immutability, array methods |
| 2 | [TypeScript fundamentals](02-typescript-fundamentals.md) | types, interfaces, unions, classes, access modifiers, `readonly` |
| 3 | [TypeScript advanced](03-typescript-advanced.md) | generics, narrowing, utility types, decorators, `satisfies`, strictness |
| 4 | [Async and error handling](04-async-and-error-handling.md) | Promises, `async/await`, event loop, error handling, `AbortController` |
| 5 | [DOM, CSS and responsive design](05-dom-css-responsive.md) | events, bubbling, semantic HTML, flex/grid, Tailwind, breakpoints |
| 6 | [npm, Git and DevTools](06-npm-git-devtools.md) | package.json, semver, lockfiles, Git flow, Chrome DevTools, Angular DevTools |

**Time:** ~4 h reading + 2 h practice. If you already use React daily, skim notes 1 and 5 and focus on 2, 3 and 4.

## Self-assessment checklist

Tick each only when you can do it without notes.

- [ ] I can explain why `this` is lost in `setTimeout(this.save)` and fix it with an arrow function.
- [ ] I can update an array of notes immutably (add, update by id, remove by id) with `map`, `filter` and spread.
- [ ] I can write an `interface Note` and a `type NoteStatus = 'draft' | 'saved'` and explain when to use each.
- [ ] I can write a generic `Paginated<T>` type and a generic `findById<T extends { id: string }>()` function.
- [ ] I can narrow `Note | null` safely and explain what `strictNullChecks` protects me from.
- [ ] I can use `Partial`, `Pick`, `Omit`, `Record` and `Readonly` correctly.
- [ ] I can explain what a decorator like `@Component()` does at a high level.
- [ ] I can write an `async` function with `try/catch/finally` and cancel a `fetch` with `AbortController`.
- [ ] I can explain event bubbling and stop it on a "delete" button inside a clickable card.
- [ ] I can build a responsive 1 → 2 → 4 column card grid with Tailwind.
- [ ] I can read `package.json`, explain `^22.1.0` vs `~22.1.0`, and why `package-lock.json` is committed.
- [ ] I can create a feature branch, commit, rebase on `main` and open a pull request.
- [ ] I can find a failing network request and a JavaScript error in Chrome DevTools.
