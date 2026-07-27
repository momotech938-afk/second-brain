# Dynamic Components and the Dashboard Widget Registry

Back to [Module 16](README.md) · Previous: [View and content queries](02-queries.md) · Next: [Reusable UI kit](04-reusable-ui-kit.md)

## 1. Learning objectives

- Render components chosen at runtime with `NgComponentOutlet` and `ViewContainerRef.createComponent`.
- Set inputs, listen to outputs and apply directives to dynamic components.
- Lazy-load component classes.
- Clean up dynamically created components.
- Implement Myspce's admin-configured dashboard.

## 2. Prerequisites

[Queries](02-queries.md), [Resources](../08-signals-and-reactivity/04-resources.md).

## 3. What is it?

Creating components whose **type** isn't known until runtime — from a config object, a server response, or user choice.

## 4. Why does Angular need it?

The Myspce admin panel (confirmed feature) lets you choose which dashboard widgets appear, in what order and size. The dashboard reads that config from `/bootstrap` and must render the right components.

## 5. How it works internally

`ViewContainerRef` is an anchor in a view where Angular can insert views. `createComponent(Type, options)` creates the component (with an injector and optional projected nodes), inserts its host view, and returns a `ComponentRef` you can use to set inputs and destroy it. `NgComponentOutlet` is a directive that does the same declaratively.

## 6. Syntax and API reference

```typescript
// Declarative
<ng-container *ngComponentOutlet="component; inputs: { config: cfg }; injector: childInjector" />

// Imperative
const ref = vcr.createComponent(StatsWidget, {
  bindings: [inputBinding('config', () => this.cfg()), outputBinding('opened', id => this.open(id))],  // v20+
  directives: [FocusRing],                     // v20+: apply directives to the dynamic host
  injector,                                    // optional parent injector
  environmentInjector,                         // optional
});
ref.setInput('config', cfg);                  // alternative to bindings
ref.instance;                                 // component instance
ref.location.nativeElement;                   // host element
ref.destroy();                                // remove and destroy
vcr.clear();                                  // destroy all in this container
```

`inputBinding`, `outputBinding` and `twoWayBinding` (from `@angular/core`, v20+) wire signals to dynamic components reactively — when `this.cfg()` changes, the input updates.

## 7. Smallest working example

```typescript
// shared/ui/dynamic-host.ts
import { Component, ViewContainerRef, Type, effect, input, inject } from '@angular/core';

@Component({ selector: 'app-dynamic-host', template: '' })
export class DynamicHost {
  component = input.required<Type<unknown>>();
  private vcr = inject(ViewContainerRef);

  constructor() {
    effect(onCleanup => {
      const ref = this.vcr.createComponent(this.component());
      onCleanup(() => ref.destroy());
    });
  }
}
```

## 8. Step-by-step

1. Injecting `ViewContainerRef` in a component gives the container at its host location (new components render as siblings after the host).
2. The effect creates the component and destroys it when the input changes or the host is destroyed.

## 9. Myspce real-world example — the widget registry

### Config from the admin panel (part of `/bootstrap`)

```typescript
// core/models/widgets.ts  (proposed shape; matches the build plan's dashboard_widgets table)
export type WidgetType = 'greeting' | 'quick_actions' | 'stats' | 'recent_activity' | 'todays_tasks' | 'week_progress';

export interface WidgetConfig {
  id: string;
  type: WidgetType;
  size: 'sm' | 'md' | 'lg' | 'full';
  position: number;
  visible: boolean;
  config: Record<string, unknown>;    // widget-specific (e.g. { limit: 5 })
}
```

### Registry: type → lazy component

