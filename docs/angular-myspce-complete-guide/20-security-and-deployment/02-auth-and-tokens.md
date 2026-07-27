# Authentication, Authorisation and Token Storage

Back to [Module 20](README.md) · Previous: [Security model](01-security.md) · Next: [Accessibility and i18n](03-accessibility-i18n.md)

## 1. Learning objectives

- Separate authentication (who you are) from authorisation (what you may do).
- Compare token storage options and their XSS/CSRF trade-offs.
- Understand why guards don't secure anything.
- Design sessions: timeouts, re-auth, logout everywhere.

## 2. Authn vs authz

| | Authentication | Authorisation |
|---|---|---|
| Question | Who is this? | May they do this? |
| Myspce | email/password or Google login → tokens | role (`super_admin`/`admin`/`user`), ownership of every row (`user_id`) |
| Where enforced | NestJS auth endpoints | **NestJS on every request** (guards, ownership checks) |
| Frontend role | collect credentials, hold tokens, restore sessions | **hide or disable UI** the user can't use |

## 3. Token storage trade-offs

| Option | XSS can steal it? | CSRF risk | Notes |
|---|---|---|---|
| `localStorage` / `sessionStorage` | **yes** | no | simplest; any XSS = full account takeover |
| In-memory (signal) | harder (must run in page during session) | no | lost on reload → need a refresh mechanism |
| httpOnly cookie | **no** (JS can't read it) | **yes**, unless mitigated | browser sends it automatically |
| **In-memory access token + httpOnly refresh cookie** (Myspce) | access token only while the page is open; refresh token not readable | low with `SameSite=Strict` + path scoping | the recommended balance |

### Why the Myspce design works

- **Access token** (short-lived, ~15 min) lives in `AuthStore` memory, sent as `Authorization: Bearer`. Cookies aren't used for API auth, so classic CSRF against `/notes` etc. doesn't apply.
- **Refresh token** in an httpOnly, `Secure`, `SameSite=Strict` cookie scoped to `/api/v1/auth`. Cross-site requests won't carry it (`Strict`), JS can't read it, and only the auth endpoints receive it.
- **Rotation** on the server: each refresh issues a new refresh token and invalidates the old one; reuse of an old token signals theft → revoke all sessions.
- **XSS still matters**: an attacker running script in your page can call your API while the page is open. That's why [CSP and sanitisation](01-security.md) are non-negotiable.

If the backend later uses cookie-based auth for all endpoints, add CSRF protection: Angular's `withXsrfConfiguration()` reads an `XSRF-TOKEN` cookie and sends it as `X-XSRF-TOKEN` on mutating requests; the server must validate it.

## 4. Guards vs backend authorisation

```text
Frontend guard says "no"         → user doesn't see /admin (UX)
Attacker edits JS / calls API    → only the NestJS @Roles('admin') guard stops them (security)
```

Rules:

1. Every API endpoint authorises on the server (role + ownership).
2. Frontend guards and `*appHasRole` mirror those rules for a good UX.
3. Never trust client-supplied role or user id; the server derives them from the token.
4. Return `403` for forbidden, `404` for "not yours" when existence itself is sensitive.

## 5. Sessions

| Feature | Frontend | Backend |
|---|---|---|
| Access token expiry (~15 min) | refresh-on-401 interceptor | short JWT `exp` |
| Refresh token expiry (days) | redirect to login when refresh fails | cookie `Max-Age` + DB record |
| Idle auto-lock (Settings) | `IdleLock` → lock screen, re-auth | — |
| Re-auth for sensitive actions | dialog → `POST /auth/reauth` | short-lived "recently authenticated" claim |
| Logout | clear memory, call `/auth/logout` | revoke refresh token, clear cookie |
| Logout everywhere | settings → "Sign out of all devices" | revoke all refresh tokens |
| Session list | `GET /auth/sessions` | device/IP/last used |

### Lock screen vs logout

Auto-lock (confirmed setting) should **not** destroy the session; it hides content and requires the password/TOTP to continue:

```typescript
// core/layout/shell.html
@if (idle.locked()) {
  <app-lock-screen (unlocked)="idle.unlock()" />   <!-- full-screen overlay; calls /auth/reauth -->
} @else {
  <!-- normal shell -->
}
```

Clear any revealed secrets when locking.

## 6. HTTPS everywhere

- Production only over HTTPS (TLS at CloudFront/ALB/nginx); redirect HTTP → HTTPS.
- `Strict-Transport-Security: max-age=31536000; includeSubDomains` header.
- `Secure` cookies require HTTPS; `localhost` is treated as secure for development.
- `navigator.clipboard` and Web Crypto require secure contexts.

## 7. Zero-knowledge later (your confirmed roadmap)

Today: server-side encryption (AES-256-GCM, keys on the server). Later: zero-knowledge, where the browser derives a key from a master password (Web Crypto: PBKDF2/Argon2 via WASM) and encrypts before upload. Frontend implications to plan for now:

- keep all crypto behind one `VaultCrypto` service interface;
- never assume the server can search secret fields;
- design the "unlock vault" flow separately from "log in".

## 8. Common mistakes

| Mistake | Fix |
|---|---|
| JWT in localStorage | memory + httpOnly refresh cookie |
| Trusting `user.role` from the client for API decisions | server-side checks |
| Long-lived access tokens to "avoid refresh complexity" | short-lived + refresh |
| Lock screen that's just a CSS overlay over sensitive DOM | remove sensitive content from the DOM (`@if`) |
| Open redirect via `returnUrl` | allow only relative paths ([Module 10](../10-routing/02-params-and-navigation.md)) |

## 9. Interview questions

- **Q: Where should a SPA store tokens?** Short-lived access token in memory; refresh token in an httpOnly, Secure, SameSite cookie, rotated by the server.
- **Q: Do route guards secure an Angular app?** No; they're UX. The API must authorise every request.
- **Q: XSS vs CSRF?** XSS runs attacker script in your origin (steals data, acts as the user); CSRF tricks the browser into sending authenticated requests from another site (mitigated by SameSite cookies, CSRF tokens, or bearer tokens).

## 10. Completion checklist

- [ ] I can draw Myspce's token flow with storage locations.
- [ ] Lock screen removes sensitive DOM and requires re-auth.
- [ ] I can list which checks the NestJS backend must do for every endpoint.
