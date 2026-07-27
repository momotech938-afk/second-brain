# Configuration, Environments and Builds

Back to [Module 01](README.md) · Previous: [Project structure](03-project-structure-and-bootstrap.md) · Next: [Angular vs React](05-angular-vs-react.md)

## 1. Learning objectives

- Read and change `angular.json` targets and configurations.
- Understand the TypeScript and Angular compiler options in `tsconfig.json`.
- Configure environment-specific values (API URL) without leaking secrets.
- Proxy `/api` to a local backend during development.
- Read a production build's output and set budgets.

## 2. Prerequisites

[Project structure and bootstrap](03-project-structure-and-bootstrap.md).

## 3. `angular.json` — the workspace file

```json
{
  "projects": {
    "myspce": {
      "projectType": "application",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular/build:application",
          "options": {
            "browser": "src/main.ts",
            "tsConfig": "tsconfig.app.json",
            "assets": [{ "glob": "**/*", "input": "public" }],
            "styles": ["src/styles.css"]
          },
          "configurations": {
            "production": {
              "budgets": [
                { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" },
                { "type": "anyComponentStyle", "maximumWarning": "4kB", "maximumError": "8kB" }
              ],
              "outputHashing": "all"
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true,
              "fileReplacements": [
                { "replace": "src/environments/environment.ts", "with": "src/environments/environment.development.ts" }
              ]
            }
          },
          "defaultConfiguration": "production"
        },
        "serve": {
          "builder": "@angular/build:dev-server",
          "configurations": {
            "production": { "buildTarget": "myspce:build:production" },
            "development": { "buildTarget": "myspce:build:development" }
          },
          "defaultConfiguration": "development"
        },
        "test": { "builder": "@angular/build:unit-test" }
      }
    }
  }
}
```

| Key | Meaning |
|---|---|
| `architect.build/serve/test` | the targets `ng build`/`ng serve`/`ng test` run |
| `configurations` | named variants (`ng build -c development`) |
| `defaultConfiguration` | `ng build` = production, `ng serve` = development |
| `budgets` | fail/warn the build when bundles grow too large |
| `fileReplacements` | swap files per configuration (environments) |
| `assets` | static files copied into the output |
| `prefix` | component selector prefix (`app-note-card`) |

## 4. Environments

```bash
ng generate environments
```

```typescript
// src/environments/environment.ts  (production — the default)
export const environment = {
  production: true,
  apiUrl: '/api/v1',               // same origin behind nginx/CloudFront
  googleClientId: 'xxxx.apps.googleusercontent.com',
};

// src/environments/environment.development.ts
export const environment = {
  production: false,
  apiUrl: '/api/v1',               // proxied to the NestJS dev server
  googleClientId: 'xxxx-dev.apps.googleusercontent.com',
};
```

> **Security rule:** everything in `environment.ts` ends up in public JavaScript. A Google OAuth **client ID** is public by design; a client **secret**, database password, JWT secret or encryption key must **never** be here. Those live only on the NestJS server.

Use it through an injection token, not direct imports everywhere, so tests can override it:

```typescript
// core/http/api-url.token.ts
import { InjectionToken } from '@angular/core';
export const API_URL = new InjectionToken<string>('API_URL');
// app.config.ts:  { provide: API_URL, useValue: environment.apiUrl }
```

### Runtime config (optional)

Build-time environments mean one build per environment. If you want **one build** deployed to staging and production, load `/config.json` at startup with `provideAppInitializer` and store it in a service. For a single-user app with one production target, build-time environments are simpler.

## 5. Dev proxy to the NestJS backend

Calling `http://localhost:3000` from `http://localhost:4200` triggers CORS. A proxy makes the browser think the API is same-origin:

```json
// proxy.conf.json
{
  "/api": {
    "target": "http://localhost:3000",
    "secure": false,
    "changeOrigin": true,
    "logLevel": "info"
  }
}
```

```json
// angular.json → serve → options
"options": { "proxyConfig": "proxy.conf.json" }
```

Now `http.get('/api/v1/notes')` from Angular hits `http://localhost:3000/api/v1/notes`. Cookies (refresh token) work naturally because everything is on `localhost:4200`.

