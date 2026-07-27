# npm, Git and Browser DevTools

Back to [Module 00](README.md) · Previous: [DOM, CSS and responsive design](05-dom-css-responsive.md) · Next module: [01 Angular fundamentals](../01-angular-fundamentals/README.md)

## 1. Learning objectives

- Read and edit `package.json`; understand semver ranges and lockfiles.
- Install, update and audit dependencies safely.
- Use a simple, professional Git workflow for Myspce.
- Debug with Chrome DevTools and Angular DevTools.

## 2. npm and package.json

```json
{
  "name": "myspce",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test",
    "lint": "ng lint"
  },
  "dependencies": {
    "@angular/core": "^22.1.0",
    "@angular/router": "^22.1.0",
    "rxjs": "~7.8.0",
    "tslib": "^2.8.0"
  },
  "devDependencies": {
    "@angular/cli": "^22.1.0",
    "typescript": "~6.0.0",
    "vitest": "^4.0.0"
  }
}
```

| Concept | Meaning |
|---|---|
| `dependencies` | needed by the app in the browser |
| `devDependencies` | needed only to build/test (CLI, TypeScript, Vitest) |
| `^22.1.0` | any `22.x.y` ≥ 22.1.0 (minor + patch updates) |
| `~6.0.0` | any `6.0.x` (patch only) — TypeScript is pinned like this because Angular supports a narrow range |
| `package-lock.json` | exact resolved versions; **commit it** so every machine installs the same tree |
| `npm ci` | clean install exactly from the lockfile (use in CI) |
| `npx` | run a package's binary without installing globally |

> The exact `vitest` version that `ng new` installs depends on the CLI release; treat the number above as illustrative.

### Commands

```bash
npm install                       # install from package.json (+ update lock if needed)
npm ci                            # exact install from lockfile (CI, Docker)
npm install date-fns              # add a dependency
npm install -D @playwright/test   # add a dev dependency
npm outdated                      # see what's behind
npm audit                         # known vulnerabilities
npm run build                     # run a script
ng update @angular/core @angular/cli   # Angular-aware upgrade with migrations
```

**Never** upgrade Angular packages with plain `npm install @angular/core@latest`: `ng update` runs migrations (for example, v22 adds `ChangeDetectionStrategy.Eager` to old components).

### Node version

Angular 22 requires Node **^22.22.0, ^24.13.1 or ^26.0.0**. Use a version manager (`nvm`, `fnm`, Volta) and pin it:

```text
# .nvmrc
24
```

## 3. Git workflow for Myspce

```bash
git switch -c feat/notes-grid            # one branch per feature
git add -p                               # stage hunks deliberately
git commit -m "feat(notes): add responsive notes grid"
git fetch origin && git rebase origin/main
git push -u origin feat/notes-grid       # open a pull request
```

Commit message convention (Conventional Commits): `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `ci`. It makes the history readable and changelogs automatic.

`.gitignore` essentials: `node_modules/`, `dist/`, `.angular/`, `coverage/`, `.env*`.

Since Myspce is a personal project on a non-company account, keep it in a **private** repo and never commit secrets (API keys, OAuth client secrets, DB passwords). Frontend code is public once deployed anyway.

## 4. Chrome DevTools

| Panel | Use in Angular |
|---|---|
| Elements | inspect rendered DOM, check classes from bindings, toggle `.dark` |
| Console | errors; `ng.getComponent($0)` returns the component instance of the selected element (dev mode) |
| Sources | breakpoints in your `.ts` (source maps), async stack traces |
| Network | see API calls, status codes, headers (`Authorization`, cookies), payloads |
| Application | cookies, localStorage, service workers |
| Performance | record interactions; find long tasks |
| Lighthouse | performance, accessibility, best-practice audits |

## 5. Angular DevTools (browser extension)

- **Components** tab: component tree, inputs, signals and their values.
- **Profiler**: what ran change detection and how long it took.
- **Injector tree**: which injector provides which service.
- **Signal graph**: dependencies between signals, computed and effects.

In v22, the dev-mode app also exposes the signal graph and DI graph to Chrome DevTools as third-party tools.

## 6. Debugging guide

| Symptom | Where to look |
|---|---|
| Blank page | Console for errors; Network for a failed `main-*.js` |
| API 401 | Network → request headers → is `Authorization` present? |
| CORS error | Console says "blocked by CORS policy" → backend config or dev proxy ([Module 13](../13-http-and-api-integration/03-error-handling-retry.md)) |
| Value not updating | Angular DevTools → check the signal value; then check change detection |
| Install fails with peer dependency errors | Your Node/TypeScript versions are outside Angular's supported range |

## 7. Practice

1. Create the Myspce repo with a `.gitignore`, `.nvmrc` and a first commit.
2. Run `npm audit` on a fresh Angular app and read the report.
3. In DevTools, block the `/api/v1/notes` request and observe how your app behaves (Network → right-click → Block request URL).

## 8. Interview questions

- **Q: Why commit `package-lock.json`?** A: Reproducible installs; everyone and CI get identical dependency versions.
- **Q: `npm install` vs `npm ci`?** A: `ci` deletes `node_modules` and installs exactly from the lockfile, failing if it's out of sync; `install` may update the lockfile.
- **Q: Why `ng update` instead of `npm install @latest`?** A: It updates related packages together and runs code migrations.

## 9. Completion checklist

- [ ] I can explain every field of my `package.json`.
- [ ] My Node version matches Angular 22's range.
- [ ] I've used Angular DevTools to inspect a signal value.
