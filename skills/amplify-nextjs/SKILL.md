---
name: "amplify-nextjs"
description: "Scaffold a new AWS Amplify Gen 2 + Next.js App Router project following Luis's architecture patterns: standalone app, atomic design, next-intl i18n, Server Actions, and Tailwind v4 design system."
argument-hint: "Project name, brief description, auth pattern (guest|cognito|groups), and features (contact-form|storage|api-gateway)"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse the user input and extract:
- **Project name**: kebab-case identifier. Used as the directory name and `package.json` `name`. Required.
- **Description**: Short sentence about what the project does. Required.
- **Auth pattern**: `guest`, `cognito`, or `groups`. Required — if not specified, ask before continuing.
- **Features** (optional, comma-separated): `contact-form`, `storage`, `api-gateway`. Default: none.

If the auth pattern is missing, ask once:
> "¿Qué patrón de autenticación necesita el proyecto? Opciones: `guest` (acceso público con API key), `cognito` (login con email/contraseña), `groups` (Cognito con roles Admin y User)."

Do not start implementation until you have the project name, description, and auth pattern.

---

## WHEN TO USE THIS ARCHITECTURE

**Use Amplify + Next.js when:**
- The backend is primarily data-driven (CRUD operations) or requires serverless functions without server management.
- The team is small (1–4 devs) and needs to iterate fast.
- AWS is the cloud provider (or the client is open to it).
- Real-time data via WebSocket subscriptions is a requirement (AppSync provides this out of the box).
- Email notification workflows via SES are required.
- The product requires Cognito authentication with identity pools.

**Do NOT use this architecture when:**
- The project already has an established backend (Spring Boot, Rails, Django, Laravel). In that case, use Next.js with custom API routes or Server Actions calling the existing backend.
- The data model requires complex relational queries with multiple JOINs. DynamoDB + AppSync is optimized for access patterns, not ad-hoc queries. Use RDS + tRPC or REST instead.
- The team is large (10+ devs) with dedicated infrastructure ownership. Use Terraform or standalone CDK with a separate backend team.
- The project requires long-running processes (more than 15 minutes). Lambda has hard execution limits; use ECS or EC2 for those workloads.

---

## INITIALIZATION

Run these commands **in order** from the parent directory:

```bash
# 1. Create Next.js app
npx create-next-app@latest {project-name} \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --no-turbopack

cd {project-name}

# 2. Add Amplify backend (follow https://docs.amplify.aws/nextjs/start/manual-installation/)
npm create amplify@latest
# When prompted: confirm adding Amplify to current directory

# 3. Install project dependencies
npm install next-intl zod react-hook-form @hookform/resolvers framer-motion

# 4. Install dev dependencies
npm install -D jest @types/jest ts-jest jest-environment-node

# 5. Remove default Next.js page content (keep the files, clear the body)
```

After running the above, the project will have:
- `src/` — Next.js app source
- `amplify/` — Amplify Gen 2 backend definition
- `amplify_outputs.json` — Generated after first sandbox run (add to `.gitignore`)

Add to `.gitignore`:
```
amplify_outputs.json
.amplify/
```

---

## PROJECT STRUCTURE (CANONICAL)

This is the authoritative structure. Follow it exactly. Do not add top-level directories not listed here.

