# Built-in Pipes

Back to [Module 06](README.md) · Next: [Custom pipes](02-custom-pipes.md)

## 1. Learning objectives

- Use every common built-in pipe with its parameters.
- Chain pipes and understand evaluation order.
- Configure the app locale for India (`en-IN`) and set defaults.

## 2. Prerequisites

[Data binding](../02-components-and-templates/02-data-binding.md).

## 3. What is it?

A pipe is a template function: `{{ value | pipeName:arg1:arg2 }}`. Built-in pipes live in `@angular/common` and must be imported into each component that uses them (e.g. `imports: [DatePipe, DecimalPipe]`).

## 4. Why does Angular need it?

Formatting belongs in the view, not in your data. The API sends `"2026-10-28T16:30:00Z"`; the UI shows "28 Oct 2026" in one place and "4 hours ago" in another. Pipes keep that logic reusable and out of components.

## 5. How it works internally

A pipe is a class with `transform(value, ...args)`. Built-in pipes are **pure**: Angular calls `transform` only when the input value or an argument changes (by `===`). Results are effectively memoised per binding.

## 6. Syntax and API reference

| Pipe | Class | Example | Output (en-IN, approximate; verify in your app) |
|---|---|---|---|
| `date` | `DatePipe` | `{{ d \| date }}` | locale's `mediumDate` pattern (check yours — it differs between `en-US` and `en-IN`) |
| | | `{{ d \| date: 'd MMM y' }}` | `28 Oct 2026` |
| | | `{{ d \| date: 'EEE, d MMM y · h:mm a' }}` | `Wed, 28 Oct 2026 · 9:42 pm` |
| | | `{{ d \| date: 'shortTime' }}` | `9:42 pm` |
| | | `{{ d \| date: 'medium' : 'Asia/Kolkata' }}` | timezone override |
| `number` | `DecimalPipe` | `{{ 1234567.891 \| number }}` | `12,34,567.891` (Indian grouping) |
| | | `{{ 3.14159 \| number: '1.0-2' }}` | `3.14` (minInt.minFrac-maxFrac) |
| `currency` | `CurrencyPipe` | `{{ 1250 \| currency: 'INR' }}` | `₹1,250.00` |
| | | `{{ 1250 \| currency: 'INR' : 'symbol' : '1.0-0' }}` | `₹1,250` |
| `percent` | `PercentPipe` | `{{ 0.5 \| percent }}` | `50%` |
| `uppercase` / `lowercase` / `titlecase` | | `{{ 'work notes' \| titlecase }}` | `Work Notes` |
| `slice` | `SlicePipe` | `{{ tags \| slice: 0 : 3 }}` | first 3 items (arrays or strings) |
| `json` | `JsonPipe` | `<pre>{{ note \| json }}</pre>` | debugging only |
| `keyvalue` | `KeyValuePipe` | `@for (kv of counts \| keyvalue; track kv.key)` | iterate object entries (sorted by key) |
| `async` | `AsyncPipe` | `{{ user$ \| async }}` | subscribes/unsubscribes to Observables/Promises |
| `i18nPlural` | `I18nPluralPipe` | `{{ n \| i18nPlural: { '=0': 'No notes', '=1': '1 note', other: '# notes' } }}` | pluralisation |
| `i18nSelect` | `I18nSelectPipe` | `{{ role \| i18nSelect: { admin: 'Administrator', other: 'User' } }}` | value → label |

### Date format tokens you'll use

| Token | Meaning | Example |
|---|---|---|
| `d` / `dd` | day | 8 / 08 |
| `EEE` / `EEEE` | weekday | Wed / Wednesday |
| `MMM` / `MMMM` | month | Oct / October |
| `y` | year | 2026 |
| `h` / `hh` / `H` | hour 12h / padded / 24h | 9 / 09 / 21 |
| `mm` | minutes | 05 |
| `a` | am/pm | pm |

## 7. Smallest working example

```typescript
import { Component } from '@angular/core';
import { CurrencyPipe, DatePipe, PercentPipe } from '@angular/common';

@Component({
  selector: 'app-pipe-demo',
  imports: [DatePipe, CurrencyPipe, PercentPipe],
  template: `
    <p>{{ today | date: 'EEE, d MMM y' }}</p>
    <p>{{ 1250 | currency: 'INR' : 'symbol' : '1.0-0' }}</p>
    <p>{{ 0.5 | percent }}</p>
  `,
})
export class PipeDemo {
  today = new Date();
}
```

## 8. Step-by-step

1. Each pipe is imported into the component (standalone).
2. `:` separates arguments.
3. Pipes run only when their input changes.

## 9. Myspce real-world example — locale setup and the Files list