```typescript
// features/dashboard/widgets/widget-registry.ts
import { Type } from '@angular/core';
import { WidgetType } from '../../../core/models/widgets';

export const WIDGET_REGISTRY: Record<WidgetType, () => Promise<Type<unknown>>> = {
  greeting:        () => import('./greeting-widget').then(m => m.GreetingWidget),
  quick_actions:   () => import('./quick-actions-widget').then(m => m.QuickActionsWidget),
  stats:           () => import('./stats-widget').then(m => m.StatsWidget),
  recent_activity: () => import('./recent-activity-widget').then(m => m.RecentActivityWidget),
  todays_tasks:    () => import('./todays-tasks-widget').then(m => m.TodaysTasksWidget),
  week_progress:   () => import('./week-progress-widget').then(m => m.WeekProgressWidget),
};
```

`Record<WidgetType, …>` makes the compiler complain if a new widget type is added without a registry entry.

Every widget follows one contract:

```typescript
// each widget component
config = input<Record<string, unknown>>({});
```

### Dashboard page

```typescript
// features/dashboard/pages/dashboard-page.ts
import { Component, computed, inject, resource } from '@angular/core';
import { NgComponentOutlet } from '@angular/common';
import { BootstrapStore } from '../../../core/bootstrap/bootstrap-store';
import { WIDGET_REGISTRY } from '../widgets/widget-registry';
import { WidgetConfig } from '../../../core/models/widgets';

const SIZE: Record<WidgetConfig['size'], string> = {
  sm: 'col-span-12 md:col-span-3',
  md: 'col-span-12 md:col-span-6',
  lg: 'col-span-12 md:col-span-8',
  full: 'col-span-12',
};

@Component({
  selector: 'app-dashboard-page',
  imports: [NgComponentOutlet],
  template: `
    <div class="grid grid-cols-12 gap-4">
      @for (w of widgets.value() ?? []; track w.id) {
        <section [class]="SIZE[w.size]" [attr.aria-label]="w.type">
          <ng-container *ngComponentOutlet="w.component; inputs: { config: w.config }" />
        </section>
      }
    </div>
    @if (widgets.error()) { <p role="alert">Couldn't load the dashboard layout.</p> }
  `,
})
export default class DashboardPage {
  protected readonly SIZE = SIZE;
  private bootstrap = inject(BootstrapStore);

  private layout = computed(() =>
    (this.bootstrap.data()?.widgets ?? DEFAULT_WIDGETS).filter(w => w.visible).toSorted((a, b) => a.position - b.position),
  );

  protected widgets = resource({
    params: () => this.layout(),
    loader: ({ params }) =>
      Promise.all(params.map(async w => ({ ...w, component: await WIDGET_REGISTRY[w.type]() }))),
  });
}

const DEFAULT_WIDGETS: WidgetConfig[] = [
  { id: 'd1', type: 'greeting', size: 'full', position: 0, visible: true, config: {} },
  { id: 'd2', type: 'quick_actions', size: 'full', position: 1, visible: true, config: {} },
  { id: 'd3', type: 'stats', size: 'lg', position: 2, visible: true, config: {} },
  { id: 'd4', type: 'recent_activity', size: 'md', position: 3, visible: true, config: { limit: 5 } },
  { id: 'd5', type: 'todays_tasks', size: 'md', position: 4, visible: true, config: {} },
  { id: 'd6', type: 'week_progress', size: 'sm', position: 5, visible: true, config: {} },
];
```

(`BootstrapData` from [Module 13](../13-http-and-api-integration/01-httpclient-basics.md) would gain a `widgets: WidgetConfig[]` field.)

What this gives you:

- **Admin-driven layout:** reorder/hide/resize in the admin panel → `/bootstrap` changes → dashboard re-renders.
- **Code splitting:** each widget is its own lazy chunk; hidden widgets are never downloaded.
- **Adding a widget** = one component + one registry line + one union member.

## 10. Advanced example — imperative creation with bindings (toasts)

The toast service in the [next note](04-reusable-ui-kit.md) renders a `ToastHost` once; but imperative creation is how dialog/toast libraries attach components to `document.body`:

```typescript
import { ApplicationRef, EnvironmentInjector, createComponent, inject, inputBinding, signal } from '@angular/core';

const appRef = inject(ApplicationRef);
const env = inject(EnvironmentInjector);
const message = signal('Saved');

const ref = createComponent(ToastView, {
  environmentInjector: env,
  bindings: [inputBinding('message', message)],
});
document.body.appendChild(ref.location.nativeElement);
appRef.attachView(ref.hostView);                 // include it in change detection
// later
appRef.detachView(ref.hostView);
ref.destroy();
```

For overlays, prefer the Angular CDK (Overlay/Dialog), which handles positioning, scroll strategies, focus and cleanup.

## 11. Visual explanation

```text
/bootstrap → widgets config ─► layout (computed: visible, sorted)
                                   │
                                   ▼ resource loader: import() each widget type
                        [{ type, size, config, component }]
                                   │
                                   ▼ @for + *ngComponentOutlet (inputs: { config })
                         Greeting | QuickActions | Stats | Activity | Today | Week
```

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Statically importing every widget into the dashboard | lazy `import()` in the registry |
| String-keyed registry with no typing | `Record<WidgetType, …>` |
| Creating components imperatively without destroying them | `ref.destroy()` / `vcr.clear()` / effect cleanup |
| `appendChild` without `attachView` | component never updates |
| Setting inputs by assigning `ref.instance.x = …` on signal inputs | `ref.setInput()` or `inputBinding` |

## 13. Debugging guide

- Widget missing → is it `visible`? Is the registry import path right? Check Network for the chunk.
- `NG0303` style errors don't apply to dynamic inputs; a wrong input name with `setInput` throws `NG0303`-like runtime errors — check names.
- Dynamic component not updating → was its view attached to the app (for body-level components)?

## 14. Best practices

- Declarative `NgComponentOutlet` for templates; imperative only for overlays.
- One uniform input contract for registry components.
- Typed registries keyed by unions.

## 15. Angular vs React

`NgComponentOutlet` ≈ `const C = registry[type]; return <C config={…} />`. `React.lazy` ≈ the registry's `import()`.

## 16. Testing

```typescript
it('renders widgets in position order and skips hidden ones', async () => {
  const data = signal({ widgets: [
    { id: 'b', type: 'stats', size: 'lg', position: 2, visible: true, config: {} },
    { id: 'a', type: 'greeting', size: 'full', position: 1, visible: true, config: {} },
    { id: 'c', type: 'week_progress', size: 'sm', position: 3, visible: false, config: {} },
  ] });
  TestBed.configureTestingModule({ providers: [{ provide: BootstrapStore, useValue: { data } }] });
  const f = TestBed.createComponent(DashboardPage);
  await f.whenStable();
  const labels = [...f.nativeElement.querySelectorAll('section')].map((s: HTMLElement) => s.getAttribute('aria-label'));
  expect(labels).toEqual(['greeting', 'stats']);
});
```

(Widgets themselves may need their own fakes provided, since they inject stores.)

## 17. Practice questions

1. Add a `quote` widget type end to end.
2. Let widgets emit `configChange` so the admin preview can edit them inline (hint: `outputBinding`).
3. Why should the registry use lazy imports?

## 18. Interview questions

- **Q: How do you render a component chosen at runtime?** `NgComponentOutlet` in templates, or `ViewContainerRef.createComponent` / `createComponent` imperatively.
- **Q: How do you set inputs on a dynamic component?** `ComponentRef.setInput()`, `inputs` on `NgComponentOutlet`, or `inputBinding` in v20+.

## 19. Revision notes

`*ngComponentOutlet="C; inputs: {...}"` · `vcr.createComponent(C, { bindings, directives })` · `setInput` · `destroy` · typed lazy registry · admin config → layout → resource → outlet.

## 20. Completion checklist

- [ ] Dashboard renders from widget config.
- [ ] Each widget is a lazy chunk.
- [ ] Adding a widget type is a three-line change.