```
{project-name}/
├── amplify/                          # AWS Amplify Gen 2 backend (infrastructure)
│   ├── backend.ts                    # Composition root — only defineBackend() here
│   ├── auth/
│   │   └── resource.ts               # Cognito user pool + identity pool config
│   ├── data/
│   │   └── resource.ts               # AppSync GraphQL schema (models + custom mutations)
│   ├── functions/
│   │   └── {domain}/                 # One directory per Lambda function domain
│   │       ├── resource.ts           # defineFunction() — name, runtime, env, IAM grants
│   │       └── handler.ts            # Lambda handler — business logic only
│   ├── cdk/                          # CDK constructs for resources Amplify does NOT manage
│   │   └── {resource}/               # e.g., ses/, api-gateway/
│   │       └── resource.ts           # CDK construct class
│   ├── package.json                  # { "type": "module" } — required for Amplify functions
│   └── tsconfig.json
│
├── src/
│   ├── app/
│   │   └── [locale]/                 # next-intl dynamic locale segment (always present)
│   │       ├── layout.tsx            # Root layout: providers, nav, footer
│   │       ├── page.tsx              # Home page (Server Component)
│   │       ├── not-found.tsx         # Locale-aware 404
│   │       └── {route}/
│   │           └── page.tsx          # Additional pages
│   │
│   ├── actions/                      # Next.js Server Actions — the ONLY place Amplify client is called
│   │   ├── {domain}.ts               # One file per domain (contact.ts, orders.ts, etc.)
│   │   └── __tests__/
│   │       └── {domain}.test.ts      # Unit tests for Server Actions
│   │
│   ├── components/
│   │   ├── atoms/                    # Base elements: Button, Tag, Badge, Input, Label
│   │   ├── molecules/                # Composite pieces: Form, Card, Timeline entry
│   │   └── organisms/                # Full sections: Nav, Footer, Hero, Grid
│   │
│   ├── data/                         # Static in-memory data (no API call needed)
│   │   └── {domain}.ts               # e.g., projects.ts, services.ts, experience.ts
│   │
│   ├── hooks/                        # Custom React hooks (client-only utilities)
│   │   └── use{Name}.ts
│   │
│   ├── i18n/                         # next-intl configuration
│   │   ├── routing.ts                # defineRouting — locales and defaultLocale
│   │   ├── request.ts                # getRequestConfig — dynamic message import
│   │   └── navigation.ts             # Locale-aware Link, useRouter, usePathname
│   │
│   ├── lib/
│   │   ├── types.ts                  # Domain types + Result<T,E> + AppError union
│   │   └── schemas/
│   │       └── {domain}.ts           # Zod schemas for form validation and Server Action input
│   │
│   └── providers/                    # React context providers (client wrappers for libraries)
│       └── {Name}Provider.tsx
│
├── messages/                         # i18n translation files (always 3 locales)
│   ├── es/                           # Spanish — source language, always complete
│   │   └── common.json               # Split by page/domain when the project grows
│   ├── en/
│   │   └── common.json
│   └── pt/
│       └── common.json
│
├── content/                          # Markdown/MDX content (only if the project has rich content)
│   └── {type}/
│       └── {slug}.{locale}.md
│
├── public/
│   └── images/
│
├── amplify_outputs.json              # Generated — in .gitignore
├── next.config.ts
├── tsconfig.json
├── jest.config.ts
├── package.json
└── amplify.yml                       # Amplify CI/CD pipeline config
```

**Rules:**
- `amplify/backend.ts` is the only composition root. Never import Amplify resources from each other.
- `src/actions/` is the only place where the Amplify Data client is instantiated. Never call Amplify from a component directly.
- `src/data/` is for static data only. If data comes from AppSync/DynamoDB, it belongs in a Server Action.
- `src/lib/types.ts` owns all domain types. Never define types inline in components.
- Components under `atoms/`, `molecules/`, `organisms/` never contain business logic.

---

## TYPESCRIPT CONFIGURATION

**`tsconfig.json`**:
```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"],
      "$amplify/*": ["./.amplify/generated/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

`"strict": true` is non-negotiable. Never disable it. Never use `any` without an explicit documented reason.

---

## TAILWIND V4 + DESIGN SYSTEM

Invoke `/tailwind-theme` to configure `src/app/globals.css` with the `@theme` token block. Pass color overrides if the project has a specific brand palette.

Rules:
- All Tailwind classes in components must reference `@theme` tokens (`bg-primary-700`, `text-neutral-0`). No raw hex values.
- Tokens only when needed — do not predefine unused scales.

---

## i18n SETUP (ALWAYS MANDATORY)

Invoke `/next-intl-setup src/` to configure `next-intl` with locales `es` (default), `en`, `pt`. This creates `next.config.ts` wrapper, `src/i18n/routing.ts`, `request.ts`, `navigation.ts`, `src/middleware.ts`, the `[locale]` root layout, and the per-locale `messages/` directory structure.

Rules:
- Always import `Link`, `useRouter`, `usePathname` from `@/i18n/navigation` — never from `next/link` or `next/navigation`.
- Message files use per-locale directories: `messages/es/common.json`, `messages/en/common.json`, `messages/pt/common.json`.
- All locale directories must have identical file and key structures.

---

## AMPLIFY BACKEND

### `amplify/backend.ts` — Composition Root

```typescript
import { defineBackend } from '@aws-amplify/backend';
import { auth } from './auth/resource';
import { data } from './data/resource';
// import each function:
// import { contactEmail } from './functions/contact-email/resource';

