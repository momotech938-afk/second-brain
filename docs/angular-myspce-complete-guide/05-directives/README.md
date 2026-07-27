# Module 05 — Directives

Directives add reusable behaviour to elements: copy-to-clipboard, autofocus, click-outside, role-based rendering.

Back to [README](../README.md) · Previous: [Module 04](../04-component-communication/README.md) · Next: [Module 06](../06-pipes/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Built-in directives](01-built-in-directives.md) | attribute vs structural, `NgClass`, `NgStyle`, `NgTemplateOutlet`, `NgComponentOutlet`, router directives |
| 2 | [Custom attribute directives](02-custom-attribute-directives.md) | `@Directive`, inputs/outputs, `host` bindings, `ElementRef`, `Renderer2`, DI inside directives, legacy `@HostListener`/`@HostBinding` |
| 3 | [Structural and host directives](03-structural-and-host-directives.md) | `ng-template`, `TemplateRef`, `ViewContainerRef`, custom structural directives, `hostDirectives` composition |

**Time:** ~3 h reading + 2 h practice.

## Self-assessment checklist

- [ ] I can explain the difference between attribute and structural directives, and between a directive and a component.
- [ ] I know which built-in directives are still useful when `@if`/`@for` and `[class]` exist.
- [ ] I can write an attribute directive with inputs, an output and `host` bindings.
- [ ] I can safely touch the DOM with `ElementRef`/`Renderer2`, and explain when not to.
- [ ] I can inject services into a directive and clean up listeners with `DestroyRef`.
- [ ] I can write a structural directive with `TemplateRef` and `ViewContainerRef`.
- [ ] I can compose behaviours with `hostDirectives`.
- [ ] I can test a directive with a host test component.
- [ ] I built `CopyToClipboard`, `Autofocus`, `ClickOutside` and `HasRole` for Myspce.
