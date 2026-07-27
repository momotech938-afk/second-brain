# Component Tests

Back to [Module 19](README.md) · Previous: [Testing setup](01-testing-setup.md) · Next: [Service and HTTP tests](03-service-http-tests.md)

## 1. Learning objectives

- Test presentational components through inputs, outputs and DOM.
- Test page components with fake stores.
- Test signals, computed values and effects in components.
- Test directives and pipes in templates.
- Use Angular Testing Library and basic accessibility checks.

## 2. Testing presentational components

```typescript
// shared/ui/chip-filter.spec.ts
import { TestBed } from '@angular/core/testing';
import { ChipFilter } from './chip-filter';

describe('ChipFilter', () => {
  const options = [{ value: 'all', label: 'All' }, { value: 'work', label: 'Work', count: 3 }];

  async function setup(value = 'all') {
    const fixture = TestBed.createComponent(ChipFilter);
    fixture.componentRef.setInput('options', options);
    fixture.componentRef.setInput('value', value);
    await fixture.whenStable();
    const buttons = () => [...fixture.nativeElement.querySelectorAll('button')] as HTMLButtonElement[];
    return { fixture, buttons };
  }

  it('marks the selected chip with aria-checked', async () => {
    const { buttons } = await setup('work');
    expect(buttons().map(b => b.getAttribute('aria-checked'))).toEqual(['false', 'true']);
  });

  it('shows counts when provided', async () => {
    const { buttons } = await setup();
    expect(buttons()[1]!.textContent).toContain('3');
  });

  it('updates the model on click', async () => {
    const { fixture, buttons } = await setup();
    buttons()[1]!.click();
    await fixture.whenStable();
    expect(fixture.componentInstance.value()).toBe('work');
    expect(buttons()[1]!.getAttribute('aria-checked')).toBe('true');
  });
});
```

Patterns:

- A `setup()` helper per spec file.
- Inputs via `setInput` (works for `input()` and `model()`).
- Query by role/attribute/text the way users perceive the UI.

### Testing outputs

```typescript
it('emits favoriteToggled with the note id', async () => {
  const fixture = TestBed.createComponent(NoteCard);
  fixture.componentRef.setInput('note', MOCK_NOTES[0]);
  const spy = vi.fn();
  fixture.componentInstance.favoriteToggled.subscribe(spy);
  await fixture.whenStable();
  fixture.nativeElement.querySelector('[aria-label^="Favourite"]').click();
  expect(spy).toHaveBeenCalledWith('n1');
});
```

## 3. Testing page components with fake stores

```typescript
// features/notes/pages/notes-page.spec.ts
import { signal } from '@angular/core';

function fakeStore(notes = MOCK_NOTES) {
  return {
    category: signal('all'),
    query: signal(''),
    categoryOptions: signal([{ value: 'all', label: 'All', count: notes.length }]),
    visibleNotes: signal(notes),
    toggleFavorite: vi.fn(),
    create: vi.fn(),
  };
}

describe('NotesPage', () => {
  it('renders one card per visible note', async () => {
    TestBed.configureTestingModule({ providers: [provideTestDefaults({ provide: NotesStore, useValue: fakeStore() })] });
    const fixture = TestBed.createComponent(NotesPage);
    await fixture.whenStable();
    expect(fixture.nativeElement.querySelectorAll('app-note-card').length).toBe(2);
  });

  it('shows the empty state when there are no notes', async () => {
    TestBed.configureTestingModule({ providers: [provideTestDefaults({ provide: NotesStore, useValue: fakeStore([]) })] });
    const fixture = TestBed.createComponent(NotesPage);
    await fixture.whenStable();
    expect(fixture.nativeElement.textContent).toContain('No notes yet');
  });
});
```

Because the fake store exposes **signals**, you can change state mid-test (`store.visibleNotes.set([])`) and `await fixture.whenStable()` to see the UI react.

### Shallow-ish rendering

To isolate a page from heavy children, replace them:

