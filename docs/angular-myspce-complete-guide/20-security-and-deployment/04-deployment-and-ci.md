# Deployment, CI/CD and Production Troubleshooting

Back to [Module 20](README.md) · Previous: [Accessibility and i18n](03-accessibility-i18n.md) · Next: [Module 21 Labs](../21-myspce-practice-labs/README.md)

## 1. Learning objectives

- Produce a correct production build and configuration.
- Serve the SPA with nginx in Docker (SPA fallback, caching, compression, security headers).
- Know the S3 + CloudFront alternative for the free-tier AWS plan.
- Build a CI/CD pipeline.
- Diagnose common production failures.
- Run the Myspce release checklist.

## 2. Production build

```bash
ng build                              # production configuration by default
ls dist/myspce/browser                # deploy THIS folder
```

Checklist for `angular.json` production config: `optimization: true`, `outputHashing: "all"`, `sourceMap: false` (or hidden + uploaded privately), budgets set, `environment.ts` with production values.

## 3. Docker + nginx

```dockerfile
# Dockerfile — multi-stage: build with Node, serve with nginx
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx ng build

FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist/myspce/browser /usr/share/nginx/html
EXPOSE 80
```

```nginx
# nginx.conf
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  gzip on;
  gzip_types text/plain text/css application/javascript application/json image/svg+xml;

  # Security headers
  add_header X-Content-Type-Options "nosniff" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header X-Frame-Options "DENY" always;
  add_header Permissions-Policy "camera=(), microphone=(), geolocation=(self)" always;
  add_header Content-Security-Policy-Report-Only "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob: https://*.amazonaws.com; connect-src 'self' https://*.amazonaws.com; frame-ancestors 'none'; base-uri 'self'; object-src 'none'" always;
  # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;   # only when served over HTTPS

  # API → NestJS (same origin, so no CORS and cookies just work)
  location /api/ {
    proxy_pass http://api:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  # Hashed assets: cache for a year
  location ~* \.(?:js|css|woff2?|svg|png|jpg|webp|avif|ico)$ {
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri =404;
  }

  # index.html: always revalidate so new deploys are picked up
  location = /index.html {
    add_header Cache-Control "no-cache";
  }

  # SPA fallback: deep links like /notes/01J… serve index.html
  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

Important: `add_header` directives inside a `location` replace those from the `server` block — repeat security headers in locations that add their own headers (or use an `include` snippet).

```bash
docker build -t myspce-web .
docker run --rm -p 8080:80 myspce-web
```

## 4. AWS free-tier options (confirmed plan: "free AWS later")

| Option | Frontend | API | Notes |
|---|---|---|---|
| A. Single EC2 (t3.micro free tier) | nginx container serving the SPA | NestJS container behind the same nginx | one box, simplest; TLS via Let's Encrypt |
| B. S3 + CloudFront | static files on S3, CloudFront CDN | NestJS on EC2/ECS behind CloudFront `/api/*` behaviour | global CDN; configure CloudFront error responses (403/404 → `/index.html`, 200) for SPA fallback |
| C. ECS Fargate | nginx image | NestJS image | closest to your work setup; not free-tier friendly |

Whichever you choose, keep **frontend and API on the same domain** (`/api` path) for cookies and CORS simplicity.

## 5. Environments

| Env | Build | API | Notes |
|---|---|---|---|
| local | `ng serve` | proxy to `localhost:3000` (or mock interceptor) | |
| staging (optional) | `ng build -c staging` | staging API | test migrations and deploys |
| production | `ng build` | `/api/v1` same origin | |

Never point a development build at production data.

## 6. CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push:
    branches: [main]
jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write }   # OIDC to AWS, no long-lived keys
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - run: npm audit --omit=dev --audit-level=high
      - run: npx ng lint
      - run: npx ng test --no-watch --coverage
      - run: npx ng build
      - run: npx playwright install --with-deps chromium && npx playwright test --project=desktop-chrome
      # Option B (S3 + CloudFront):
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::<account-id>:role/myspce-deploy, aws-region: ap-south-1 }
      - run: |
          aws s3 sync dist/myspce/browser s3://<bucket> --delete --exclude index.html --cache-control "public,max-age=31536000,immutable"
          aws s3 cp dist/myspce/browser/index.html s3://<bucket>/index.html --cache-control "no-cache"
          aws cloudfront create-invalidation --distribution-id <id> --paths /index.html
```

Replace `<account-id>`, `<bucket>` and `<id>` with your values. The order (assets first, `index.html` last) avoids users loading a new `index.html` that points to files not uploaded yet. Keep old hashed assets for a while so open tabs can still lazy-load their chunks.

## 7. Production troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Blank page, console `Failed to load module script … MIME type text/html` | server returns `index.html` for JS files (wrong path/base href) | check `<base href>`, deploy `browser/` folder, nginx `try_files` order |
| 404 on refresh of `/notes/123` | no SPA fallback | nginx `try_files … /index.html` / CloudFront error responses |
| `ChunkLoadError` / "Failed to fetch dynamically imported module" after deploy | old tab requesting deleted chunks | keep old assets; reload on that error (`AppErrorHandler`, [Module 13](../13-http-and-api-integration/03-error-handling-retry.md)) |
| Everything works locally, API fails in prod with CORS | API on another origin | same-origin `/api` proxy |
| Logged out on every reload in prod | refresh cookie not set/sent (`Secure` on HTTP, wrong `Path`/`Domain`, `SameSite`) | serve over HTTPS; check cookie attributes in DevTools |
| Styles missing | CSP blocks inline styles | add nonce or allow styles; check console |
| Old version keeps showing | `index.html` cached | `Cache-Control: no-cache` on `index.html`; invalidate CDN |

## 8. Code review and maintainability

Before merging any PR in Myspce, check:

- [ ] Feature code in its feature folder; no cross-feature imports.
- [ ] Components presentational unless they're pages; pages talk to stores.
- [ ] State in signals; derived state in `computed`; no state-syncing effects.
- [ ] Every `@for` has a good `track`; no logic in templates.
- [ ] Every subscription has a teardown.
- [ ] Inputs typed; no `any`.
- [ ] Tests added/updated; CI green.
- [ ] No secrets, no `console.log` of sensitive data, no `bypassSecurityTrust*` on user input.
- [ ] Accessible names on icon buttons; forms labelled.

## 9. Myspce security and release checklist

**Security**
- [ ] HTTPS only, HSTS enabled.
- [ ] CSP enforced (after report-only phase), `frame-ancestors 'none'`.
- [ ] No tokens or secrets in localStorage/sessionStorage/IndexedDB/URLs/logs.
- [ ] Access token in memory, refresh cookie httpOnly + Secure + SameSite=Strict + path-scoped.
- [ ] Auth header sent only to `/api`.
- [ ] Credentials list API returns no secrets; reveal requires re-auth; clipboard auto-clears.
- [ ] Idle lock works; logout clears memory and server session.
- [ ] Backend enforces roles and ownership for every endpoint (verify with direct API calls).
- [ ] `npm audit` clean for high/critical; Dependabot on.
- [ ] Source maps not public.

**Release**
- [ ] `ng build` with no warnings you haven't reviewed; budgets pass.
- [ ] Unit, integration and E2E tests green.
- [ ] Lighthouse mobile: LCP ≤ 2.5 s, CLS ≤ 0.1 on Dashboard and Notes.
- [ ] axe: no serious/critical violations.
- [ ] Deep links and refresh work on every route.
- [ ] Assets cached long, `index.html` no-cache.
- [ ] Rollback plan: previous image tag / previous S3 version available.

## 10. Interview questions

- **Q: Why does refreshing a deep link 404 on a static host?** The server looks for a file at that path; SPAs need a fallback that serves `index.html` for unknown paths.
- **Q: How do you avoid stale apps after deploys?** Hashed asset names with long caching, `index.html` with `no-cache`, and handling chunk-load errors.

## 11. Completion checklist

- [ ] Docker image serves the app with SPA fallback, caching and headers.
- [ ] CI runs tests and builds on every push; deploy on `main`.
- [ ] Release checklist completed once end-to-end.
