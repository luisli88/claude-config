# Error Handling Pattern — Result<T,E> + AppError

This reference documents the error handling pattern used across Server Actions and services. It is consumed by `amplify-nextjs` and `amplify-action`.

---

## Type definitions — `src/lib/types.ts`

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

---

## Rules

- `Result<T,E>` and `AppError` are defined once in `src/lib/types.ts`. Never redefine inline.
- Server Actions always return `Promise<Result<T, AppError>>`. Never throw for business logic errors.
- Throw only for truly unexpected runtime errors (programming mistakes). These are caught by React Error Boundaries.
- `AppError` variants cover the common cases. Add a new variant only when none of the existing ones fit.
- `unexpected` is the catch-all — always include it in `catch` blocks.

---

## Usage in Server Actions

```typescript
export async function myAction(input: Input): Promise<Result<MyData, AppError>> {
  // 1. Validate input
  const parsed = schema.safeParse(input);
  if (!parsed.success) {
    return {
      ok: false,
      error: { type: 'validation', fields: parsed.error.flatten().fieldErrors as Record<string, string> },
    };
  }

  try {
    // 2. Business logic
    const data = await someOperation(parsed.data);
    if (!data) {
      return { ok: false, error: { type: 'not_found', resource: 'MyResource' } };
    }

    return { ok: true, value: data };
  } catch (cause) {
    return { ok: false, error: { type: 'unexpected', cause } };
  }
}
```

---

## Consuming Results in components

```typescript
// In a Server Component or Client Component
const result = await myAction(data);

if (!result.ok) {
  switch (result.error.type) {
    case 'validation':
      // show field errors
      break;
    case 'unauthorized':
      redirect('/login');
      break;
    default:
      // show generic error
  }
  return;
}

// result.value is available and typed here
const { value } = result;
```