const backend = defineBackend({ auth, data });

// Grant IAM permissions to Lambda functions here, not in resource.ts:
// backend.contactEmail.resources.lambda.addToRolePolicy(...)
```

This file only wires resources together and grants cross-resource permissions. No business logic.

---

## AUTH PATTERNS

### Choosing the right pattern — security criteria

Before selecting a pattern, answer these questions:

| Question | Guides toward |
|---|---|
| Does any user need to log in? | No → `guest`. Yes → `cognito` or `groups`. |
| Do different users see different data? | Yes → `cognito` (owner-based) or `groups` (group-based). |
| Does the app have a back-office or admin area with **different data access rules**? | Yes → `groups`. |
| Are the roles only used for UI visibility (show/hide menus) with no data access difference? | Stay with `cognito` — use a custom Cognito attribute for the role. Don't add groups for UI-only differences. |
| Does the app handle financial transactions, sensitive PII, or compliance-regulated data? | Require email verification in `cognito`. Consider MFA. Add server-side re-authorization in every Lambda handler. Never trust client-sent role claims. |
| Is the only authenticated operation a contact form or a newsletter signup? | `guest` is sufficient. Don't add Cognito overhead for anonymous mutations. |

**Principle of least privilege:** Grant only the permissions each user role actually needs in the schema `.authorization()` rules. An authenticated user who doesn't need to write should only have `allow.authenticated().to(['read'])`. Never use `allow.authenticated()` without scoping the operations.

---

### Pattern 1: `guest` — Public API Key

**When to use:** The app has no authenticated users. All data is public or the only write operation is a contact form (anyone can submit without a login).

**`amplify/auth/resource.ts`:**
```typescript
import { defineAuth } from '@aws-amplify/backend';

/** Cognito user pool with guest (unauthenticated) identity pool access. */
export const auth = defineAuth({
  loginWith: {
    email: true,
  },
});
```

> This is a minimal configuration. Amplify Auth also supports social providers (Google, Facebook, Apple), MFA (TOTP, SMS), passwordless (magic link, OTP), phone login, custom user attributes, and more. See the full capabilities at https://docs.amplify.aws/nextjs/build-a-backend/auth/.

**`amplify/data/resource.ts`:**
```typescript
import { type ClientSchema, a, defineData } from '@aws-amplify/backend';

const schema = a.schema({
  // Use .authorization(allow => [allow.publicApiKey()]) for public reads
  // Use .authorization(allow => [allow.guest()]) for unauthenticated mutations
  sendContactEmail: a
    .mutation()
    .arguments({
      email: a.string().required(),
      message: a.string().required(),
    })
    .returns(a.string())
    .authorization(allow => [allow.guest(), allow.publicApiKey()])
    .handler(a.handler.function(contactEmail)),
});

export type Schema = ClientSchema<typeof schema>;

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'apiKey',
    apiKeyAuthorizationMode: { expiresInDays: 365 },
  },
});
```

**In the frontend, obtain credentials via identity pool (no login required):**
```typescript
import { generateClient } from 'aws-amplify/data';
import { fetchAuthSession } from 'aws-amplify/auth';
import { createServerRunner } from '@aws-amplify/adapter-nextjs';
import outputs from '@/../amplify_outputs.json';

const { runWithAmplifyServerContext } = createServerRunner({ config: outputs });

// In a Server Action:
const client = generateClient<Schema>({ authMode: 'apiKey' });
```

---

### Pattern 2: `cognito` — Full Authentication

**When to use:** Users log in with email and password. Protected pages/data require a valid session.

**`amplify/auth/resource.ts`:**
```typescript
import { defineAuth } from '@aws-amplify/backend';

