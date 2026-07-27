# Module 01 — Angular Setup, CLI and Project Structure

How Angular is put together, how to create and run a project, what every generated file does, and how Angular's thinking differs from React's.

Back to [README](../README.md) · Previous: [Module 00](../00-prerequisites/README.md) · Next: [Module 02](../02-components-and-templates/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Architecture and mental model](01-architecture-mental-model.md) | components, templates, DI, router, compiler, change detection; standalone vs NgModule |
| 2 | [Setup and CLI](02-setup-and-cli.md) | Node/TypeScript versions, installing the CLI, `ng new`, generators, dev server |
| 3 | [Project structure and bootstrap](03-project-structure-and-bootstrap.md) | every generated file; `main.ts → app.config.ts → App` |
| 4 | [Configuration and builds](04-configuration-and-builds.md) | `angular.json`, `tsconfig`, environments, build output, budgets, proxies, troubleshooting |
| 5 | [Angular vs React](05-angular-vs-react.md) | concept-by-concept mapping and the mindset shift |

**Lab:** [Lab 1 — Create the Angular application shell](../21-myspce-practice-labs/lab-01-app-shell.md)

**Time:** ~3 h reading + 2 h practice.

## Self-assessment checklist

- [ ] I can state Angular 22's supported Node and TypeScript versions and check mine.
- [ ] I can create a new Angular 22 app and explain each option `ng new` asked me.
- [ ] I can trace what happens from `index.html` to the first rendered component.
- [ ] I can explain what `app.config.ts` is for and add a provider to it.
- [ ] I can generate a component, service, guard and pipe with the CLI using the v20+ naming style.
- [ ] I can explain the difference between `ng serve` and `ng build`, and find the production output.
- [ ] I can set up an environment-specific API URL and a dev-server proxy to `/api`.
- [ ] I can read `angular.json` and change a budget.
- [ ] I can explain standalone components vs NgModules, and recognise an NgModule app.
- [ ] I can list five concrete differences between Angular and React.
