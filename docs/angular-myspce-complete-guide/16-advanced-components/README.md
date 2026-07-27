# Module 16 — Content Projection and Advanced Components

The techniques behind reusable UI: slots, templates, queries, dynamic components, overlays — and the Myspce UI kit (page header, modal, confirm, toast, tabs, dropdown, widget registry).

Back to [README](../README.md) · Previous: [Module 15](../15-state-management/README.md) · Next: [Module 17](../17-performance/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Content projection](01-content-projection.md) | `ng-content`, multiple slots, `ngProjectAs`, conditional projection, `ng-template`, `ng-container`, `ngTemplateOutlet` |
| 2 | [View and content queries](02-queries.md) | `viewChild`, `viewChildren`, `contentChild`, `contentChildren`, `read`, timing, legacy decorators |
| 3 | [Dynamic components](03-dynamic-components.md) | `ViewContainerRef.createComponent`, `NgComponentOutlet`, inputs/outputs on dynamic components, cleanup, the admin-driven dashboard widget registry |
| 4 | [Reusable UI kit](04-reusable-ui-kit.md) | modal/dialog, confirm service, toast service, tabs, dropdown menu, CDK Overlay and Angular Aria, accessibility and keyboard support, API design |

**Lab:** [Lab 11 — Dialogs, notifications and UI components](../21-myspce-practice-labs/lab-11-dialogs-and-toasts.md)

**Time:** ~4 h reading + 3 h practice.

## Self-assessment checklist

- [ ] I can build a component with default and named projection slots.
- [ ] I know why projected content is created even when not displayed, and how `ng-template` avoids it.
- [ ] I can pass templates into a component and render them with context.
- [ ] I can query elements, components and directives with signal queries and know when they resolve.
- [ ] I can create components dynamically, set inputs, subscribe to outputs, and destroy them.
- [ ] I can implement the dashboard widget registry from admin config.
- [ ] I can build an accessible modal with focus trap, Escape to close, and focus return.
- [ ] I can build a toast service and a promise-based confirm dialog.
- [ ] I can design component APIs that are hard to misuse.
