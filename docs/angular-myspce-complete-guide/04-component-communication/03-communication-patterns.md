# Component Communication Patterns

Back to [Module 04](README.md) · Previous: [Outputs and `model()`](02-outputs-and-model.md) · Next module: [05 Directives](../05-directives/README.md)

## 1. Learning objectives

- Choose the right channel for any two components that need to talk.
- Share state between siblings and distant components with a signal service.
- Use the router as a communication channel (URL state).
- Avoid the classic anti-patterns (event buses, deep prop drilling, mutable shared objects).

## 2. Prerequisites

[Inputs](01-inputs.md), [Outputs and `model()`](02-outputs-and-model.md). Services are covered properly in [Module 07](../07-services-and-dependency-injection/README.md); a preview is enough here.

## 3. The decision table

| Relationship | Use | Myspce example |
|---|---|---|
| Parent → child | `input()` | `NotesPage` → `NoteCard [note]` |
| Child → parent | `output()` | `NoteCard (favoriteToggled)` → `NotesPage` |
| Parent ⇄ child value | `model()` | `ChipFilter [(value)]` |
| Siblings under one parent | lift state to the parent, pass down | `SearchBox` and `NotesGrid` share the parent's `query` |
| Far apart / across routes | shared **signal service** | Topbar's theme toggle ↔ Settings page ↔ `ThemeStore` |
| Anything that should survive refresh / be shareable | **URL** (route/query params) | `/notes?category=work` |
| Parent calling a child method | template ref or `viewChild()` (sparingly) | `confirm.open()` on a modal |
| Truly global events (rare) | a service with a signal or `Subject` | "session expired" broadcast |

## 4. Why does this matter?

Most Angular codebases that feel "messy" have communication problems: values copied between components and drifting out of sync, events passed through five layers, or a global event bus nobody can trace.

## 5. How it works — lifting state up

```mermaid
flowchart TB
  P[NotesPage<br/>query = signal('')] -->|"[(value)]"| S[SearchBox]
  P -->|"[notes]=filtered()"| G[NotesGrid]
```

The parent owns `query`. `SearchBox` writes it via `model()`, `NotesGrid` reads the derived list. Siblings never reference each other.

## 6. Shared signal service (preview)

```typescript
// core/theme/theme-store.ts
import { Service, computed, effect, signal } from '@angular/core';

export type ThemeMode = 'light' | 'dark' | 'system';

@Service()
export class ThemeStore {
  readonly mode = signal<ThemeMode>(this.readSaved());
  private readonly systemDark = signal(matchMedia('(prefers-color-scheme: dark)').matches);
  readonly isDark = computed(() => this.mode() === 'dark' || (this.mode() === 'system' && this.systemDark()));

  constructor() {
    matchMedia('(prefers-color-scheme: dark)').addEventListener('change', e => this.systemDark.set(e.matches));
    effect(() => {
      document.documentElement.classList.toggle('dark', this.isDark());
      try { localStorage.setItem('myspce.theme', this.mode()); } catch { /* storage blocked */ }
    });
  }

  toggle() { this.mode.set(this.isDark() ? 'light' : 'dark'); }

  private readSaved(): ThemeMode {
    try {
      const v = localStorage.getItem('myspce.theme');
      return v === 'light' || v === 'dark' || v === 'system' ? v : 'system';
    } catch {
      return 'system';
    }
  }
}
```

```typescript
// core/layout/topbar.ts
export class Topbar { protected theme = inject(ThemeStore); }
// template: <button type="button" (click)="theme.toggle()" [attr.aria-label]="theme.isDark() ? 'Switch to light' : 'Switch to dark'">…</button>

// features/settings/pages/appearance-tab.ts
export class AppearanceTab { protected theme = inject(ThemeStore); }
// template: <app-segmented [options]="['light','dark','system']" [(value)]="theme.mode" />
```

Topbar and Settings never know about each other; both talk to `ThemeStore`. (Browser-only APIs like `matchMedia` are fine because Myspce is client-rendered; see [Module 18](../18-ssr-and-hydration/02-platform-aware-code.md) for SSR-safe versions.)

## 7. The URL as shared state

Filters that users might bookmark or share belong in the URL:

```typescript
// features/notes/pages/notes-page.ts — with withComponentInputBinding()
export class NotesPage {
  private router = inject(Router);
  category = input<string>('all');   // from ?category=work

  setCategory(c: string) {
    this.router.navigate([], { queryParams: { category: c === 'all' ? null : c }, queryParamsHandling: 'merge' });
  }
}
```

