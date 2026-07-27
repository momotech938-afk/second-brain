# Lab 10 — Search, Filter, Sort and Paginate (URL-synced)

Back to [Labs](README.md) · Solution: [lab-10-solution.md](solutions/lab-10-solution.md) · Previous: [Lab 9](lab-09-protected-routes.md) · Next: [Lab 11](lab-11-dialogs-and-toasts.md)

## 1. Learning objective

Make list state shareable and refresh-safe: the Notes list's search, category, sort and page live in the **URL query string**, the search is debounced, stale requests are cancelled, and the global top-bar search shows grouped results.

## 2. Prerequisites

[Module 10 note 2](../10-routing/02-params-and-navigation.md), [Module 14 note 2](../14-rxjs/02-operators.md), [Module 08 note 5](../08-signals-and-reactivity/05-signals-vs-rxjs-and-patterns.md), [Lab 6](lab-06-typed-api-services.md), [Lab 7](lab-07-ui-states.md).

## 3. Problem statement

Today the filters live in a store signal: reload loses them, Back doesn't undo them, and every keystroke fires a request. Fix all three.

## 4. Expected behaviour

- `/notes?q=aws&category=work&sort=title&page=2` restores exactly that view on reload or when shared.
- Typing "angular" fires **one** request ~300 ms after you stop typing; earlier in-flight requests are cancelled.
- Changing category, sort or search resets `page` to 1. Default values are **omitted** from the URL (`/notes`, not `/notes?category=all&page=1`).
- Browser Back undoes the last filter change. Typing doesn't create a history entry per keystroke (`replaceUrl`).
- Invalid URL values (`page=-3`, `sort=evil`) fall back to defaults.
- Top-bar search (`Ctrl/⌘+K` focuses it): results grouped by Notes, Credentials and Tasks from the planned `GET /search?q=`, with keyboard navigation.

## 5. Relevant concepts

`withComponentInputBinding()` for query params, `Router.navigate` with `queryParams` + `queryParamsHandling: 'merge'`, `replaceUrl`, `toObservable` + `debounceTime` + `distinctUntilChanged`, `switchMap` cancellation, `rxResource`, `httpResource`, listbox/combobox ARIA.

## 6. Step-by-step implementation

1. Define `NotesListParams` (`q`, `category`, `sort: 'updated' | 'title' | 'created'`, `page`) and a `parseNotesParams(raw)` function with validation and defaults.
2. In `NotesPage`, read query params via inputs (`q = input<string>()`, etc.) and compute `params = computed(() => parseNotesParams({...}))`.
3. Write `setParams(patch, { replace })` that navigates with merged query params, drops defaults, and resets `page` when anything other than `page` changes.
4. Search box: keep a local `draft` signal for what's typed; debounce it to the URL (`toObservable(draft).pipe(debounceTime(300), distinctUntilChanged())` → `setParams({ q }, { replace: true })`).
5. Feed the parsed params to the `rxResource` (Lab 6). `rxResource` cancels the previous stream when params change.
6. Add a sort `<select>` (Last updated, Title A–Z, Created).
7. Build `GlobalSearch` in the top bar using `httpResource` on a debounced query, with grouped results and arrow-key navigation.
8. Extend the mock: `sort` support on `/notes`, and `GET /search?q=` returning `{ notes, credentials, tasks }` (top 5 each). The response shape is **proposed**; the endpoint is planned.

## 7. Complete working code

`parseNotesParams`, `NotesPage` changes, the debounced search box wiring, and `GlobalSearch` are in the [solution](solutions/lab-10-solution.md).

## 8. Important lines explained

- `queryParamsHandling: 'merge'` — change one param without wiping the others.
- `{ q: value || null }` — `null` **removes** the param from the URL.
- `replaceUrl: true` for typing — Back goes to the previous *meaningful* state, not each letter.
- `toObservable(draft).pipe(debounceTime(300), distinctUntilChanged())` — signals for state, RxJS for time.
- `parseNotesParams` — the URL is user input; never trust it.

## 9. Expected output

Type "aw", pause, type "s": exactly one `GET /notes?q=aws` (with the mock's debug log). URL shows `/notes?q=aws`. Press Back: the filter clears in one step.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Typing feels laggy / cursor jumps | input bound directly to the URL param; bind to a local `draft` and debounce |
| Page doesn't reset on filter change | `setParams` didn't set `page: null` for non-page changes |
| URL full of defaults | map defaults to `null` before navigating |
| Two requests per change | both an `effect` and the resource fetch; let the resource own fetching |
| Back button needs 7 presses | used `replaceUrl: false` while typing |
| Inputs undefined | `withComponentInputBinding()` missing; or the param name doesn't match the input name |

## 11. Test cases

```typescript
describe('parseNotesParams', () => {
  it('falls back to defaults for invalid values', () => {
    expect(parseNotesParams({ page: '-3', sort: 'evil' })).toEqual({ q: '', category: 'all', sort: 'updated', page: 1 });
  });
  it('keeps valid values', () => {
    expect(parseNotesParams({ q: ' aws ', category: 'work', sort: 'title', page: '2' }))
      .toEqual({ q: 'aws', category: 'work', sort: 'title', page: 2 });
  });
});

it('debounces search into the URL', async () => {
  vi.useFakeTimers();
  const harness = await RouterTestingHarness.create('/notes');
  const input: HTMLInputElement = harness.routeNativeElement!.querySelector('input[type=search]')!;
  for (const ch of ['a', 'aw', 'aws']) { input.value = ch; input.dispatchEvent(new Event('input')); }
  await vi.advanceTimersByTimeAsync(300);
  expect(TestBed.inject(Router).url).toBe('/notes?q=aws');
  vi.useRealTimers();
});
```

## 12. Independent practice task

Apply URL-synced state to **Tasks**: `view` (today | tomorrow | week | upcoming | completed, matching the planned API), `category`, and `priority` (multi-select: `?priority=high&priority=medium`). Then add a **"Load more"** mode for the Credentials list (infinite list instead of pages) that appends pages to a local array and resets when filters change.

## 13. Solution

See [solutions/lab-10-solution.md](solutions/lab-10-solution.md).

## 14. Completion checklist

- [ ] Notes state fully in the URL; defaults omitted; invalid values safe.
- [ ] One debounced request per pause; old ones cancelled.
- [ ] Back/forward behave sensibly.
- [ ] Global search with grouped results and keyboard support.
- [ ] Tasks URL state + Credentials "Load more".
- [ ] Tests pass; tagged `lab-10`.