/** Cognito user pool with email/password login and email verification. */
export const auth = defineAuth({
  loginWith: {
    email: {
      verificationEmailStyle: 'CODE',
      verificationEmailSubject: 'Verifica tu cuenta',
      verificationEmailBody: (createCode) =>
        `Tu código de verificación es: ${createCode()}`,
    },
  },
  passwordPolicy: {
    minLength: 8,
    requireLowercase: true,
    requireUppercase: true,
    requireNumbers: true,
  },
  userAttributes: {
    email: { required: true, mutable: true },
  },
});
```

> This is a minimal configuration. Amplify Auth supports additional options: MFA (TOTP/SMS), social providers, account recovery settings, and custom user attributes. See https://docs.amplify.aws/nextjs/build-a-backend/auth/.

**`amplify/data/resource.ts` with owner authorization:**
```typescript
const schema = a.schema({
  Order: a
    .model({
      userId: a.string().required(),
      items: a.string().array(),
      status: a.enum(['pending', 'confirmed', 'delivered']),
    })
    .authorization(allow => [
      allow.owner(),                        // owner reads/writes their own records
      allow.authenticated().to(['read']),   // authenticated users can read (optional)
    ]),
});

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'userPool',  // JWT from Cognito
  },
});
```

**Middleware for protected routes** (`src/middleware.ts`):
```typescript
import { fetchAuthSession } from 'aws-amplify/auth/server';
import { runWithAmplifyServerContext } from '@aws-amplify/adapter-nextjs';
import { createNextIntlMiddleware } from 'next-intl/middleware';
import { routing } from './i18n/routing';
import outputs from '../amplify_outputs.json';

// Compose next-intl middleware with auth protection as needed
```

**Server Action with authenticated user context:**
```typescript
'use server';

import { cookies } from 'next/headers';
import { createServerRunner } from '@aws-amplify/adapter-nextjs';
import { fetchAuthSession } from 'aws-amplify/auth/server';
import outputs from '@/../amplify_outputs.json';

const { runWithAmplifyServerContext } = createServerRunner({ config: outputs });

export async function getAuthenticatedData() {
  return runWithAmplifyServerContext({
    nextServerContext: { cookies },
    async operation(contextSpec) {
      const session = await fetchAuthSession(contextSpec);
      if (!session.tokens) throw new Error('Unauthenticated');
      // proceed with authenticated client
    },
  });
}
```

---

### Pattern 3: `groups` — Role-Based Access (Admin + User)

**When to use:** The app has a back-office (admin panel) and a user-facing area with **different data access rules at the schema level**. Do not use groups just to show or hide UI elements — use a Cognito custom attribute for that instead.

**`amplify/auth/resource.ts`:**
```typescript
import { defineAuth } from '@aws-amplify/backend';

/** Cognito with Admin and User groups for role-based access control. */
export const auth = defineAuth({
  loginWith: {
    email: true,
  },
  groups: ['Admin', 'User'],
});
```

> This is a minimal configuration. Groups can have IAM roles attached and trigger-based automation. See https://docs.amplify.aws/nextjs/build-a-backend/auth/.

**`amplify/data/resource.ts` with group authorization:**
```typescript
const schema = a.schema({
  Product: a
    .model({
      name: a.string().required(),
      price: a.float().required(),
      stock: a.integer(),
    })
    .authorization(allow => [
      allow.group('Admin'),                   // Admins have full CRUD
      allow.authenticated().to(['read']),     // Authenticated users can read
    ]),

  Order: a
    .model({
      productId: a.string().required(),
      quantity: a.integer().required(),
    })
    .authorization(allow => [
      allow.owner(),                          // Users manage their own orders
      allow.group('Admin'),                   // Admins manage all orders
    ]),
});
```

**Adding a user to a group** (Post-confirmation Lambda trigger):
```typescript
// amplify/functions/cognito-triggers/add-to-user-group/handler.ts
import { PostConfirmationTriggerHandler } from 'aws-lambda';
import { CognitoIdentityProviderClient, AdminAddUserToGroupCommand } from '@aws-sdk/client-cognito-identity-provider';

const client = new CognitoIdentityProviderClient({});