## 6. `tsconfig.json` and Angular compiler options

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "experimentalDecorators": true,
    "importHelpers": true,
    "target": "ES2022",
    "module": "preserve"
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "typeCheckHostBindings": true,
    "strictTemplates": true
  }
}
```

The exact list `ng new` writes may differ slightly between minor versions; what matters:

- `strict` — full TypeScript strictness.
- `strictTemplates` — templates type-checked like TypeScript. **Default in v22** even if omitted; the v22 migration adds `"strictTemplates": false` to old projects that didn't have it, so check upgraded projects.
- `extendedDiagnostics` (optional) — extra template warnings such as "nullish coalescing on a non-nullable value".

Path aliases make imports readable:

```json
"compilerOptions": {
  "paths": {
    "@core/*": ["./src/app/core/*"],
    "@shared/*": ["./src/app/shared/*"],
    "@features/*": ["./src/app/features/*"]
  }
}
```

## 7. Builds and output

```bash
ng build                    # production
ng build -c development     # unoptimised, with source maps
ng build --stats-json       # writes stats.json for bundle analysis (Module 17)
```

```text
dist/myspce/
├── browser/
│   ├── index.html
│   ├── main-7XQH3KDV.js        # app code (hashed name for caching)
│   ├── chunk-ABCD1234.js       # lazy route chunks (notes, credentials…)
│   ├── polyfills-….js          # only if polyfills are configured
│   ├── styles-….css
│   └── favicon.ico
└── 3rdpartylicenses.txt
```

The build log lists **initial** chunks (loaded at startup) and **lazy** chunks (loaded on navigation). Keep initial small. In v22, lazy-chunk optimisation (merging tiny chunks) is on by default in production.

Production build does: AOT compile, tree-shaking, minification, CSS optimisation, hashed filenames, license extraction, budgets.

## 8. Step-by-step: adding a budget for Myspce

1. Run `ng build` and note the initial total.
2. Set `maximumWarning` ~20% above it and `maximumError` ~50% above it.
3. When a budget fails, find the culprit with `--stats-json` + esbuild analyzer before raising the limit.

## 9. Myspce example — scripts

```json
"scripts": {
  "start": "ng serve",
  "start:api": "ng serve --proxy-config proxy.conf.json",
  "build": "ng build",
  "build:analyze": "ng build --stats-json",
  "test": "ng test",
  "test:ci": "ng test --no-watch --coverage",
  "lint": "ng lint"
}
```

`ng lint` needs `ng add angular-eslint` first.

## 10. Advanced example — multiple configurations

```json
"configurations": {
  "production": { "…": "…" },
  "staging": {
    "fileReplacements": [
      { "replace": "src/environments/environment.ts", "with": "src/environments/environment.staging.ts" }
    ]
  }
}
```

`ng build -c staging`. Each configuration can override any build option.

## 11. Visual explanation

```text
              ┌─ development: no optimisation, source maps, env.development.ts
ng build -c ──┼─ staging:     optimised, env.staging.ts
              └─ production:  optimised, hashed, budgets, env.ts
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Putting secrets in `environment.ts` | Keep secrets on the server only |
| Hard-coding `http://localhost:3000` in services | Use `API_URL` + proxy |
| Raising budgets every time they fail | Investigate the bundle first |
| Deploying `dist/myspce` instead of `dist/myspce/browser` | Serve the `browser` folder |

## 13. Debugging guide

| Symptom | Fix |
|---|---|
| Deep link `/notes/123` gives 404 on the server | Configure SPA fallback to `index.html` ([Module 20](../20-security-and-deployment/04-deployment-and-ci.md)) |
| Proxy not working | Path must start with `/api`; restart `ng serve` after editing the proxy file |
| `Budget exceeded` | `ng build --stats-json`, then inspect |
| Env value wrong in production | Check `fileReplacements` and which configuration you built |

## 14. Best practices

- One `API_URL` token; one proxy for dev.
- Budgets on from day one.
- Treat everything in the browser bundle as public.

## 15. Angular vs React

`angular.json` ≈ `vite.config.ts` + `package.json` scripts. Angular environments ≈ Vite's `import.meta.env` files, with the same rule: no secrets.

## 16. Testing

Provide a test `API_URL` in specs:

```typescript
TestBed.configureTestingModule({ providers: [{ provide: API_URL, useValue: '/api/v1' }] });
```

## 17. Practice questions

1. Add a `staging` configuration with its own `apiUrl`.
2. Set up `proxy.conf.json` and confirm in the Network tab that `/api/v1/health` reaches port 3000.
3. Make the build fail by lowering the initial budget, then restore it.

## 18. Interview questions

- **Q: How do Angular environments work?** `fileReplacements` swaps the environment file at build time per configuration.
- **Q: Why use a dev proxy?** To avoid CORS and keep cookies same-origin during development.
- **Q: What's the difference between initial and lazy chunks?** Initial chunks load at startup; lazy chunks load when their route/`@defer` block is needed.

## 19. Revision notes

`angular.json` targets + configurations · environments via `fileReplacements` · `API_URL` token · `proxy.conf.json` · budgets · `dist/<app>/browser`.

## 20. Completion checklist

- [ ] Myspce has dev/prod environments and an `API_URL` token.
- [ ] `/api` is proxied in development.
- [ ] I can read the build log and explain initial vs lazy chunks.
