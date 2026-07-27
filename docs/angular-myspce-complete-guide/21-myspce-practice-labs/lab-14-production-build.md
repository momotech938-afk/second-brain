# Lab 14 — Production Build, Docker, nginx and CI/CD

Back to [Labs](README.md) · Solution: [lab-14-solution.md](solutions/lab-14-solution.md) · Previous: [Lab 13](lab-13-performance.md) · Next: [Module 22 — Final Project Readiness](../22-final-project-readiness/README.md)

## 1. Learning objective

Ship the frontend the way it will run for real: production and demo build configurations, environment handling without secrets, a Docker image serving the SPA through nginx with correct caching, SPA fallback and security headers, graceful handling of stale chunks after deploys, and a CI pipeline that tests, builds and (optionally) deploys.

## 2. Prerequisites

[Module 20 notes 1 and 4](../20-security-and-deployment/README.md), [Module 01 note 4](../01-angular-fundamentals/04-configuration-and-builds.md), [Lab 12](lab-12-tests.md), [Lab 13](lab-13-performance.md).

## 3. Problem statement

Confirmed: hosting is "free AWS later". The NestJS API doesn't exist yet, so you need two things: a **production** build that expects the real API at `/api/v1`, and a **demo** build that bundles the mock backend so the deployed app can be clicked through today. Neither may contain secrets.

## 4. Expected behaviour

- `ng build` → production: mock excluded, `apiUrl: '/api/v1'`, hashed file names, budgets enforced.
- `ng build -c demo` → same optimisations, mock interceptor included, a visible "Demo data" badge.
- `docker build` + `docker run -p 8080:80` → app at `http://localhost:8080`. Deep links (`/notes/123`) work on refresh. Hashed assets cached 1 year; `index.html` `no-cache`.
- Response headers include `X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options` and a report-only CSP, on **every** response (including assets).
- After a new deploy, an old open tab that fails to load a chunk shows "A new version is available — Reload" instead of breaking.
- GitHub Actions: lint → test → build → E2E → image build on every PR; deploy only from `main` (optional, OIDC — no AWS keys in secrets).
- `grep -ri "secret\|password\|apikey" dist/` finds nothing sensitive (only UI labels).

## 5. Relevant concepts

`angular.json` configurations and `fileReplacements`, environment files, budgets, output hashing, multi-stage Dockerfile, nginx `try_files`, `add_header` inheritance, `Cache-Control`, `ErrorHandler`, CSP, GitHub Actions, OIDC.

## 6. Step-by-step implementation

1. Create `environment.demo.ts` (`production: true`, `useMockApi: true`) and a `demo` configuration in `angular.json` that extends production settings with a `fileReplacements` entry.
2. Make sure the mock is only *referenced* via `environment.useMockApi`. In production builds, `useMockApi` is a constant `false`, so the mock interceptor array is empty. Verify with the bundle analyser that mock code is gone from `ng build`.
3. Add a `DemoBadge` component rendered when `environment.useMockApi` is true.
4. Add `AppErrorHandler`: detect chunk-load failures and show a "new version" toast with a Reload action. Log everything else to the console with a short id (no PII).
5. Write the `Dockerfile` (multi-stage, `npm ci`, build arg for the configuration) and `nginx.conf`. Use a `security-headers.conf` snippet included in every `location` that sets its own headers.
6. Add `.dockerignore` (`node_modules`, `dist`, `.angular`, `.git`, `e2e/test-results`).
7. Add `docker-compose.yml` with the web container (and a commented-out `api` service for when NestJS exists).
8. Verify: deep link refresh, `curl -I` for headers and cache, Lighthouse once more.
9. CI: a `ci.yml` (PRs) and a `deploy.yml` (main) based on Module 20 note 4. Deployment is optional until you choose the AWS option.
10. Write `docs/RELEASE_CHECKLIST.md` from Module 20 note 4 §9 and tick it.

## 7. Complete working code

Environment files, the `angular.json` excerpt, `AppErrorHandler`, `DemoBadge`, Dockerfile, nginx config with the header snippet, compose file, and workflows are in the [solution](solutions/lab-14-solution.md).

## 8. Important lines explained