export const handler: PostConfirmationTriggerHandler = async (event) => {
  await client.send(new AdminAddUserToGroupCommand({
    GroupName: 'User',
    UserPoolId: event.userPoolId,
    Username: event.userName,
  }));
  return event;
};
```

---

## LAMBDA FUNCTION PATTERN

Invoke `/amplify-function <domain-name> "<iam-actions>"` to add a Lambda domain. This creates `amplify/functions/{domain}/resource.ts` and `handler.ts`, and updates `backend.ts` with the IAM grants.

Rules:
- Handler type always comes from `Schema['{operationName}']['functionHandler']` — never from `aws-lambda`.
- IAM permissions are always granted in `backend.ts`, not in `resource.ts`.
- Secrets go in `ampx secret set` and are referenced with `secret('NAME')` in `resource.ts`.

---

## AMPLIFY vs CDK — DECISION CRITERIA

**Amplify first.** Before writing any CDK construct, check whether Amplify already provides the feature natively at https://docs.amplify.aws/nextjs/build-a-backend/. Amplify manages Auth, Data, Storage, Functions, and Hosting. CDK is only for resources that fall outside those categories. Creating duplicate resources (e.g., a CDK S3 bucket alongside Amplify Storage) introduces infrastructure drift and breaks Amplify's permission model.

### Amplify manages (do NOT replicate in CDK):

| Resource | Amplify API |
|---|---|
| Cognito User Pool + Identity Pool | `defineAuth()` |
| AppSync GraphQL API + DynamoDB | `defineData()` |
| Lambda functions | `defineFunction()` |
| S3 buckets for user uploads | `defineStorage()` |
| Amplify Hosting (SSR, static) | Configured in Amplify Console |

**Never create these resources in CDK if you're already using Amplify.** Duplicating them creates drift and conflicts.

### Use CDK for resources Amplify does NOT manage:

| Resource | When |
|---|---|
| SES domain identity + DKIM | When the app sends transactional email from a verified domain |
| API Gateway (REST) | When you need a REST endpoint for third-party webhooks (e.g., Stripe, Twilio) that can't go through AppSync |
| CloudFront custom distribution | When you need geo-restriction, WAF rules, or custom cache behaviors beyond Amplify Hosting defaults |
| SQS / SNS | When Lambda needs to be decoupled via a message queue |
| RDS | When relational data requirements can't be met by DynamoDB |

### CDK construct pattern `amplify/cdk/{resource}/resource.ts`

```typescript
import { Stack } from 'aws-cdk-lib';
import { EmailIdentity, Identity } from 'aws-cdk-lib/aws-ses';

/** CDK construct: SES email identity for transactional email sending. */
export function addSesIdentity(stack: Stack, domain: string) {
  return new EmailIdentity(stack, 'SesEmailIdentity', {
    identity: Identity.domain(domain),
  });
}
```

Invoke in `amplify/backend.ts`:
```typescript
import { addSesIdentity } from './cdk/ses/resource';

const backend = defineBackend({ auth, data });

// Access the underlying CDK stack
const { stack } = backend.createStack('CustomResources');
addSesIdentity(stack, 'yourdomain.com');
```

### SES Pattern (contact-form feature)

When the `contact-form` feature is requested:

1. SES domain identity is configured via CDK (above).
2. Lambda function calls SES SDK directly:

```typescript
// amplify/functions/shared/email/ses.ts
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses';

const ses = new SESClient({ region: process.env.SES_REGION ?? 'us-east-1' });

interface SendEmailInput {
  to: string;
  from: string;
  subject: string;
  body: string;
}

/** Sends a plain-text email via SES. */
export async function sendEmail({ to, from, subject, body }: SendEmailInput) {
  await ses.send(new SendEmailCommand({
    Destination: { ToAddresses: [to] },
    Source: from,
    Message: {
      Subject: { Data: subject, Charset: 'UTF-8' },
      Body: { Text: { Data: body, Charset: 'UTF-8' } },
    },
  }));
}
```

3. IAM permission granted in `backend.ts`:

```typescript
import { PolicyStatement, Effect } from 'aws-cdk-lib/aws-iam';

