# Custom Pipes

Back to [Module 06](README.md) · Previous: [Built-in pipes](01-built-in-pipes.md) · Next module: [07 Services and DI](../07-services-and-dependency-injection/README.md)

## 1. Learning objectives

- Write typed, null-safe custom pipes.
- Explain pure vs impure pipes and their performance impact.
- Choose correctly between a pipe, a component method, and `computed()`.
- Unit-test pipes.

## 2. Prerequisites

[Built-in pipes](01-built-in-pipes.md), [TypeScript generics](../00-prerequisites/03-typescript-advanced.md).

## 3. What is it?

A class decorated with `@Pipe({ name })` implementing `PipeTransform`. Its `transform` method receives the piped value and any arguments.

## 4. Why does Angular need it?

Myspce needs formatting Angular doesn't ship: file sizes ("1.2 MB"), relative time ("Edited 2 hours ago"), initials, masked text. As pipes they're reusable in any template and memoised.

## 5. How it works internally

- **Pure (default):** Angular caches the last input/args per binding and calls `transform` only when one changes (`===`). A pure pipe instance can be shared.
- **Impure (`pure: false`):** `transform` runs on **every** change-detection check of that view. Needed only when the output depends on something other than the inputs (time, mutable objects). Expensive; use rarely.

Mutating an array passed to a pure pipe (`list.push(x)`) won't re-run it — same reference. Immutable updates fix this.

## 6. Syntax and API reference

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'fileSize' })        // standalone by default; pure by default
export class FileSizePipe implements PipeTransform {
  transform(bytes: number | null | undefined, decimals = 1): string { /* … */ }
}
```

Use: import `FileSizePipe` in the component, then `{{ file.sizeBytes | fileSize }}` or `{{ x | fileSize: 2 }}`.

Naming: pipe `name` in camelCase (`fileSize`), class named `FileSizePipe` (pipes keep the `Pipe` suffix in the style guide, because the class name and template name differ).

## 7. Smallest working example — `fileSize`

```typescript
// shared/pipes/file-size.ts
import { Pipe, PipeTransform } from '@angular/core';

const UNITS = ['B', 'KB', 'MB', 'GB', 'TB'] as const;

@Pipe({ name: 'fileSize' })
export class FileSizePipe implements PipeTransform {
  transform(bytes: number | null | undefined, decimals = 1): string {
    if (bytes == null || Number.isNaN(bytes)) return '—';
    if (bytes < 1024) return `${bytes} B`;
    const i = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), UNITS.length - 1);
    const value = bytes / 1024 ** i;
    return `${value.toFixed(value >= 100 ? 0 : decimals)} ${UNITS[i]}`;
  }
}
```

`{{ 1258291 | fileSize }}` → `1.2 MB` (as in the Files design: "PDF · 1.2 MB").

## 8. Step-by-step

1. Null-safe: returns a dash for missing sizes instead of throwing.
2. `Math.log` base 1024 picks the unit.
3. Large values (≥ 100) drop decimals for a cleaner look.

## 9. Myspce real-world example — `timeAgo`

Recent Activity shows "Edited 2 hours ago", "Updated 4 hours ago", "Added 1 day ago".

```typescript
// shared/pipes/time-ago.ts
import { LOCALE_ID, Pipe, PipeTransform, inject } from '@angular/core';

const STEPS: [Intl.RelativeTimeFormatUnit, number][] = [
  ['year', 365 * 24 * 3600],
  ['month', 30 * 24 * 3600],
  ['week', 7 * 24 * 3600],
  ['day', 24 * 3600],
  ['hour', 3600],
  ['minute', 60],
  ['second', 1],
];

@Pipe({ name: 'timeAgo' })
export class TimeAgoPipe implements PipeTransform {
  private rtf = new Intl.RelativeTimeFormat(inject(LOCALE_ID), { numeric: 'auto' });

