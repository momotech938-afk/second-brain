# Lab 14 — Solution

Back to [Lab 14](../lab-14-production-build.md) · [Labs index](../README.md)

## Environment files

These replace the single development-style `environment.ts` used since Lab 6. Production becomes the default file, and development gets its own file via `fileReplacements`. (`ng generate environments` creates this layout.)

```typescript
// src/environments/environment.ts — production (default)
export const environment = {
  production: true,
  apiUrl: '/api/v1',
  useMockApi: false,
  googleLoginUrl: '/api/v1/auth/google',
};
```

```typescript
// src/environments/environment.development.ts — ng serve
export const environment = {
  production: false,
  apiUrl: '/api/v1',
  useMockApi: true,
  googleLoginUrl: '/auth/callback?mock=1',
};
```

```typescript
// src/environments/environment.demo.ts — deployed demo with fake data
export const environment = {
  production: true,
  apiUrl: '/api/v1',
  useMockApi: true,
  googleLoginUrl: '/auth/callback?mock=1',
};
```

No keys, tokens or passwords appear in any of them. Google OAuth client secrets live only on the NestJS server.

## `angular.json` (excerpt)

```json
"configurations": {
  "production": {
    "budgets": [
      { "type": "initial", "maximumWarning": "450kB", "maximumError": "600kB" },
      { "type": "anyComponentStyle", "maximumWarning": "4kB", "maximumError": "8kB" }
    ],
    "outputHashing": "all"
  },
  "demo": {
    "budgets": [
      { "type": "initial", "maximumWarning": "500kB", "maximumError": "650kB" }
    ],
    "outputHashing": "all",
    "fileReplacements": [
      { "replace": "src/environments/environment.ts", "with": "src/environments/environment.demo.ts" }
    ]
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
```

Optimisation defaults to on for configurations that don't disable it. Check `ng build -c demo` output to confirm the files are minified and hashed. Budget numbers are placeholders; use your Lab 13 measurements.

## Keeping the mock out of production

```typescript
// app.config.ts
import { environment } from '../environments/environment';
import { mockBackendInterceptor } from './core/http/mock-backend.interceptor';

const devInterceptors = environment.useMockApi ? [mockBackendInterceptor] : [];

provideHttpClient(withInterceptors([errorInterceptor, authInterceptor, reauthInterceptor, ...devInterceptors])),
```

Because `environment.useMockApi` is the literal `false` in the production file, the optimiser drops the dead branch and the mock module. **Verify** rather than trust: run `ng build --stats-json`, open the analyser, and search for `mock-backend` (it should be absent), or `grep -l "No mock for" dist/myspce/browser/*.js` (no output).

If the OAuth callback page (Lab 8) imports `mockSession` directly, that import keeps the mock alive. Move the "set fake cookie" step into the interceptor (handle `GET /api/v1/auth/mock-google` there) and have the dev `googleLoginUrl` point to that URL.

## `DemoBadge`

```typescript
// core/layout/demo-badge.ts
import { Component } from '@angular/core';
import { environment } from '../../../environments/environment';

@Component({
  selector: 'app-demo-badge',
  template: `
    @if (demo) {
      <p class="fixed bottom-3 left-3 z-50 rounded-full bg-amber-400 px-3 py-1 text-xs font-semibold text-black shadow">
        Demo data — nothing is saved
      </p>
    }
  `,
})
export class DemoBadge {
  protected readonly demo = environment.useMockApi && environment.production;
}
```

## `AppErrorHandler`

```typescript
// core/errors/app-error-handler.ts
import { ErrorHandler, Injectable, inject } from '@angular/core';
import { DOCUMENT } from '@angular/common';
import { ToastService } from '../../shared/ui/toast/toast-service';

const CHUNK_ERROR = /Loading chunk|dynamically imported module|ChunkLoadError|Importing a module script failed/i;

@Injectable()
export class AppErrorHandler implements ErrorHandler {
  private toast = inject(ToastService);
  private win = inject(DOCUMENT).defaultView;
  private offeredReload = false;

  handleError(error: unknown): void {
    const message = error instanceof Error ? error.message : String(error);

    if (CHUNK_ERROR.test(message)) {
      if (!this.offeredReload) {
        this.offeredReload = true;
        this.toast.info('A new version of MySpace is available.', {
          action: { label: 'Reload', run: () => this.win?.location.reload() },
          ms: 60_000,
        });
      }
      return;
    }

    const id = Math.random().toString(36).slice(2, 8);
    console.error(`[app-error ${id}]`, error);   // no user data in the message; later: send to a logging endpoint
  }
}
```

```typescript
// app.config.ts
{ provide: ErrorHandler, useClass: AppErrorHandler },
```

`@Injectable()` (not `@Service()`) because it's provided explicitly under the `ErrorHandler` token, not root-provided under its own class.

## Docker

```dockerfile
# Dockerfile
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG CONFIG=production
RUN npx ng build --configuration ${CONFIG}

FROM nginx:1.27-alpine
COPY nginx/default.conf /etc/nginx/conf.d/default.conf
COPY nginx/security-headers.conf /etc/nginx/snippets/security-headers.conf
COPY --from=build /app/dist/myspce/browser /usr/share/nginx/html
EXPOSE 80
HEALTHCHECK CMD wget -qO- http://localhost/ >/dev/null || exit 1
```

