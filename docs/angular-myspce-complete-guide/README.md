# Angular + Myspce: The Complete Guide (Angular 22)

A complete, topic-wise Angular learning library that uses **Myspce** — your personal vault app (notes, credentials, memories, tasks, files, calendar, admin panel) — as the running example from the first component to the production build.

> **Version baseline:** Angular **22.x** (released June 2026) · TypeScript **6.0** · Node.js **^22.22 / ^24.13.1 / ^26** · RxJS **7.x** · Vitest (default test runner).
> Every code sample in this guide targets that baseline. Version-specific behaviour is labelled `v22`, `v21+`, `v20+` and so on. See [REFERENCES.md](REFERENCES.md) for the sources used to verify it.

---

## How to use this guide

1. Read [LEARNING_ROADMAP.md](LEARNING_ROADMAP.md) once. It tells you the order, how long each module takes, and where the checkpoints are.
2. Read [MYSPCE_PROJECT_CONTEXT.md](MYSPCE_PROJECT_CONTEXT.md). It explains what is **confirmed** about Myspce and what is only a **proposed example**.
3. Work through the modules in order. Each module folder has a `README.md` with its table of contents and a self-assessment checklist.
4. After each module, do the matching lab in [21-myspce-practice-labs](21-myspce-practice-labs/README.md).
5. Update [PROGRESS_TRACKER.md](PROGRESS_TRACKER.md) honestly. A topic is "Mastered" only when you can build it without looking at the notes.
6. Finish with [22-final-project-readiness](22-final-project-readiness/README.md) before you start the real Myspce frontend.

Keep [ANGULAR_CHEAT_SHEET.md](ANGULAR_CHEAT_SHEET.md) and [GLOSSARY.md](GLOSSARY.md) open in a side tab while you study.

## How each note is structured

Major topics follow the same 20-part structure, so you always know where to look:

| # | Section | # | Section |
|---|---|---|---|
| 1 | Learning objectives | 11 | Visual explanation |
| 2 | Prerequisites | 12 | Common mistakes |
| 3 | What is it? | 13 | Debugging guide |
| 4 | Why does Angular need it? | 14 | Best practices |
| 5 | How it works internally | 15 | Angular vs React |
| 6 | Syntax and API reference | 16 | Testing |
| 7 | Smallest working example | 17 | Practice questions |
| 8 | Step-by-step explanation | 18 | Interview questions |
| 9 | Myspce real-world example | 19 | Revision notes |
| 10 | Advanced example | 20 | Completion checklist |

Small topics skip sections that would only be filler.

## Table of contents

### Start here
- [Learning roadmap](LEARNING_ROADMAP.md)
- [Myspce project context](MYSPCE_PROJECT_CONTEXT.md)
- [Progress tracker](PROGRESS_TRACKER.md)
- [Glossary](GLOSSARY.md)
- [Angular cheat sheet](ANGULAR_CHEAT_SHEET.md)
- [References](REFERENCES.md)

### Modules
| Module | Topic |
|---|---|
| [00](00-prerequisites/README.md) | Prerequisites: JavaScript, TypeScript, async, DOM/CSS, npm, Git, DevTools |
| [01](01-angular-fundamentals/README.md) | Angular setup, CLI, project structure, configuration, Angular vs React |
| [02](02-components-and-templates/README.md) | Components, templates and data binding |
| [03](03-template-control-flow/README.md) | `@if`, `@for`, `@switch`, expressions and loading states |
| [04](04-component-communication/README.md) | `input()`, `output()`, `model()` and communication patterns |
| [05](05-directives/README.md) | Built-in and custom directives, host directives |
| [06](06-pipes/README.md) | Built-in and custom pipes |
| [07](07-services-and-dependency-injection/README.md) | Services, `@Service()`, `inject()`, providers, injector hierarchy |
| [08](08-signals-and-reactivity/README.md) | `signal`, `computed`, `effect`, `linkedSignal`, resources |
| [09](09-lifecycle-and-change-detection/README.md) | Lifecycle hooks, OnPush (default in v22), zoneless, cleanup |
| [10](10-routing/README.md) | Router, params, layouts, lazy loading, guards, resolvers |
| [11](11-template-driven-forms/README.md) | Template-driven forms |
| [12](12-reactive-forms/README.md) | Reactive forms, validation, dynamic forms, **Signal Forms (stable in v22)** |
| [13](13-http-and-api-integration/README.md) | HttpClient, interceptors, auth, errors, uploads, testing |
| [14](14-rxjs/README.md) | Observables, operators, subjects, interop, testing |
| [15](15-state-management/README.md) | Service and signal stores, NgRx and SignalStore |
| [16](16-advanced-components/README.md) | Content projection, queries, dynamic components, UI kit |
| [17](17-performance/README.md) | Rendering, `@defer`, bundles, measuring |
| [18](18-ssr-and-hydration/README.md) | SSR, hydration, platform-aware code |
| [19](19-testing/README.md) | Vitest, TestBed, HTTP/forms/router tests, Playwright E2E |
| [20](20-security-and-deployment/README.md) | Security, auth tokens, accessibility, i18n, deployment, CI/CD |
| [21](21-myspce-practice-labs/README.md) | 14 hands-on Myspce labs with separate solutions |
| [22](22-final-project-readiness/README.md) | Final assessment, debugging drills, implementation plan, rubric |

## Conventions used in code

- **Standalone components only** (NgModules appear only in legacy sections).
- **v20+ file naming:** `note-card.ts`, `note-card.html`, `note-card.css`, class `NoteCard` (no `.component` suffix). Older projects (like most production apps written before 2025) use `note-card.component.ts` and `NoteCardComponent`; both work.
- **OnPush everywhere:** in v22 it is the default, so new components don't need to declare it. Examples still write it explicitly where it helps you learn.
- **Signals for state**, RxJS for streams over time, resources for async data.
- **File paths** are shown at the top of every code block as a comment, relative to `src/app/`.
- Tailwind CSS v4 is used for styling (a confirmed Myspce decision).

## Not covered in depth

Angular Material, Angular Aria component APIs, i18n tooling beyond basics, Module Federation and micro frontends, and native mobile (Ionic/Capacitor) are only mentioned. Each is a large topic of its own.