- `"fileReplacements": [{ "replace": "src/environments/environment.ts", "with": "src/environments/environment.demo.ts" }]` — build-time configuration; values are baked into the bundle, so **only public values** belong there.
- `try_files $uri $uri/ /index.html;` — SPA fallback for client routes; real missing assets still 404 via the asset `location`.
- `include /etc/nginx/snippets/security-headers.conf;` in each location — nginx drops server-level `add_header`s in a location that defines its own.
- `Cache-Control: public, max-age=31536000, immutable` for hashed files, `no-cache` for `index.html` — instant updates without cache-busting hacks.
- `/Loading chunk|dynamically imported module|ChunkLoadError/` — the error shapes a stale chunk produces across browsers/bundlers.
- `permissions: id-token: write` + `configure-aws-credentials` with a role — short-lived credentials instead of stored keys.

## 9. Expected output

```text
$ curl -sI http://localhost:8080/notes/123 | grep -iE 'HTTP/|cache-control|x-content-type|content-security'
HTTP/1.1 200 OK
Cache-Control: no-cache
X-Content-Type-Options: nosniff
Content-Security-Policy-Report-Only: default-src 'self'; …

$ curl -sI http://localhost:8080/main-ABCD1234.js | grep -iE 'cache-control|x-content-type'
Cache-Control: public, max-age=31536000, immutable
X-Content-Type-Options: nosniff
```

(File hashes differ per build.)

## 10. Common mistakes and debugging

| Problem | Fix |
|---|---|
| Blank page, "MIME type text/html" for JS | wrong folder deployed (`dist/myspce` instead of `dist/myspce/browser`), or `<base href>` mismatch |
| 404 on refresh | missing SPA fallback |
| Security headers missing on JS/CSS | location-level `add_header` overrode them; use the include snippet |
| Mock shipped in production | mock imported directly somewhere (e.g. `mockSession` in a page) — route all access through `environment.useMockApi` checks or move it into the interceptor |
| API keys in `environment.ts` | never; anything in the bundle is public. Secrets live on the server |
| CI deploys on PRs | restrict the deploy job to `push` on `main` |
| New deploy, users see old version | `index.html` cached by CDN; invalidate it and send `no-cache` |

## 11. Test cases

```bash
# smoke test script (scripts/smoke.sh) — run against the container
set -euo pipefail
BASE=${1:-http://localhost:8080}
code() { curl -s -o /dev/null -w '%{http_code}' "$1"; }
[ "$(code "$BASE/")" = 200 ]
[ "$(code "$BASE/notes/01JTEST")" = 200 ]                                  # SPA fallback
[ "$(code "$BASE/does-not-exist.js")" = 404 ]                              # real missing asset
curl -sI "$BASE/" | grep -qi 'x-content-type-options: nosniff'
JS=$(curl -s "$BASE/" | grep -oE 'main-[A-Za-z0-9]+\.js' | head -1)
curl -sI "$BASE/$JS" | grep -qi 'immutable'
curl -sI "$BASE/$JS" | grep -qi 'x-content-type-options: nosniff'        # headers on assets too
echo "smoke OK"
```

```typescript
it('AppErrorHandler offers reload on chunk errors', () => {
  const toast = fakeToast();
  TestBed.configureTestingModule({ providers: [{ provide: ToastService, useValue: toast }] });
  const handler = TestBed.runInInjectionContext(() => new AppErrorHandler());
  handler.handleError(new TypeError('Failed to fetch dynamically imported module: /chunk-ABC.js'));
  expect(toast.info).toHaveBeenCalledWith(expect.stringMatching(/new version/i), expect.anything());
});
```

## 12. Independent practice task

Pick **one** AWS option from [Module 20 note 4 §4](../20-security-and-deployment/04-deployment-and-ci.md#4-aws-free-tier-options-confirmed-plan-free-aws-later) and write `docs/DEPLOYMENT.md` for it: architecture diagram (text is fine), one-time setup steps, the deploy command or workflow, rollback procedure, and how HTTPS is provided. Deploy the **demo** build if you're ready. Otherwise mark each step "not done yet". Don't invent a domain or account details; use placeholders like `<your-domain>`.

## 13. Solution

See [solutions/lab-14-solution.md](solutions/lab-14-solution.md).

## 14. Completion checklist

- [ ] Production and demo configurations; mock absent from production bundle.
- [ ] Docker image runs; smoke script passes.
- [ ] Headers on every response; caching correct.
- [ ] Stale-chunk handling.
- [ ] CI on PRs; deploy workflow (optional) uses OIDC.
- [ ] `DEPLOYMENT.md` + release checklist.
- [ ] Tagged `lab-14` — **all labs complete**. Go to [Module 22](../22-final-project-readiness/README.md).
