# Lab 1 — Create the Angular Application Shell

Back to [Labs](README.md) · Solution: [lab-01-solution.md](solutions/lab-01-solution.md) · Next: [Lab 2](lab-02-layout-and-navigation.md)

## 1. Learning objective

Create the `myspce-practice` workspace on Angular 22 with Tailwind v4, the recommended folder structure, environments, a dev proxy, and a minimal root component.

## 2. Prerequisites

[Module 00](../00-prerequisites/README.md), [Module 01](../01-angular-fundamentals/README.md). Node 22.22+/24.13.1+/26 installed.

## 3. Problem statement

You need a clean, consistent starting point that every later lab builds on.

## 4. Expected behaviour

- `ng serve` shows "MySpace" in the browser with Tailwind styles applied.
- Dark mode can be toggled by adding `class="dark"` to `<html>` in DevTools.
- `ng test` passes.
- Folder structure matches [Module 01 note 3](../01-angular-fundamentals/03-project-structure-and-bootstrap.md#recommended-myspce-srcapp-structure).

## 5. Relevant concepts

CLI, workspace files, bootstrap, `app.config.ts`, environments, `API_URL` token, proxy, Tailwind v4.

## 6. Step-by-step implementation

1. Create the workspace:
   ```bash
   node -v
   npx @angular/cli@22 new myspce-practice --style=css --ssr=false
   cd myspce-practice
   ```
2. Install Tailwind v4:
   ```bash
   npm install tailwindcss @tailwindcss/postcss postcss
   ```
   Create `.postcssrc.json`:
   ```json
   { "plugins": { "@tailwindcss/postcss": {} } }
   ```
3. Replace `src/styles.css`:
   ```css
   @import "tailwindcss";
   @custom-variant dark (&:where(.dark, .dark *));

   @theme {
     --color-primary: oklch(0.55 0.2 260);
     --radius-card: 0.75rem;
     --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
   }

   html { color-scheme: light dark; }
   body { @apply bg-slate-50 text-slate-900 antialiased dark:bg-slate-950 dark:text-slate-100; }
   ```
4. Create folders:
   ```bash
   mkdir -p src/app/{core/{auth,bootstrap,http,layout,models,theme},shared/{ui,directives,pipes,utils},features/{dashboard,notes,credentials,tasks,settings},admin}
   ```
5. Environments and the API token:
   ```bash
   ng generate environments
   ```
   Set `apiUrl: '/api/v1'` in both files, and create `src/app/core/http/api-url.token.ts`:
   ```typescript
   import { InjectionToken } from '@angular/core';
   export const API_URL = new InjectionToken<string>('API_URL');
   ```
6. Provide it in `app.config.ts`:
   ```typescript
   import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
   import { provideRouter, withComponentInputBinding } from '@angular/router';
   import { routes } from './app.routes';
   import { API_URL } from './core/http/api-url.token';
   import { environment } from '../environments/environment';

   export const appConfig: ApplicationConfig = {
     providers: [
       provideBrowserGlobalErrorListeners(),
       provideRouter(routes, withComponentInputBinding()),
       { provide: API_URL, useValue: environment.apiUrl },
     ],
   };
   ```
7. Dev proxy `proxy.conf.json` (for when NestJS exists):
   ```json
   { "/api": { "target": "http://localhost:3000", "secure": false, "changeOrigin": true } }
   ```
   and in `angular.json` → `serve.options`: `"proxyConfig": "proxy.conf.json"`.
8. Root component `src/app/app.html`:
   ```html
   <main class="grid min-h-dvh place-items-center">
     <div class="rounded-card bg-white p-8 text-center shadow-sm dark:bg-slate-900">
       <h1 class="text-3xl font-bold text-primary">MySpace</h1>
       <p class="mt-2 text-slate-500">Keep learning, keep building, keep growing.</p>
     </div>
   </main>
   <router-outlet />
   ```
9. Update `app.spec.ts` to assert the heading text, run `ng test --no-watch`, then commit:
   ```bash
   git add -A && git commit -m "chore: lab 1 app shell" && git tag lab-01
   ```

## 7. Complete working code

All code is in the steps above; the full file set (including `app.ts` and `app.spec.ts`) is in the [solution](solutions/lab-01-solution.md).

## 8. Important lines explained

- `@custom-variant dark (...)` — makes `dark:` utilities follow a `.dark` class instead of the OS setting, so the app's theme toggle controls it.
- `@theme { --color-primary … }` — creates `bg-primary`/`text-primary` utilities **and** a CSS variable the admin theme builder can override.
- `withComponentInputBinding()` — enables route params as inputs from day one.
- `API_URL` token — one place to configure the API base; tests can override it.

## 9. Expected output

A centred card with a blue "MySpace" heading. In DevTools, adding `class="dark"` on `<html>` switches to the dark background.

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Tailwind classes have no effect | `.postcssrc.json` missing or misnamed; restart `ng serve` |
| `ng new` fails with Node version error | install a supported Node |
| `NG0201: No provider for InjectionToken API_URL` | missing provider in `app.config.ts` |
| `dark:` follows OS, not the class | missing `@custom-variant` line |

## 11. Test cases

```typescript
// src/app/app.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { App } from './app';

describe('App', () => {
  it('renders the MySpace heading', async () => {
    TestBed.configureTestingModule({ providers: [provideRouter([])] });
    const fixture = TestBed.createComponent(App);
    await fixture.whenStable();
    expect(fixture.nativeElement.querySelector('h1')?.textContent).toContain('MySpace');
  });
});
```

## 12. Independent practice task

Add a **staging** build configuration with its own environment file (`apiUrl: 'https://staging.example/api/v1'`), and an npm script `build:staging`. Verify by building and searching the output for the URL.

## 13. Solution

See [solutions/lab-01-solution.md](solutions/lab-01-solution.md).

## 14. Completion checklist

- [ ] Workspace on Angular 22, `ng version` checked.
- [ ] Tailwind v4 with class-based dark mode.
- [ ] Folder structure created.
- [ ] `API_URL` token provided; proxy configured.
- [ ] Test passes; tagged `lab-01`.