```typescript
// app.config.ts
import { LOCALE_ID } from '@angular/core';
import { DATE_PIPE_DEFAULT_OPTIONS, registerLocaleData } from '@angular/common';
import localeEnIn from '@angular/common/locales/en-IN';

registerLocaleData(localeEnIn);

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: LOCALE_ID, useValue: 'en-IN' },
    { provide: DATE_PIPE_DEFAULT_OPTIONS, useValue: { dateFormat: 'd MMM y', timezone: '+0530' } },
    // …router, http, etc.
  ],
};
```

`registerLocaleData` loads Indian formatting rules (e.g. 12,34,567 grouping, lowercase am/pm). `DATE_PIPE_DEFAULT_OPTIONS` sets the format/timezone used when you write `{{ d | date }}` with no arguments. Users in other timezones would see IST; for a single Indian user that's what you want — otherwise omit `timezone` so the browser's zone is used.

```html
<!-- features/files/components/file-row.html -->
<span class="truncate">{{ file().originalName }}</span>
<span class="text-xs text-slate-500">{{ file().mimeType | uppercase }} · {{ file().sizeBytes | fileSize }}</span>
<time class="text-xs text-slate-500" [attr.datetime]="file().createdAt">{{ file().createdAt | date }}</time>
```

(`fileSize` is a custom pipe — [next note](02-custom-pipes.md).)

## 10. Advanced example — chaining and the `async` pipe

```html
{{ note().title | slice: 0 : 40 | titlecase }}
<!-- evaluates left to right: slice first, then titlecase -->

{{ (summary$ | async)?.notes ?? 0 | number }}
<!-- parentheses group: pipes have lower precedence than ?. and ?? ; pipe binds to the whole left side -->
```

With signals and resources, you'll rarely need `async` in new code. Use it when a library gives you an Observable and you don't want to convert it with `toSignal()`.

## 11. Visual explanation

```text
value ──► slice:0:40 ──► titlecase ──► rendered text
            pure           pure        (each step recomputed only if its input changed)
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Forgetting to import the pipe class | `NG0302: The pipe 'date' could not be found` → add `DatePipe` to `imports` |
| Formatting with `new Date().toLocaleDateString()` in the component | use `date` pipe (consistent, locale-aware) |
| `{{ a ?? b \| number }}` expecting `(a ?? b) \| number` | pipes bind loosely; add parentheses explicitly for clarity |
| `json` pipe left in production templates | debugging only |
| `currency` with no currency code | defaults to USD; always pass `'INR'` (or set `DEFAULT_CURRENCY_CODE`) |

## 13. Debugging guide

- `NG0302` → import the pipe.
- `InvalidPipeArgument: 'Unable to convert "abc" into a date'` → the input isn't a valid date string/number/Date; check the API field.
- Wrong grouping (`1,234,567` instead of `12,34,567`) → locale not registered/provided.

## 14. Best practices

- Set `LOCALE_ID`, `DEFAULT_CURRENCY_CODE` (`'INR'`) and date defaults once in `app.config.ts`.
- Keep API dates as ISO strings; format only in templates.
- Use `<time [attr.datetime]>` for semantic, accessible dates.

## 15. Angular vs React

React has no pipes; you call `format(date, 'd MMM y')` from date-fns or `Intl.DateTimeFormat` inside JSX. Angular's pipes are the built-in, memoised equivalent.

## 16. Testing

Pipes are plain classes; built-in ones need a locale:

```typescript
import { DatePipe } from '@angular/common';
it('formats dates', () => {
  const pipe = new DatePipe('en-US');
  expect(pipe.transform('2026-10-28T00:00:00Z', 'd MMM y', 'UTC')).toBe('28 Oct 2026');
});
```

## 17. Practice questions

1. Show the dashboard date as "Tue, 28 Oct 2026" and time as "09:42 PM".
2. Format a task count with `i18nPlural`: "No tasks", "1 task", "12 tasks".
3. Iterate `{ work: 3, personal: 5 }` with `keyvalue` and render chips.

## 18. Interview questions

- **Q: Why are pipes efficient?** Pure pipes only re-run when their inputs change, so formatting isn't recomputed on every check.
- **Q: How do you change number/date formatting for India?** Register `en-IN` locale data and provide `LOCALE_ID: 'en-IN'`.

## 19. Revision notes

`date:'fmt':'tz'` · `number:'1.0-2'` · `currency:'INR':'symbol':'1.0-0'` · `percent` · case pipes · `slice` · `keyvalue` · `async` · `i18nPlural`/`i18nSelect` · import each pipe · register locale.

## 20. Completion checklist

- [ ] `en-IN` locale and INR default configured.
- [ ] All dates in my practice app go through the `date` pipe.
- [ ] I can write a date format string from memory.
