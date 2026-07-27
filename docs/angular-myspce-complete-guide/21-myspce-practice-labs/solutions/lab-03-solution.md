# Lab 3 — Solution

Back to [Lab 3](../lab-03-forms-and-controls.md) · [Labs index](../README.md)

## `PasswordInput` (FormValueControl)

```typescript
// shared/forms/password-input.ts
import { Component, computed, input, model, output, signal } from '@angular/core';
import type { FormValueControl } from '@angular/forms/signals';

@Component({
  selector: 'app-password-input',
  template: `
    <div class="flex items-center gap-2 rounded-lg border border-slate-300 px-3 py-2 focus-within:ring-2 focus-within:ring-blue-500 dark:border-slate-700"
         [class.opacity-50]="disabled()">
      <input [id]="inputId()" class="min-w-0 flex-1 bg-transparent outline-none" [type]="visible() ? 'text' : 'password'"
             [value]="value()" [disabled]="disabled()" [attr.autocomplete]="autocomplete()"
             (input)="value.set($any($event.target).value)" (blur)="touch.emit()" />
      <button type="button" class="text-sm text-slate-500" (click)="visible.update(v => !v)" [attr.aria-pressed]="visible()"
              [attr.aria-label]="visible() ? 'Hide password' : 'Show password'">{{ visible() ? 'Hide' : 'Show' }}</button>
    </div>
    @if (showStrength() && value()) {
      <div class="mt-1 h-1 rounded bg-slate-200 dark:bg-slate-800" aria-hidden="true">
        <div class="h-1 rounded transition-all" [style.width.%]="strength() * 25"
             [class]="strength() < 2 ? 'bg-rose-500' : strength() < 4 ? 'bg-amber-500' : 'bg-emerald-500'"></div>
      </div>
      <p class="sr-only" aria-live="polite">Password strength {{ strength() }} of 4</p>
    }
  `,
})
export class PasswordInput implements FormValueControl<string> {
  readonly value = model('');
  readonly disabled = input(false);
  readonly touched = input(false);
  readonly touch = output<void>();

  inputId = input('password');
  autocomplete = input<'current-password' | 'new-password' | 'off'>('current-password');
  showStrength = input(false);

  protected visible = signal(false);
  protected strength = computed(() => {
    const v = this.value();
    return [v.length >= 12, /[A-Z]/.test(v) && /[a-z]/.test(v), /\d/.test(v), /[^A-Za-z0-9]/.test(v)].filter(Boolean).length;
  });
}
```

## `LoginPage`

```typescript
// features/auth/login-page.ts
import { Component, inject, signal } from '@angular/core';
import { Router } from '@angular/router';
import { form, FormField, required, email, minLength, submit } from '@angular/forms/signals';
import { PasswordInput } from '../../shared/forms/password-input';

@Component({
  selector: 'app-login-page',
  imports: [FormField, PasswordInput],
  templateUrl: './login-page.html',
})
export default class LoginPage {
  private router = inject(Router);
  protected model = signal({ email: '', password: '' });
  protected loginForm = form(this.model, p => {
    required(p.email, { message: 'Email is required.' });
    email(p.email, { message: 'Enter a valid email address.' });
    required(p.password, { message: 'Password is required.' });
    minLength(p.password, 8, { message: 'Password is at least 8 characters.' });
  });

  async onSubmit(e: Event) {
    e.preventDefault();
    await submit(this.loginForm, async f => {
      await new Promise(r => setTimeout(r, 600));             // mock latency (real AuthStore in Lab 8)
      if (f().value().email === 'fail@example.com') {
        return { kind: 'invalidCredentials', message: 'Email or password is incorrect.' };
      }
      await this.router.navigateByUrl('/dashboard');
      return undefined;
    });
  }
}
```

```html
<!-- features/auth/login-page.html -->
<main class="grid min-h-dvh place-items-center p-4">
  <form (submit)="onSubmit($event)" novalidate class="w-full max-w-sm space-y-4 rounded-2xl bg-white p-6 shadow-sm dark:bg-slate-900">
    <h1 class="text-xl font-semibold">Sign in to MySpace</h1>

    @let em = loginForm.email();
    <div>
      <label for="email" class="text-sm font-medium">Email</label>
      <input id="email" type="email" autocomplete="username" [formField]="loginForm.email"
             class="mt-1 w-full rounded-lg border border-slate-300 px-3 py-2 dark:border-slate-700 dark:bg-slate-800"
             [attr.aria-invalid]="em.touched() && em.invalid()" aria-describedby="email-errors" />
      @if (em.touched() && em.invalid()) {
        <ul id="email-errors" class="mt-1 text-sm text-rose-600">@for (e of em.errors(); track e.kind) { <li>{{ e.message }}</li> }</ul>
      }
    </div>

    @let pw = loginForm.password();
    <div>
      <label for="password" class="text-sm font-medium">Password</label>
      <app-password-input inputId="password" autocomplete="current-password" [formField]="loginForm.password" />
      @if (pw.touched() && pw.invalid()) {
        <ul class="mt-1 text-sm text-rose-600">@for (e of pw.errors(); track e.kind) { <li>{{ e.message }}</li> }</ul>
      }
    </div>

    @for (e of loginForm().errors(); track e.kind) {
      <p role="alert" class="rounded-lg bg-rose-50 p-2 text-sm text-rose-700 dark:bg-rose-950 dark:text-rose-200">{{ e.message }}</p>
    }

    <button type="submit" class="w-full rounded-lg bg-blue-600 py-2 font-medium text-white disabled:opacity-50" [disabled]="loginForm().submitting()">
      {{ loginForm().submitting() ? 'Signing in…' : 'Sign in' }}
    </button>
    <a href="/api/v1/auth/google" class="block rounded-lg border py-2 text-center text-sm">Continue with Google</a>
  </form>
</main>
```

