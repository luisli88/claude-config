---
name: amplify-action
description: Generate a Next.js Server Action connected to an Amplify Gen 2 AppSync operation — Zod input validation, Result<T,E> return type, Amplify client call, and unit test scaffold.
argument-hint: "Domain name (e.g. contact, orders) and auth mode (guest|cognito|groups)"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse:
- **Domain name**: camelCase or kebab-case. Required. Used as the file name (`src/actions/{domain}.ts`) and to name the exported function. Ask if not provided.
- **Auth mode**: `guest`, `cognito`, or `groups`. Required — ask if not provided.

---

## FILES TO CREATE

### `src/actions/{domain}.ts`

Select the template based on auth mode.

#### `guest` — Public API key

```typescript
'use server';

import { generateClient } from 'aws-amplify/data';
import { Amplify } from 'aws-amplify';
import outputs from '@/../amplify_outputs.json';
import { type Schema } from '@/../amplify/data/resource';
import { {domain}Schema, type {Domain}Input } from '@/lib/schemas/{domain}';
import { type Result, type AppError } from '@/lib/types';

Amplify.configure(outputs, { ssr: true });

const client = generateClient<Schema>({ authMode: 'apiKey' });

/**
 * {Description of what the action does}.
 * Validates input server-side before calling Amplify.
 */
export async function {actionName}(input: {Domain}Input): Promise<Result<void, AppError>> {
  const parsed = {domain}Schema.safeParse(input);

  if (!parsed.success) {
    return {
      ok: false,
      error: { type: 'validation', fields: parsed.error.flatten().fieldErrors as Record<string, string> },
    };
  }

  try {
    const { errors } = await client.mutations.{operationName}(parsed.data);

    if (errors?.length) {
      return { ok: false, error: { type: 'unexpected', cause: errors } };
    }

    return { ok: true, value: undefined };
  } catch (cause) {
    return { ok: false, error: { type: 'unexpected', cause } };
  }
}
```

#### `cognito` / `groups` — Authenticated user context

```typescript
'use server';

import { cookies } from 'next/headers';
import { createServerRunner } from '@aws-amplify/adapter-nextjs';
import { generateServerClientUsingCookies } from '@aws-amplify/adapter-nextjs/api';
import outputs from '@/../amplify_outputs.json';
import { type Schema } from '@/../amplify/data/resource';
import { {domain}Schema, type {Domain}Input } from '@/lib/schemas/{domain}';
import { type Result, type AppError } from '@/lib/types';

const { runWithAmplifyServerContext } = createServerRunner({ config: outputs });

/**
 * {Description of what the action does}.
 * Requires an authenticated Cognito session.
 */
export async function {actionName}(input: {Domain}Input): Promise<Result<void, AppError>> {
  const parsed = {domain}Schema.safeParse(input);

  if (!parsed.success) {
    return {
      ok: false,
      error: { type: 'validation', fields: parsed.error.flatten().fieldErrors as Record<string, string> },
    };
  }

  try {
    return await runWithAmplifyServerContext({
      nextServerContext: { cookies },
      async operation(contextSpec) {
        const client = generateServerClientUsingCookies<Schema>({
          config: outputs,
          cookies,
          authMode: 'userPool',
        });

        const { errors } = await client.mutations.{operationName}(parsed.data);

        if (errors?.length) {
          return { ok: false, error: { type: 'unexpected', cause: errors } };
        }

        return { ok: true, value: undefined };
      },
    });
  } catch (cause) {
    return { ok: false, error: { type: 'unexpected', cause } };
  }
}
```

---

### `src/lib/schemas/{domain}.ts`

```typescript
import { z } from 'zod';

/** Validation schema for {domain} operations. Used on client and server. */
export const {domain}Schema = z.object({
  // Add fields matching the AppSync mutation arguments
});

export type {Domain}Input = z.infer<typeof {domain}Schema>;
```

---

### `src/lib/types.ts` — Create if it doesn't exist

```typescript
/** Generic Result type for operations that can fail with a typed error. */
export type Result<T, E = AppError> =
  | { ok: true; value: T }
  | { ok: false; error: E };

/** Application-level errors with discriminated union for exhaustive handling. */
export type AppError =
  | { type: 'validation'; fields: Record<string, string> }
  | { type: 'not_found'; resource: string }
  | { type: 'unauthorized' }
  | { type: 'conflict'; message: string }
  | { type: 'unexpected'; cause: unknown };
```

If `src/lib/types.ts` already exists, add only the missing types — do not overwrite.

---

### `src/actions/__tests__/{domain}.test.ts`

```typescript
import { {actionName} } from '../{domain}';
import { generateClient } from 'aws-amplify/data';

jest.mock('aws-amplify/data');
jest.mock('@/../amplify_outputs.json', () => ({}));
jest.mock('aws-amplify', () => ({ Amplify: { configure: jest.fn() } }));

const mockClient = {
  mutations: {
    {operationName}: jest.fn(),
  },
};

(generateClient as jest.Mock).mockReturnValue(mockClient);

describe('{actionName}', () => {
  it('returns validation error for invalid input', async () => {
    const result = await {actionName}({} as any);
    expect(result.ok).toBe(false);
    if (!result.ok) expect(result.error.type).toBe('validation');
  });

  it('returns ok on successful mutation', async () => {
    mockClient.mutations.{operationName}.mockResolvedValue({ data: 'ok', errors: null });

    const result = await {actionName}({
      // provide a valid input matching the schema
    } as any);

    expect(result.ok).toBe(true);
  });
});
```

---

## RULES

- Server Actions always re-validate with Zod, even if the client already validated. Never trust client-sent data.
- Server Actions always return `Result<T, AppError>`. Never throw for business logic errors.
- `generateClient` is instantiated at module level (singleton) for `guest` mode. For `cognito`/`groups`, create the client inside `runWithAmplifyServerContext`.
- The Zod schema in `src/lib/schemas/{domain}.ts` is the single source of validation — used on both client (react-hook-form resolver) and server (Server Action re-validation). Never duplicate.
- `src/lib/types.ts` is the single source for `Result<T,E>` and `AppError`. Never redefine them inline.
- `AppError` variants cover: `validation`, `not_found`, `unauthorized`, `conflict`, `unexpected`. Add new variants only when the existing ones don't fit.

---

## CHECKLIST

- [ ] `src/actions/{domain}.ts` created with correct auth mode template
- [ ] `src/lib/schemas/{domain}.ts` created with Zod schema and inferred type exported
- [ ] `src/lib/types.ts` has `Result<T,E>` and `AppError` (create or verify)
- [ ] Unit test created under `src/actions/__tests__/`
- [ ] Server Action re-validates with Zod before calling Amplify
- [ ] Return type is `Promise<Result<T, AppError>>` — no throws for business errors
