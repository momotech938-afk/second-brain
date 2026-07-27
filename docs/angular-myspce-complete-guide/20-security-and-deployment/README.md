# Module 20 — Security, Deployment and Best Practices

Myspce stores your passwords. This module covers the browser-side half of keeping that safe, plus accessibility, i18n basics, production builds, Docker/nginx, CI/CD and troubleshooting.

Back to [README](../README.md) · Previous: [Module 19](../19-testing/README.md) · Next: [Module 21 Labs](../21-myspce-practice-labs/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Security model](01-security.md) | Angular's XSS protection and sanitisation, safe DOM handling, `bypassSecurityTrust*`, CSP and Trusted Types, dependency audits, secrets, clipboard and secret-handling rules for a vault |
| 2 | [Auth and token storage](02-auth-and-tokens.md) | authn vs authz, JWT storage trade-offs, CSRF vs XSS, HTTPS, guards vs backend authorisation, session timeouts, logout everywhere |
| 3 | [Accessibility and i18n](03-accessibility-i18n.md) | semantic HTML, keyboard, focus, ARIA, colour contrast, reduced motion, responsive design, i18n and locale basics |
| 4 | [Deployment and CI/CD](04-deployment-and-ci.md) | production builds, environments, Docker + nginx, S3/CloudFront option, security headers, CI/CD, chunk errors, production troubleshooting, the release checklist |

**Labs:** [Lab 9 — Protected routes and role-aware UI](../21-myspce-practice-labs/lab-09-protected-routes.md) · [Lab 14 — Production build and deployment](../21-myspce-practice-labs/lab-14-production-build.md)

**Time:** ~4 h reading + 2 h practice.

## Self-assessment checklist

- [ ] I can explain how Angular prevents XSS and when my code can bypass that protection.
- [ ] I never use `innerHTML` with untrusted data or `bypassSecurityTrust*` without sanitising first.
- [ ] I can configure a Content Security Policy for Myspce.
- [ ] I can justify memory + httpOnly-cookie token storage and explain CSRF vs XSS trade-offs.
- [ ] I can state why guards are not authorisation.
- [ ] I can make a page keyboard-accessible and screen-reader-friendly.
- [ ] I can build a production image with Docker + nginx including SPA fallback and caching headers.
- [ ] I can set up CI/CD that tests, builds and deploys.
- [ ] I can diagnose blank-page, 404-on-refresh and chunk-load errors in production.
- [ ] I can run through the Myspce security and release checklist.
