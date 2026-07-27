# Setup and the Angular CLI

Back to [Module 01](README.md) · Previous: [Architecture](01-architecture-mental-model.md) · Next: [Project structure and bootstrap](03-project-structure-and-bootstrap.md)

## 1. Learning objectives

- Install a Node version supported by Angular 22.
- Install the Angular CLI and create the Myspce workspace.
- Use generators with the v20+ naming style.
- Run the dev server and understand what it's doing.

## 2. Prerequisites

[npm, Git and DevTools](../00-prerequisites/06-npm-git-devtools.md).

## 3. What is it?

The **Angular CLI** (`ng`) is the command-line tool that creates projects, generates code, runs the dev server, builds, tests and updates Angular. Under the hood it uses **esbuild + Vite** (the `@angular/build` package) for fast builds and a fast dev server.

## 4. Why does Angular need it?

It enforces a consistent structure, wires new files correctly (imports, naming, tests), and runs version migrations. Hand-configuring a bundler for Angular's compiler would be painful.

## 5. Version compatibility (verified)

| Angular | Node.js | TypeScript | RxJS |
|---|---|---|---|
| **22.0.x** | ^22.22.0 \|\| ^24.13.1 \|\| ^26.0.0 | >=6.0.0 <6.1.0 | ^6.5.3 \|\| ^7.4.0 |
| 21.0.x | ^20.19.0 \|\| ^22.12.0 \|\| ^24.0.0 | >=5.9.0 <6.0.0 | ^6.5.3 \|\| ^7.4.0 |

Source: angular.dev version compatibility table. Minor releases (22.1, 22.2…) can widen the TypeScript range; always check the table before upgrading. Node 20 is **not** supported by Angular 22.

## 6. Install and create

```bash
node -v                          # must match the table, e.g. v24.x
npm install -g @angular/cli@22   # global CLI
ng version                       # confirm

ng new myspce
```

`ng new` asks a few questions. Recommended answers for Myspce:

| Prompt | Answer | Why |
|---|---|---|
| Stylesheet format | **CSS** (or Tailwind if offered) | Tailwind v4 is configured from CSS; SCSS is unnecessary with Tailwind |
| Enable SSR / SSG? | **No** | Myspce is a private, logged-in app (assumption A1); SSR adds complexity with no SEO benefit |
| AI tools configuration | your choice | generates best-practice files for coding agents |

Non-interactive version:

```bash
ng new myspce --style=css --ssr=false --skip-git=false
cd myspce
ng serve --open
```

What you get by default in v21+/v22:

- **Standalone** components, **zoneless** change detection, **OnPush by default** (v22).
- **Vitest** as the unit-test runner.
- `@angular/build:application` builder (esbuild + Vite).
- `strictTemplates` enabled.

### Add Tailwind v4 (if `ng new` didn't)

```bash
npm install tailwindcss @tailwindcss/postcss postcss
```

```json
// .postcssrc.json (project root)
{ "plugins": { "@tailwindcss/postcss": {} } }
```

```css
/* src/styles.css */
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

## 7. Generators (v20+ naming)

Since v20 the style guide drops the `.component`/`.service` suffixes. `ng generate component note-card` creates `note-card.ts`, `note-card.html`, `note-card.css`, `note-card.spec.ts` with class `NoteCard`.

```bash
# Components
ng g c core/layout/shell
ng g c core/layout/sidebar
ng g c shared/ui/note-card --inline-style        # styles in the .ts file
ng g c features/notes/pages/notes-page --skip-tests

# Services (v22 generates @Service() by default)
ng g s features/notes/data-access/notes-api
ng g s core/http/api-config --injectable         # @Injectable() if you need other options

# Others
ng g guard core/auth/auth --functional           # CanActivateFn
ng g interceptor core/auth/auth                  # HttpInterceptorFn
ng g pipe shared/pipes/file-size
ng g directive shared/directives/copy-to-clipboard
ng g environments                                # creates src/environments/*
ng g interface core/models/note
```

Useful flags: `--dry-run` (show what would be created), `--skip-tests`, `--inline-template`, `--flat`, `--type=page` (adds a type suffix if your team prefers `notes.page.ts`).

## 8. Step-by-step: what `ng serve` does

1. Reads `angular.json` → `serve` target → `@angular/build:dev-server`.
2. Compiles TypeScript and templates with the Angular compiler.
3. Bundles with esbuild; serves via Vite at `http://localhost:4200`.
4. Watches files; on save, rebuilds only what changed and **hot-replaces** styles and templates (HMR is on by default).
5. Serves everything from memory — nothing is written to `dist/`.