backend.contactEmail.resources.lambda.addToRolePolicy(
  new PolicyStatement({
    effect: Effect.ALLOW,
    actions: ['ses:SendEmail', 'ses:SendRawEmail'],
    resources: ['*'],
  }),
);
```

4. SES environment variables injected via `resource.ts`:
```typescript
environment: {
  SES_FROM: 'noreply@yourdomain.com',
  SES_TO: 'contact@yourdomain.com',
  SES_REGION: 'us-east-1',
},
```

---

## SERVER ACTIONS PATTERN

Invoke `/amplify-action <domain> <auth-mode>` to generate a Server Action for a domain. This creates `src/actions/{domain}.ts` (with Zod validation + `Result<T,E>` return type), `src/lib/schemas/{domain}.ts`, and the unit test scaffold.

See `references/amplify-clients.md` for the client factory patterns (guest vs authenticated). See `references/error-handling.md` for the `Result<T,E>` and `AppError` type definitions.

Rules:
- `src/actions/` is the **only** place where Amplify client is instantiated.
- Actions always re-validate with Zod and return `Result<T, AppError>`. Never throw for business errors.

---

## COMPONENT ARCHITECTURE

Invoke `/ui-component <ComponentName> <atom|molecule|organism> [variants]` to generate a component. This creates the file at the correct path, applies the Server vs Client decision, and types the props interface.

Rules:
- Atoms: no side effects, no data fetching, no Server Action calls.
- Molecules: may have local state (`useState`, `useForm`). Never call Server Actions directly.
- Organisms: may call Server Actions. Default to Server Component.
- Props always use `interface`, never `type alias`. No `any`.
- Default to Server Components. Add `'use client'` only when required by hooks or event handlers.

---

## ERROR HANDLING PATTERN

See `references/error-handling.md` for the `Result<T,E>` and `AppError` type definitions and usage patterns.

`src/lib/types.ts` is the single source for both types. The `/amplify-action` skill creates this file automatically if it doesn't exist.

---

## ZOD SCHEMAS

**`src/lib/schemas/{domain}.ts`:**

```typescript
import { z } from 'zod';

/** Validation schema for the contact form. Used on client (react-hook-form) and server (Server Action). */
export const contactSchema = z.object({
  email: z.string().email({ message: 'Email inválido' }),
  phone: z.string().min(7).max(20),
  productType: z.enum(['web-app', 'mobile-ios', 'backend-api', 'consulting', 'support', 'other']),
  description: z.string().min(20, { message: 'Mínimo 20 caracteres' }).max(2000),
});

export type ContactInput = z.infer<typeof contactSchema>;
```

Rules:
- Schema and its inferred type live in the same file.
- `z.infer<typeof schema>` is always exported alongside the schema.
- Error messages in Spanish (source language).
- The same schema is used on both client (zodResolver) and server (Server Action re-validation). Never duplicate validation logic.

---

## TESTING

### `jest.config.ts`

```typescript
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  testMatch: ['**/__tests__/**/*.test.ts', '**/__tests__/**/*.test.tsx'],
};

export default config;
```

### What to test

| Layer | Test focus |
|---|---|
| Server Actions (`src/actions/`) | Validation failures, Amplify client call shape, error handling |
| Zod schemas (`src/lib/schemas/`) | Valid input passes, invalid input returns correct error shape |
| Custom hooks (`src/hooks/`) | State transitions, return values |
| Lambda handlers (`amplify/functions/`) | Business logic, AWS SDK mocks |

### What NOT to test

- Component rendering / visual layout (no snapshot tests)
- Amplify library internals
- Third-party integrations (mock them at the boundary)
- `amplify/backend.ts` (infrastructure definition, not business logic)

### Server Action test pattern

```typescript
// src/actions/__tests__/contact.test.ts
import { sendContactRequest } from '../contact';
import { generateClient } from 'aws-amplify/data';

jest.mock('aws-amplify/data');
jest.mock('@/../amplify_outputs.json', () => ({}));

const mockClient = {
  mutations: {
    sendContactEmail: jest.fn(),
  },
};

(generateClient as jest.Mock).mockReturnValue(mockClient);

describe('sendContactRequest', () => {
  it('returns validation error for invalid email', async () => {
    const result = await sendContactRequest({
      email: 'not-an-email',
      phone: '1234567',
      productType: 'web-app',
      description: 'a'.repeat(20),
    });

    expect(result.ok).toBe(false);
    if (!result.ok) expect(result.error.type).toBe('validation');
  });

  it('returns ok on successful mutation', async () => {
    mockClient.mutations.sendContactEmail.mockResolvedValue({ data: 'ok', errors: null });

    const result = await sendContactRequest({
      email: 'valid@example.com',
      phone: '1234567',
      productType: 'web-app',
      description: 'a'.repeat(20),
    });

    expect(result.ok).toBe(true);
  });
});
```

---

## DEPLOYMENT — `amplify.yml`

```yaml
version: 1
backend:
  phases:
    build:
      commands:
        - npm ci
        - npx ampx pipeline-deploy --branch $AWS_BRANCH --app-id $AWS_APP_ID
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npx ampx generate outputs --branch $AWS_BRANCH --app-id $AWS_APP_ID
        - npm run build
  artifacts:
    baseDirectory: .next
    files:
      - "**/*"
  cache:
    paths:
      - node_modules/**/*
      - .next/cache/**/*
