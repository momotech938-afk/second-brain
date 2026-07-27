# Lab 1 — Solution

Back to [Lab 1](../lab-01-app-shell.md) · [Labs index](../README.md)

## Final files

```typescript
// src/app/app.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  templateUrl: './app.html',
})
export class App {}
```

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';
export const routes: Routes = [];
```

```typescript
// src/environments/environment.ts
export const environment = { production: true, apiUrl: '/api/v1' };

// src/environments/environment.development.ts
export const environment = { production: false, apiUrl: '/api/v1' };
```

## Independent task — staging configuration

```typescript
// src/environments/environment.staging.ts
export const environment = { production: true, apiUrl: 'https://staging.example/api/v1' };
```

```json
// angular.json → projects.myspce-practice.architect.build.configurations
"staging": {
  "fileReplacements": [
    { "replace": "src/environments/environment.ts", "with": "src/environments/environment.staging.ts" }
  ],
  "outputHashing": "all"
}
```

```json
// package.json → scripts
"build:staging": "ng build -c staging"
```

Verify:

```bash
npm run build:staging
grep -r "staging.example" dist/myspce-practice/browser/*.js && echo "found"
```

## Hints if you got stuck

- `ng generate environments` only creates the base and development files; you add others by hand and register them in `fileReplacements`.
- Configurations inherit the build target's `options`; you only override what differs.
- If `grep` finds nothing, you probably built the default (production) configuration.