Refresh keeps the filter; the back button undoes it. Details in [Module 10](../10-routing/02-params-and-navigation.md).

## 8. Step-by-step: choosing a channel

1. Does a parent own the data? → inputs/outputs/model.
2. Do two siblings need it? → lift it to their common parent.
3. Are they far apart, or on different routes? → a service.
4. Should it survive refresh or be linkable? → the URL.
5. Is it a one-off imperative action on a child (open, focus, scroll)? → a template ref.

## 9. Myspce real-world example — "Add Credential" from the dashboard

The dashboard quick action "Add Credential" should open the add dialog on the Credentials page.

**Option A (good):** route with intent in the URL: `routerLink="/credentials" [queryParams]="{ new: 1 }"`. `CredentialsPage` reads `new` as an input and opens the dialog, then clears the param. Works with refresh and deep links.

**Option B (fine):** a `CredentialsUi` service with `requestAdd = signal(0)` that the page watches. Works, but nothing in the URL explains why the dialog is open.

**Option C (avoid):** a global event bus `bus.emit('open-add-credential')`. Untyped, untraceable, easy to break.

## 10. Advanced example — typed component APIs with `viewChild`

When a parent really must call a child:

```typescript
// shared/ui/confirm-dialog.ts
export class ConfirmDialog {
  private result: ((ok: boolean) => void) | null = null;
  protected open = signal(false);
  title = input('Are you sure?');

  ask(): Promise<boolean> {
    this.open.set(true);
    return new Promise(resolve => (this.result = resolve));
  }
  protected answer(ok: boolean) {
    this.open.set(false);
    this.result?.(ok);
    this.result = null;
  }
}
```

```typescript
// parent
private confirm = viewChild.required(ConfirmDialog);
async deleteNote(id: string) {
  if (await this.confirm().ask()) this.store.remove(id);
}
```

The public method `ask()` is the component's typed API. A dialog **service** (Module 16) scales better when many pages need confirms.

## 11. Visual explanation

```text
             ┌──────── URL (?category=work) ────────┐
             ▼                                      ▲
   NotesPage (owns query, reads category) ── navigate()
     │ inputs ▼   ▲ outputs/model
   SearchBox  NotesGrid → NoteCard
             ╲        ╱
              ThemeStore / AuthStore (services, any distance)
```

## 12. Common mistakes

| Anti-pattern | Problem | Fix |
|---|---|---|
| Global `EventBus` with string event names | untyped, hidden coupling | typed service signals or URL |
| Passing a callback function as an input | hides the event in the parent API | use `output()` |
| Sibling A injects sibling B's component | tight coupling, DI errors | lift state or use a service |
| Copying store state into a component signal | drift | read the store's signal directly |
| Using a service for parent–child only data | over-engineering | inputs/outputs |

## 13. Debugging guide

- "Where does this value come from?" Angular DevTools → select component → see inputs and injected services → follow the chain.
- State out of sync between two components → you have two sources of truth; delete one.

## 14. Best practices

- Smallest scope that works: inputs → parent state → service → global.
- One source of truth per piece of state.
- URL for anything navigational.

## 15. Angular vs React

Same principles as "lifting state up" and "context for distant components". Angular's DI services replace Context + custom hooks; Angular's built-in router query params replace `useSearchParams`.

## 16. Testing

Test services in isolation (`TestBed.inject(ThemeStore)`), and test pages with a **fake** store provided in `providers` so you control state.

## 17. Practice questions

1. Choose a channel for each: theme toggle, Notes search text, "unsaved changes" warning, current user's avatar in topbar and settings, selected calendar month.
2. Implement Option A for "Add Credential".
3. Replace a callback-function input with an output.

## 18. Interview questions

- **Q: How do siblings communicate?** Through state lifted to a common parent, or through a shared service.
- **Q: Why prefer the URL for filters?** It survives refresh, supports back/forward and deep links, and documents the state.
- **Q: What's wrong with an event bus?** It creates invisible, untyped coupling and makes data flow hard to trace and test.

## 19. Revision notes

inputs ↓ · outputs ↑ · model ⇅ · siblings: lift · far: service · linkable: URL · imperative: ref (sparingly).

## 20. Completion checklist

- [ ] `ThemeStore` shared by Topbar and Settings.
- [ ] Notes category lives in the URL.
- [ ] I can justify the channel for every piece of Myspce v1 state.