```

**Rules:**
- `ampx pipeline-deploy` deploys backend changes before building the frontend.
- `ampx generate outputs` regenerates `amplify_outputs.json` in CI before `next build`.
- Never commit `amplify_outputs.json` — it is generated in CI from the deployed backend.

---

## ENVIRONMENT VARIABLES

### Local development — `.env.local`

Create `.env.local` at the project root (never commit this file):

```bash
# Non-secret config used by Lambda functions in local sandbox
# These mirror the values injected by defineFunction().environment in production
SES_FROM=noreply@yourdomain.com
SES_TO=contact@yourdomain.com
SES_REGION=us-east-1

# Add any other non-secret config your functions or Next.js app needs locally
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Add to `.gitignore`:
```
.env.local
.env*.local
```

`amplify_outputs.json` is generated by the sandbox and is already in `.gitignore`. Do not duplicate its values in `.env.local`.

**Secrets** (API keys, tokens, credentials) must never be in `.env.local` or in `defineFunction().environment`. Register them with `ampx secret set` and reference them in `resource.ts` with `secret('NAME')`.

### Production and staging environments

Environment variables for Lambda functions in production/staging are managed in three levels:

1. **Non-secret config injected by Amplify**: Defined in `defineFunction().environment` in `resource.ts`. Amplify injects them at deploy time. These are visible in the Lambda configuration — do not put secrets here.

2. **Secrets**: Registered with `ampx secret set` and referenced in `resource.ts` with `secret()`. Amplify handles injection into the Lambda environment automatically — no SDK calls needed in the handler.
   ```bash
   npx ampx secret set STRIPE_WEBHOOK_SECRET
   npx ampx secret set PAYMENT_API_KEY
   ```
   ```typescript
   // amplify/functions/{domain}/resource.ts
   import { defineFunction, secret } from '@aws-amplify/backend';

   export const myFunction = defineFunction({
     name: 'my-function',
     environment: {
       PAYMENT_API_KEY: secret('PAYMENT_API_KEY'),
     },
   });
   ```
   Inside the handler, access via `process.env.PAYMENT_API_KEY` — Amplify decrypts at runtime.

3. **Amplify Console environment variables**: For Next.js `NEXT_PUBLIC_*` variables and build-time config, set them in the Amplify Console under the app's **Environment variables** section. They are injected during `next build` in the CI pipeline. See: https://docs.amplify.aws/nextjs/deploy-and-host/environment-variables/.

---

## EXECUTION CHECKLIST

After scaffolding the project, verify:

- [ ] `tsconfig.json` has `"strict": true` and `$amplify/*` path alias
- [ ] `amplify/backend.ts` only contains `defineBackend()` and IAM grants — no business logic
- [ ] Every Lambda function has `resource.ts` + `handler.ts`
- [ ] All Lambda IAM permissions granted in `backend.ts`, not in `resource.ts`
- [ ] `src/actions/` is the only place `generateClient` is called
- [ ] All Server Actions re-validate with Zod and return `Result<T, AppError>`
- [ ] `src/i18n/routing.ts` defines `locales: ['es', 'en', 'pt']` and `defaultLocale: 'es'`
- [ ] `messages/` uses per-locale directories (`es/`, `en/`, `pt/`) with `common.json` in each
- [ ] All locale directories have identical file and key structures
- [ ] Components import `Link`, `useRouter`, `usePathname` from `@/i18n/navigation`
- [ ] `'use client'` is only present when required (event handlers, hooks, third-party wrappers)
- [ ] All component props are typed with explicit `interface`, no `any`
- [ ] No business logic inside component `return` statements
- [ ] Tailwind classes reference design tokens from `@theme`, not raw hex values
- [ ] `amplify_outputs.json` and `.env*.local` are in `.gitignore`
- [ ] Secrets registered with `ampx secret set` and referenced via `secret('NAME')` in `resource.ts`
- [ ] No secrets in `defineFunction().environment` as plain strings
- [ ] `NEXT_PUBLIC_*` variables set in Amplify Console, not committed to the repo
- [ ] Lambda handler types imported from `../../data/resource` (not from `aws-lambda`)
- [ ] `jest.config.ts` is present and tests run with `npm test`
- [ ] `amplify.yml` has backend deploy before frontend build