`PORT=4300 ng serve` works in v22 (environment variable takes precedence over `--port`).

## 9. Myspce example — initial workspace commands

```bash
ng new myspce --style=css --ssr=false
cd myspce
npm install tailwindcss @tailwindcss/postcss postcss
ng g environments
ng g c core/layout/shell --skip-tests
ng g c core/layout/sidebar --skip-tests
ng g c core/layout/topbar --skip-tests
ng g c features/dashboard/pages/dashboard-page
ng g c features/notes/pages/notes-page
ng g c shared/ui/page-header
ng g s core/theme/theme-store
git add -A && git commit -m "chore: scaffold myspce workspace"
```

That's exactly Lab 1: [Create the Angular application shell](../21-myspce-practice-labs/lab-01-app-shell.md).

## 10. Advanced — schematics defaults in `angular.json`

Set defaults once so every generated file follows your conventions:

```json
"schematics": {
  "@schematics/angular:component": { "style": "css", "skipTests": false },
  "@schematics/angular:service": { },
  "@schematics/angular:guard": { "functional": true }
}
```

## 11. Visual explanation

```text
ng new ──► workspace (angular.json, package.json, src/)
ng g   ──► files + wiring
ng serve ─► compile → bundle (esbuild) → dev server (Vite) → browser (HMR)
ng build ─► compile → bundle → optimise → dist/myspce/browser
ng test  ─► compile → Vitest
ng update ► bump versions + run migrations
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Node 20 with Angular 22 | Install Node 22.22+, 24.13.1+ or 26 |
| Global CLI v21 creating a v22 project (or vice versa) | `npx @angular/cli@22 new myspce`, or update the global CLI |
| Running `ng` outside the workspace | `cd myspce` first |
| Editing files in `.angular/cache` | Never; delete the folder if builds act strangely |

## 13. Debugging guide

| Error | Cause | Fix |
|---|---|---|
| `The Angular CLI requires a minimum Node.js version of …` | Old Node | Upgrade Node |
| `ERESOLVE unable to resolve dependency tree` | Mixed Angular versions or TS outside the range | Align all `@angular/*` versions; set TS `~6.0.0` |
| `Port 4200 is already in use` | Another `ng serve` | Stop it or use `PORT=4300 ng serve` |
| Weird stale build | Corrupted cache | `rm -rf .angular/cache` and restart |

## 14. Best practices

- Pin Node in `.nvmrc`; use the project's local CLI via `npx ng` or npm scripts in CI.
- Always generate with the CLI; it keeps naming and tests consistent.
- Commit right after `ng new` so you can diff later changes.

## 15. Angular vs React

`ng new` ≈ `npm create vite@latest -- --template react-ts`, but Angular's CLI also generates code, runs migrations (`ng update`) and owns the test runner. React has no equivalent of `ng update` migrations.

## 16. Testing

`ng test` runs Vitest once in CI and in watch mode locally. Run it right after `ng new`: the generated `app.spec.ts` should pass.

## 17. Practice questions

1. Create the workspace and generate the five layout/page components above.
2. Use `--dry-run` to see what `ng g s features/tasks/data-access/tasks-api` would create.
3. Change the dev server port without editing `angular.json`.

## 18. Interview questions

- **Q: What builder does a modern Angular app use?** `@angular/build:application` (esbuild + Vite dev server).
- **Q: Why use `ng update` instead of npm for upgrades?** It runs migrations that rewrite code for breaking changes.
- **Q: What changed in generated file names in v20?** No `.component`/`.service` suffixes; classes are named `NoteCard`, not `NoteCardComponent`.

## 19. Revision notes

`ng new` · `ng g c|s|guard|interceptor|pipe|directive|environments` · `ng serve` (memory, HMR) · `ng build` (dist) · `ng test` (Vitest) · `ng update` (migrations).

## 20. Completion checklist

- [ ] My Myspce workspace runs on Angular 22 with Tailwind.
- [ ] I generated files with the CLI and understand their names.
- [ ] I know how to fix the three most common install errors.
