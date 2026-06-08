# Amplify Client Factories — Monorepo

This reference documents the client factory pattern used in Amplify Gen 2 + Nx monorepo projects. It is consumed by `amplify-monorepo` and `amplify-action` when running in a monorepo context.

---

## Architecture

The monorepo uses a shared factory in `packages/core` that both apps (`apps/web` and `apps/admin`) consume. Each app then exposes two typed client functions: `getGuestClient` and `getAuthenticatedClient`.

```
packages/core/src/auth/client.ts   ← shared factory
apps/web/src/lib/client.ts         ← typed client for web app
apps/admin/src/lib/client.ts       ← typed client for admin app
```

---

## `packages/core/src/auth/client.ts`

```typescript
import { generateServerClientUsingCookies } from '@aws-amplify/adapter-nextjs/api';
import type { GraphQLAuthMode } from '@aws-amplify/core/internals/utils';
import type { NextServer } from '@aws-amplify/adapter-nextjs';
import type { ReadonlyRequestCookies } from 'next/dist/server/web/spec-extension/adapters/request-cookies';

/**
 * Factory that creates a typed Amplify server client bound to the current request cookies.
 * Each app passes its own `outputs` and `Schema` type parameter.
 */
export const createServerClient =
  <Schema extends Record<string | number | symbol, unknown> = never>(
    outputs: NextServer.CreateServerRunnerInput['config'],
    cookies: () => Promise<ReadonlyRequestCookies>,
  ) =>
  (config?: { authMode?: GraphQLAuthMode; authToken?: string }) =>
    generateServerClientUsingCookies<Schema>({ ...config, config: outputs, cookies });
```

---

## `apps/{app}/src/lib/client.ts`

Each app has its own `client.ts`. Mark it `'use server'` — it reads cookies.

```typescript
'use server';

import { cookies } from 'next/headers';
import { createServerClient } from '@<project>/core/auth/client';
import config from '@config/amplify_outputs.json';
import type { Schema } from '@amplify/data/resource';

/** Amplify client with public API key auth — for unauthenticated operations. */
export async function getGuestClient() {
  return createServerClient<Schema>(config, cookies)({ authMode: 'apiKey' });
}

/** Amplify client with Cognito userPool auth — requires an active session. */
export async function getAuthenticatedClient() {
  return createServerClient<Schema>(config, cookies)({ authMode: 'userPool' });
}
```

Replace `<project>` with the actual monorepo package scope (e.g., `@myapp`).

---

## Usage in Server Actions

```typescript
// apps/web/src/actions/orders.ts
'use server';

import { getAuthenticatedClient } from '../lib/client';

export async function getOrders(): Promise<Result<Order[], AppError>> {
  try {
    const client = await getAuthenticatedClient();
    const { data, errors } = await client.models.Order.list();

    if (errors?.length) {
      return { ok: false, error: { type: 'unexpected', cause: errors } };
    }

    return { ok: true, value: data };
  } catch (cause) {
    return { ok: false, error: { type: 'unexpected', cause } };
  }
}
```

---

## Rules

- `getGuestClient` and `getAuthenticatedClient` are only called from `src/actions/` — never from components.
- `amplify_outputs.json` is shared between both apps via `config/amplify_outputs.json`. Never duplicate it.
- The `Schema` type is imported from `apps/admin/amplify/data/resource.ts` — admin owns the backend. The web app consumes the same schema.
- Path alias `@config/*` maps to `config/` in `tsconfig.base.json` so both apps can import `amplify_outputs.json` with the same import path.