## `CredentialForm`

Use the Signal Forms version from [Module 12 note 4](../../12-reactive-forms/04-signal-forms.md#9-myspce-real-world-example--credential-form-with-signal-forms), minus `validateHttp` (no backend yet). Key parts:

```typescript
protected model = linkedSignal<CredentialModel>(() => empty());
protected credForm = form(this.model, p => {
  required(p.name, { message: 'Name is required.' });
  pattern(p.url, /^(https?:\/\/\S+)?$/, { message: 'Start with http:// or https://' });
  required(p.username, { message: 'Username is required.' });
  required(p.password, { message: 'Password is required.' });
  minLength(p.password, 8, { message: 'Use at least 8 characters.' });
}, {
  submission: {
    action: async f => {
      const v = f().value();
      this.submitted.emit({ ...v, url: v.url || null, notes: v.notes || null, categoryId: v.categoryId || null });
      return undefined;
    },
    onInvalid: f => f().errorSummary()[0]?.fieldTree().focusBoundControl(),
  },
});
```

## Independent task — change password

```typescript
// features/settings/tabs/security-tab.ts
import { Component, signal } from '@angular/core';
import { form, FormField, FormRoot, required, validate } from '@angular/forms/signals';
import { PasswordInput } from '../../../shared/forms/password-input';

function missingParts(v: string): string[] {
  return [
    v.length < 12 && 'at least 12 characters',
    !/[A-Z]/.test(v) && 'an uppercase letter',
    !/[a-z]/.test(v) && 'a lowercase letter',
    !/\d/.test(v) && 'a number',
    !/[^A-Za-z0-9]/.test(v) && 'a symbol',
  ].filter((x): x is string => !!x);
}

@Component({
  selector: 'app-security-tab',
  imports: [FormField, FormRoot, PasswordInput],
  template: `
    <form [formRoot]="pwForm" class="max-w-md space-y-4">
      <label class="text-sm font-medium" for="cur">Current password</label>
      <app-password-input inputId="cur" [formField]="pwForm.current" />
      <label class="text-sm font-medium" for="new">New password</label>
      <app-password-input inputId="new" autocomplete="new-password" [showStrength]="true" [formField]="pwForm.next" />
      @if (pwForm.next().touched() && pwForm.next().getError('weak'); as e) { <p class="text-sm text-rose-600">{{ e.message }}</p> }
      <label class="text-sm font-medium" for="confirm">Confirm new password</label>
      <app-password-input inputId="confirm" autocomplete="new-password" [formField]="pwForm.confirm" />
      @if (pwForm.confirm().touched() && pwForm.confirm().getError('mismatch'); as e) { <p class="text-sm text-rose-600">{{ e.message }}</p> }
      <button type="submit" class="rounded-lg bg-blue-600 px-4 py-2 text-white" [disabled]="pwForm().submitting()">Change password</button>
      @if (done()) { <p role="status" class="text-sm text-emerald-600">Password changed.</p> }
    </form>
  `,
})
export default class SecurityTab {
  protected done = signal(false);
  protected model = signal({ current: '', next: '', confirm: '' });
  protected pwForm = form(this.model, p => {
    required(p.current, { message: 'Enter your current password.' });
    required(p.next, { message: 'Enter a new password.' });
    validate(p.next, ({ value }) => {
      const missing = missingParts(value());
      return value() && missing.length ? { kind: 'weak', message: `Add ${missing.join(', ')}.` } : undefined;
    });
    required(p.confirm, { message: 'Confirm the new password.' });
    validate(p.confirm, ({ value, valueOf }) =>
      value() && value() !== valueOf(p.next) ? { kind: 'mismatch', message: "Passwords don't match." } : undefined,
    );
  }, {
    submission: { action: async () => { await new Promise(r => setTimeout(r, 400)); this.done.set(true); return undefined; } },
  });
}
```

Hints: `valueOf(p.next)` inside the confirm rule creates the dependency that re-validates confirm when "new" changes; returning `undefined` means valid.