  /** `now` is passed in so the pipe stays pure; the caller supplies a ticking signal. */
  transform(value: string | Date | null | undefined, now: Date = new Date()): string {
    if (!value) return '';
    const diffSec = Math.round((new Date(value).getTime() - now.getTime()) / 1000);
    for (const [unit, sec] of STEPS) {
      if (Math.abs(diffSec) >= sec || unit === 'second') {
        return this.rtf.format(Math.round(diffSec / sec), unit);
      }
    }
    return '';
  }
}
```

```html
<!-- features/dashboard/components/activity-list.html -->
@for (a of items(); track a.id) {
  <li class="flex gap-3">
    <span class="font-medium">{{ a.summary }}</span>
    <time class="text-xs text-slate-500" [attr.datetime]="a.createdAt">{{ a.verb | titlecase }} {{ a.createdAt | timeAgo: now() }}</time>
  </li>
}
```

```typescript
// activity-list.ts — a signal that ticks every minute
now = toSignal(interval(60_000).pipe(map(() => new Date())), { initialValue: new Date() });
```

**Why pass `now` instead of `pure: false`?** An impure `timeAgo` would recompute every check for every row. Passing a minute-ticking `now` keeps the pipe pure: it recomputes once a minute, for exactly the rows on screen.

## 10. Advanced example — typed generic pipe

```typescript
// shared/pipes/pluck.ts
@Pipe({ name: 'pluck' })
export class PluckPipe implements PipeTransform {
  transform<T, K extends keyof T>(items: readonly T[] | null | undefined, key: K): T[K][] {
    return items?.map(i => i[key]) ?? [];
  }
}
```

`{{ notes() | pluck: 'title' | slice: 0 : 3 }}` — strict templates know the result is `string[]`.

### Pipe vs method vs `computed()`

| Need | Use | Why |
|---|---|---|
| Format a value, reusable across templates | **pipe** | memoised, reusable |
| Derived state used in class logic or multiple bindings | **`computed()`** | cached, testable, available to the class |
| One-off formatting in one template | `computed()` or a pipe | avoid template method calls |
| `{{ format(x) }}` method call | ❌ usually | runs on every check of the view |

Method calls in templates are acceptable only for trivial work; with OnPush + signals views are checked less often, but a pipe or `computed()` is still cleaner.

## 11. Visual explanation

```text
Pure pipe:   input changes? ──no──► reuse last result
                  │ yes
                  ▼
              transform()

Impure pipe: every check ──► transform()   (N rows × every check)
```

## 12. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| `pure: false` for "it doesn't update" | slow | find the mutation, update immutably, or pass the changing value as an argument |
| `transform(value: any)` | loses template typing | type inputs and outputs |
| Throwing on `null` | crashes during loading | return a fallback |
| Injecting HTTP into a pipe to fetch data | hidden requests per binding | fetch in a store; pipe formats only |
| Heavy work in `transform` | slow lists | precompute in `computed()` or the API |

## 13. Debugging guide

- `NG0302: The pipe 'fileSize' could not be found` → add `FileSizePipe` to the component's `imports`.
- Pipe output stale → input mutated in place, or depends on hidden state (time); pass it as an argument.
- Type error on pipe arguments → strict templates check `transform`'s signature; fix the call or the types.

## 14. Best practices

- Pure, typed, null-safe, side-effect free.
- Pass changing context (now, locale) as arguments.
- Keep pipes in `shared/pipes` and test them as plain classes.

## 15. Angular vs React

React would use helper functions plus `useMemo`. Angular pipes are reusable formatters with built-in memoisation and template typing.

## 16. Testing

```typescript
// shared/pipes/file-size.spec.ts
import { FileSizePipe } from './file-size';

describe('FileSizePipe', () => {
  const pipe = new FileSizePipe();
  it.each([
    [null, '—'],
    [512, '512 B'],
    [1536, '1.5 KB'],
    [1258291, '1.2 MB'],
    [524288000, '500 MB'],
  ])('formats %s as %s', (input, expected) => {
    expect(pipe.transform(input as number | null)).toBe(expected);
  });
});
```

`TimeAgoPipe` uses `inject()`, so create it inside an injection context:

```typescript
const pipe = TestBed.runInInjectionContext(() => new TimeAgoPipe());
expect(pipe.transform('2026-10-28T10:00:00Z', new Date('2026-10-28T12:00:00Z'))).toBe('2 hours ago');
```

## 17. Practice questions

1. Write `initials` (name → "PG") and use it in the sidebar avatar.
2. Write `maskSecret` that turns `"intercert@2026"` into `"••••••••"` regardless of length.
3. Why is `timeAgo` with a `now` argument better than an impure pipe?

## 18. Interview questions

- **Q: Pure vs impure?** Pure runs only when inputs change by reference/value; impure runs every check.
- **Q: Why doesn't my pure pipe update after `array.push`?** The array reference didn't change; pure pipes compare by identity.
- **Q: When is a pipe the wrong choice?** When the logic is state derivation used by the class, or when it would perform side effects like HTTP calls.

## 19. Revision notes

`@Pipe({ name })` · `transform(value, ...args)` · pure by default · typed + null-safe · pass time as an argument · import into components · test as plain classes.

## 20. Completion checklist

- [ ] `fileSize`, `timeAgo`, `initials`, `maskSecret` built and tested.
- [ ] No impure pipes in my code.
- [ ] I can justify pipe vs `computed()` for any formatting need.