```typescript
@Component({ selector: 'app-notes-grid', template: '' })
class FakeNotesGrid { notes = input<Note[]>([]); open = output<string>(); favoriteToggled = output<string>(); }

TestBed.overrideComponent(NotesPage, { remove: { imports: [NotesGrid] }, add: { imports: [FakeNotesGrid] } });
```

Use sparingly — integration with real children catches more bugs.

## 4. Signals, computed and effects in components

```typescript
it('resets pagination when the category changes (linkedSignal)', async () => {
  const store = fakeStore();
  TestBed.configureTestingModule({ providers: [provideTestDefaults({ provide: NotesStore, useValue: store })] });
  const fixture = TestBed.createComponent(NotesPage);
  const page = fixture.componentInstance as any;
  page.page.set(3);
  store.category.set('work');
  expect(page.page()).toBe(1);   // linkedSignal recomputes synchronously on read
});

it('persists theme via effect', () => {
  const setItem = vi.fn();
  TestBed.configureTestingModule({ providers: [{ provide: KV_STORAGE, useValue: { getItem: () => null, setItem, removeItem: vi.fn() } }] });
  const theme = TestBed.inject(ThemeStore);
  theme.mode.set('dark');
  TestBed.tick();
  expect(setItem).toHaveBeenCalledWith('myspce.theme', 'dark');
});
```

## 5. Directives and pipes in templates

Directives need a host component ([Module 05](../05-directives/02-custom-attribute-directives.md#16-testing)). Pipes can be tested as classes; test them in templates only when template integration matters (locale, chaining).

```typescript
@Component({ imports: [CopyToClipboard], template: `<button appCopyToClipboard="secret" (copySucceeded)="ok = true">Copy</button>` })
class Host { ok = false; }

it('copies text', async () => {
  const writeText = vi.fn().mockResolvedValue(undefined);
  Object.assign(navigator, { clipboard: { writeText, readText: vi.fn().mockResolvedValue('secret') } });
  const f = TestBed.createComponent(Host);
  await f.whenStable();
  f.nativeElement.querySelector('button').click();
  await vi.waitFor(() => expect(writeText).toHaveBeenCalledWith('secret'));
  expect(f.componentInstance.ok).toBe(true);
});
```

## 6. Angular Testing Library (optional)

```bash
npm i -D @testing-library/angular @testing-library/user-event
```

```typescript
import { render, screen } from '@testing-library/angular';
import userEvent from '@testing-library/user-event';

it('filters by chip', async () => {
  await render(ChipFilter, { inputs: { options, value: 'all' } });
  await userEvent.click(screen.getByRole('radio', { name: /work/i }));
  expect(screen.getByRole('radio', { name: /work/i })).toHaveAttribute('aria-checked', 'true');
});
```

`getByRole` doubles as an accessibility check: if you can't find an element by role and name, neither can a screen reader. (`toHaveAttribute` needs `@testing-library/jest-dom` matchers configured for Vitest.)

## 7. Accessibility checks

- Unit level: query by role/label; assert `aria-*` attributes.
- Page level: run axe in Playwright ([E2E note](05-e2e-and-ci.md)).

## 8. Common mistakes

| Mistake | Fix |
|---|---|
| Testing private implementation (`component['cache']`) | test behaviour through inputs/outputs/DOM or public signals |
| Snapshot tests of entire templates | brittle; assert specific, meaningful output |
| Forgetting `await whenStable()` after clicks | assertions see old DOM |
| Mocking everything, including child components | integration tests give more confidence |

## 9. Practice

1. Test `EmptyState` (title, hint, action output).
2. Test `CredentialRow` emits `copyRequested` with the id and never renders a password.
3. Test `Tabs` keyboard navigation with ArrowRight/ArrowLeft.

## 10. Interview questions

- **Q: How do you set a signal input in a test?** `fixture.componentRef.setInput('name', value)`.
- **Q: How do you test a page that depends on a store?** Provide a fake store (with signals) in `providers` and assert the rendered output as you change its signals.

## 11. Completion checklist

- [ ] Specs for `ChipFilter`, `NoteCard`, `EmptyState`, `NotesPage`.
- [ ] At least one test using Testing Library roles.