Node 24 satisfies Angular 22's Node requirement (`^22.22.0 || ^24.13.1 || ^26.0.0`); pin the exact minor in CI if builds must be reproducible. The nginx tag is an example; use a current stable tag.

```text
# .dockerignore
node_modules
dist
.angular
.git
coverage
e2e/test-results
playwright-report
```

## nginx

```nginx
# nginx/security-headers.conf
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-Frame-Options "DENY" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Content-Security-Policy-Report-Only "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; connect-src 'self'; font-src 'self'; frame-ancestors 'none'; base-uri 'self'; object-src 'none'" always;
# add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;   # enable only behind HTTPS
```

```nginx
# nginx/default.conf
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  gzip on;
  gzip_types text/plain text/css application/javascript application/json image/svg+xml;

  include /etc/nginx/snippets/security-headers.conf;

  # Later, when NestJS exists (same origin → cookies and no CORS):
  # location /api/ { proxy_pass http://api:3000; proxy_set_header Host $host; proxy_set_header X-Forwarded-Proto $scheme; }

  location ~* \.(?:js|mjs|css|woff2?|svg|png|jpg|jpeg|webp|avif|ico)$ {
    include /etc/nginx/snippets/security-headers.conf;
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri =404;
  }

  location = /index.html {
    include /etc/nginx/snippets/security-headers.conf;
    add_header Cache-Control "no-cache";
  }

  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

`location /` defines no `add_header` of its own, so it inherits the server-level include. The SPA fallback internally redirects to `/index.html`, which picks up the `no-cache` location.

The CSP starts as **Report-Only**. Watch the console for violations, then switch to an enforcing `Content-Security-Policy` header. Tailwind v4 emits a normal stylesheet, but Angular may inline critical CSS. Keep `'unsafe-inline'` for styles, or configure a nonce ([Module 20 note 1](../../20-security-and-deployment/01-security.md)), before enforcing.

## Compose

```yaml
# docker-compose.yml
services:
  web:
    build:
      context: .
      args: { CONFIG: demo }        # switch to production once the API exists
    ports: ["8080:80"]
    restart: unless-stopped
  # api:                            # NestJS — not built yet
  #   image: myspce-api:latest
  #   environment: [ "DATABASE_URL=..." ]   # secrets via env/secret manager, never in the frontend
```

## CI for pull requests

```yaml
# .github/workflows/ci.yml
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  web:
    runs-on: ubuntu-latest
    permissions: { contents: read }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - run: npx ng lint
      - run: npx ng test --no-watch --coverage
      - run: npx ng build
      - run: npx ng build --configuration e2e
      - run: npx playwright install --with-deps chromium
      - run: npx serve -s dist/myspce/browser -l 4200 & npx wait-on http://localhost:4200 && npx playwright test
      - if: failure()
        uses: actions/upload-artifact@v4
        with: { name: playwright-report, path: playwright-report }
      - run: docker build --build-arg CONFIG=demo -t myspce-web:${{ github.sha }} .
      - run: docker run -d -p 8080:80 --name web myspce-web:${{ github.sha }} && sleep 2 && bash scripts/smoke.sh http://localhost:8080
```

The second `ng build` overwrites `dist/`; that's fine because E2E needs the `e2e` build (Lab 12). The Docker build rebuilds inside the image anyway.

Deploy: use `deploy.yml` from [Module 20 note 4 §6](../../20-security-and-deployment/04-deployment-and-ci.md#6-cicd-github-actions), triggered only on `push` to `main`, with OIDC (`id-token: write`) and a role ARN placeholder.

## Independent task — `docs/DEPLOYMENT.md` skeleton (Option A: single EC2)

```markdown
# Deploying MySpace (demo frontend) — Option A: single EC2

Status: <not done yet | done on YYYY-MM-DD>

## Architecture
Browser ──HTTPS──▶ EC2 (<instance-type>, <region>)
                    └─ nginx container (myspce-web) ── /api/* ─▶ (future) NestJS container

## One-time setup
1. Launch EC2 (free tier eligible type), open ports 80/443 in the security group.
2. Install Docker + Docker Compose plugin.
3. Point <your-domain> A record to the instance's Elastic IP.
4. TLS: certbot (Let's Encrypt) on the host, or a reverse proxy container (e.g. Caddy) in front of nginx.

## Deploy
ssh <user>@<host> 'cd myspce && git pull && docker compose up -d --build'

## Rollback
git checkout <previous-tag> && docker compose up -d --build

## Checks after deploy
- scripts/smoke.sh https://<your-domain>
- Lighthouse mobile ≥ 90 on /login
- Response headers present (HSTS enabled now that HTTPS works)
```

Options B (S3 + CloudFront) and C (ECS) follow the same outline; the Module 20 table lists their trade-offs. Real AWS limits and pricing change, so check the current free-tier terms before choosing.
