---
name: amplify-function
description: Add a Lambda function domain to an Amplify Gen 2 backend — resource.ts with defineFunction, typed handler deriving its type from the AppSync schema, and IAM policy registration in backend.ts.
argument-hint: "Domain name (kebab-case, e.g. contact-email) and IAM actions needed (e.g. 'ses:SendEmail ses:SendRawEmail')"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse:
- **Domain name**: kebab-case. Required. Used as the directory name and Lambda function name. Ask if not provided.
- **IAM actions**: space or comma-separated list of AWS action strings. Optional — if not provided, create the function without IAM grants and note that they must be added manually in `backend.ts`.

Derive `camelCaseName` from the domain name for the exported variable.

---

## FILES TO CREATE

### `amplify/functions/{domain-name}/resource.ts`

```typescript
import { defineFunction, secret } from '@aws-amplify/backend';

/** Lambda function for the {domain-name} domain. */
export const {camelCaseName} = defineFunction({
  name: '{domain-name}',
  timeoutSeconds: 15,
  environment: {
    // Inject non-secret config values here.
    // Reference secrets with secret('NAME') — never plain strings.
  },
});
```

### `amplify/functions/{domain-name}/handler.ts`

```typescript
import type { Schema } from '../../data/resource';

/**
 * Handler for the {operationName} AppSync operation.
 * Type is derived from the schema — arguments and return type are inferred
 * from the operation declared in data/resource.ts.
 */
export const handler: Schema['{operationName}']['functionHandler'] = async (event) => {
  const {  } = event.arguments; // typed from schema — no manual interface needed

  // Business logic here.
  // Use AWS SDK clients directly (SES, S3, DynamoDB, etc.) — not the Amplify client.

  return null;
};
```

Replace `{operationName}` with the AppSync mutation or query name that invokes this function. If the operation name is not known yet, leave it as a placeholder and add a comment.

---

## FILES TO UPDATE

### `amplify/backend.ts` — Register the function and grant IAM permissions

Add the import and include the function in `defineBackend`. Then add the IAM policy if actions were provided:

```typescript
import { defineBackend } from '@aws-amplify/backend';
import { auth } from './auth/resource';
import { data } from './data/resource';
import { {camelCaseName} } from './functions/{domain-name}/resource';
import { PolicyStatement, Effect } from 'aws-cdk-lib/aws-iam';

const backend = defineBackend({
  auth,
  data,
  {camelCaseName},
  // ... other functions
});

// Grant IAM permissions here — not in resource.ts
backend.{camelCaseName}.resources.lambda.addToRolePolicy(
  new PolicyStatement({
    effect: Effect.ALLOW,
    actions: [{iam-actions-as-string-array}],
    resources: ['*'],
  }),
);
```

Only add the `addToRolePolicy` block if IAM actions were provided.

---

## SHARED UTILITIES

If the function needs shared logic (email sending, common types, validation):

```
amplify/functions/shared/
├── email/
│   └── ses.ts          # SES client + sendEmail helper
├── common/
│   ├── types.ts        # Shared input/output types
│   └── validation.ts   # Shared Zod validation
└── {domain}/
    └── types.ts        # Domain-specific types
```

Import shared utilities using relative paths: `import { sendEmail } from '../shared/email/ses'`.

---

## RULES

- Handler type always comes from `Schema['{operationName}']['functionHandler']` — never from `aws-lambda`. This ensures `event.arguments` is typed from the schema.
- Use `async/await` — never callbacks.
- Return `null` for void mutations (AppSync requires a return type; use `a.string()` as nullable in the schema).
- Keep handlers under 100 lines. Extract shared logic to `amplify/functions/shared/`.
- IAM permissions are always granted in `backend.ts`, never in `resource.ts`.
- Secrets go in `ampx secret set` and are referenced with `secret('NAME')` in `resource.ts` — never as plain strings in `environment`.
- `amplify/package.json` must have `{ "type": "module" }` for Amplify functions to work with ESM.

---

## CHECKLIST

- [ ] `resource.ts` uses `defineFunction` with `name` matching the kebab-case domain
- [ ] `handler.ts` imports its type from `../../data/resource` (not from `aws-lambda`)
- [ ] Function registered in `defineBackend()` in `backend.ts`
- [ ] IAM permissions granted via `addToRolePolicy` in `backend.ts`
- [ ] No secrets as plain strings in `environment` — use `secret('NAME')`
- [ ] `amplify/package.json` has `{ "type": "module" }`
