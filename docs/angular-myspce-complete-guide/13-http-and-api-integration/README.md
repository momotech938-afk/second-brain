# Module 13 — HTTP Client, Interceptors and Error Handling

Talking to the (planned) Myspce NestJS API: typed requests, auth headers and token refresh, global error handling, retries, uploads, pagination and HTTP tests.

Back to [README](../README.md) · Previous: [Module 12](../12-reactive-forms/README.md) · Next: [Module 14](../14-rxjs/README.md)

> **v22 changes that matter here:** `HttpClient` now uses the **Fetch API** by default (`withFetch()` is deprecated; `withXhr()` restores XMLHttpRequest). `reportProgress` is deprecated in favour of `reportUploadProgress` / `reportDownloadProgress`, and **upload progress needs `withXhr()`**. Since v21, `HttpClient` can be injected without `provideHttpClient()`; you still call it to add interceptors.

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [HttpClient basics](01-httpclient-basics.md) | configuration, verbs, typed requests, params, headers, options, environment URLs, API service architecture, runtime validation |
| 2 | [Interceptors and auth](02-interceptors-and-auth.md) | functional interceptors, order, auth header, token refresh with race-condition handling, `HttpContext`, logout |
| 3 | [Error handling and retry](03-error-handling-retry.md) | `HttpErrorResponse`, global error interceptor, user messages, retry with backoff, timeouts, cancellation, CORS and network failures |
| 4 | [Uploads, pagination and testing](04-uploads-pagination-testing.md) | presigned S3 uploads with progress, downloads, server pagination/sorting/filtering, loading tracking, `HttpTestingController` |

**Labs:** [Lab 6 — Typed API services](../21-myspce-practice-labs/lab-06-typed-api-services.md) · [Lab 7 — UI states](../21-myspce-practice-labs/lab-07-ui-states.md) · [Lab 8 — Authentication](../21-myspce-practice-labs/lab-08-authentication.md)

**Time:** ~5 h reading + 3 h practice.

> **Never** put secrets (JWT signing keys, OAuth client secrets, DB passwords, encryption keys) in frontend code or `environment.ts`. Everything shipped to the browser is public.

## Self-assessment checklist

- [ ] I can write typed GET/POST/PATCH/DELETE calls with params and headers.
- [ ] I can explain how `HttpClient` Observables behave (cold, one request per subscription, complete after response).
- [ ] I can write functional interceptors and register them in the right order.
- [ ] I can attach the access token only to Myspce API requests.
- [ ] I can implement refresh-on-401 so that five parallel 401s trigger **one** refresh.
- [ ] I can turn `HttpErrorResponse`s into user-friendly messages in one place.
- [ ] I can retry idempotent requests with exponential backoff and add timeouts.
- [ ] I can upload a file to a presigned URL with progress and without leaking the auth header.
- [ ] I can implement server-side pagination, sorting and filtering.
- [ ] I can test services and interceptors with `HttpTestingController`.
