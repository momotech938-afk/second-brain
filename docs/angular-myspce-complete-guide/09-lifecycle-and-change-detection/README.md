# Module 09 — Lifecycle Hooks and Change Detection

When Angular creates, updates and destroys components, how it decides what to re-render (OnPush is the **default** in v22, apps are zoneless by default since v21), and how to avoid leaks.

Back to [README](../README.md) · Previous: [Module 08](../08-signals-and-reactivity/README.md) · Next: [Module 10](../10-routing/README.md)

## Notes

| # | Note | You'll learn |
|---|---|---|
| 1 | [Lifecycle hooks](01-lifecycle-hooks.md) | every hook in order, `afterNextRender` / `afterEveryRender`, which hooks you still need with signals |
| 2 | [Change detection](02-change-detection.md) | how CD works, OnPush vs Eager, signals and CD, Zone.js vs zoneless, `ChangeDetectorRef`, immutable data |
| 3 | [Cleanup and memory leaks](03-cleanup-and-memory-leaks.md) | `DestroyRef`, `takeUntilDestroyed`, timers, listeners, third-party libraries, finding leaks |

**Time:** ~3 h reading + 2 h practice.

## Self-assessment checklist

- [ ] I can list the lifecycle hooks in order and say what each is for.
- [ ] I can choose between `ngOnInit`, `constructor`, `afterNextRender` and `effect` for a given task.
- [ ] I can explain exactly what triggers an OnPush component to refresh.
- [ ] I can explain what `ChangeDetectionStrategy.Eager` is and why v22 migrations add it.
- [ ] I can explain how a zoneless app knows when to render.
- [ ] I know when (rarely) to call `markForCheck()` and why `detectChanges()` is almost never needed.
- [ ] I can fix a "UI doesn't update" bug caused by mutation or a non-signal field.
- [ ] I can clean up timers, subscriptions, listeners and library instances.
- [ ] I can find a memory leak with Chrome DevTools heap snapshots.
