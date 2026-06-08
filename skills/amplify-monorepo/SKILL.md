---
name: amplify-monorepo
description: Scaffold projects with Amplify Gen 2 + Nx monorepo architecture — two Next.js apps (public web + admin), shared design system, transactional email with Maizzle, next-intl i18n, and SaaS webhook integrations.
disable-model-invocation: false
---

# Amplify Monorepo Architecture Skill

## When to use this architecture

**Use it when the project requires:**

- Two distinct frontends: a public-facing app (end users) and a separate administration app
- Three or more user roles with clearly different permissions (e.g. ADMIN, OPERATOR, USER) — if there is only on or two basic roles, splitting into two apps is not justified
- Serverless backend on AWS with business logic in Lambda
- Authentication with Cognito (including email OTP, user groups)
- Transactional email with rich HTML templates (confirmations, receipts, notifications)
- Integration with external SaaS services that require webhooks (payment gateways, SMS platforms, CRMs, etc.)
- Multi-language support in the frontend (next-intl)
- Shared design system between both apps

**Do not use it when:**

- There is only one frontend — a single Next.js app is enough
- All users share the same role or permissions are trivial — the overhead of Cognito groups and group-based authorization adds no value
- The project is a content site (CMS, blog, landing page) with no transactional operations — a Next.js app with a headless CMS is more appropriate
- The backend has complex logic that does not fit serverless — consider a dedicated server (Nest.js, Go)
- There is no transactional email, external integrations, or sensitive data — the CDK complexity is not justified
- The team has no experience with AWS/Amplify — the learning curve is steep

## Security criteria for adopting this architecture

This architecture enforces security controls that only make sense for transactional projects or projects with sensitive data. Evaluate each criterion before adopting it:

| Criterion                                                  | Justifies this architecture                                         | Simpler alternative                                       |
| ---------------------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------- |
| Handles PII (documents, phone numbers, identity)           | Yes — Cognito + group-based authorization                           | Does not apply if data is public                          |
| Payment or subscription operations                         | Yes — webhook with authorizer, transaction audit log                | Does not apply for informational sites                    |
| Roles with radically different permissions (admin vs user) | Yes — separate admin app, Cognito groups, per-function IAM policies | Does not apply if all users see the same content          |
| Third-party webhooks that cannot authenticate with Cognito | Yes — API Gateway + Lambda authorizer with HMAC token               | Does not apply if there are no external integrations      |
| Audit trail for critical actions                           | Yes — `AuditLog` model with actor + resource + action               | Does not apply for low-risk operations                    |
| Attack surface separation                                  | Yes — admin on a separate domain, not publicly exposed              | Does not apply if the admin panel is internal or low-risk |

**Warning:** If none of the left-column criteria apply to the project, this architecture adds complexity with no real security benefit.

---

## Tech stack

| Layer             | Technology                                                                      |
| ----------------- | ------------------------------------------------------------------------------- |
| Monorepo          | Nx 22+                                                                          |
| Frontend          | Next.js 15+ (App Router), TypeScript strict, TailwindCSS                        |
| Auth              | AWS Cognito (via Amplify Gen 2)                                                 |
| Backend           | AWS Amplify Gen 2 (`@aws-amplify/backend`)                                      |
| Database          | DynamoDB via AppSync GraphQL                                                    |
| Lambda            | Node.js 20+, TypeScript                                                         |
| Email             | Amazon SES + Maizzle                                                            |
| SaaS integrations | Webhooks via API Gateway + Lambda authorizer (payment gateways, SMS, CRM, etc.) |
| i18n              | next-intl                                                                       |
| UI                | CVA + TailwindCSS (custom design system)                                        |
| Tests             | Jest + Testing Library, Playwright                                              |
| CI/CD             | Amplify Hosting + amplify.yml                                                   |

---

## Monorepo structure

```
<project-name>/
├── apps/
│   ├── admin/                          # Administration app (Next.js)
│   │   ├── amplify/                    # AWS Amplify Gen 2 backend ← backend lives here
│   │   │   ├── backend.ts              # Single composition root
│   │   │   ├── auth/
│   │   │   │   └── resource.ts         # Cognito: login, groups, attributes, triggers
│   │   │   ├── data/
│   │   │   │   └── resource.ts         # GraphQL schema: models, queries, mutations
│   │   │   ├── functions/
│   │   │   │   ├── cognito-triggers/   # pre-signup, post-confirmation, custom-message
│   │   │   │   ├── notifications/      # One function per email type
│   │   │   │   ├── [integration]/      # Webhook handler + authorizer per SaaS integration
│   │   │   │   ├── services/           # Multi-step business logic
│   │   │   │   ├── users/              # Cognito UserPool operations (list, set, toggle)
│   │   │   │   └── shared/             # Shared utilities across functions
│   │   │   │       ├── common/         # Types, constants, audit, identity, security
│   │   │   │       ├── email/          # Template renderer, SES send with retry
│   │   │   │       └── [integration]/  # SaaS SDK client, types, utils (e.g. payment/, sms/)
│   │   │   └── cdk/                    # CDK constructs for resources outside Amplify scope
│   │   │       ├── ses/                # SES domain identity + IAM policy
│   │   │       └── functions/          # Lambda Layers (email templates, etc.)
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── login/
│   │   │   │   └── dashboard/
│   │   │   │       ├── layout.tsx
│   │   │   │       ├── page.tsx
│   │   │   │       └── [resource]/     # One folder per entity (users, plans, sessions…)
│   │   │   ├── actions/                # Next.js Server Actions: mutations and queries
│   │   │   ├── components/
│   │   │   │   ├── auth/
│   │   │   │   ├── layout/
│   │   │   │   └── [resource]/         # Domain components: table, modals, types
│   │   │   ├── i18n/
│   │   │   │   ├── routing.ts          # Locale definitions
│   │   │   │   ├── request.ts
│   │   │   │   └── load-messages.ts
│   │   │   └── lib/
│   │   │       └── formatters.ts       # Date/currency formatters with timezone
│   │   └── messages/                   # Translations by locale: es/, en/, pt/
│   │
│   ├── web/                            # Public app (Next.js)
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   └── (public)/           # Public route group
│   │   │   ├── actions/                # Server Actions: backend operations
│   │   │   ├── components/
│   │   │   │   ├── amplify/            # ConfigureAmplifyClientSide.tsx
│   │   │   │   ├── auth/
│   │   │   │   ├── layout/
│   │   │   │   ├── molecules/
│   │   │   │   └── organisms/
│   │   │   ├── i18n/
│   │   │   └── lib/
│   │   │       └── client.ts           # getGuestClient / getAuthenticatedClient
│   │   └── messages/
│   │
│   └── web-e2e/                        # E2E tests with Playwright
│
├── packages/
│   ├── ui/                             # Shared design system (@<project>/ui)
│   │   └── src/
│   │       ├── atoms/                  # Button, Input, Badge, Heading, Text…
│   │       ├── molecules/              # Form, Card, Modal, Dropdown, LanguageSelector…
│   │       ├── organisms/              # DataTable, Sidebar, Banner…
│   │       ├── icons/
│   │       └── styles/
│   │           └── globals.css
│   │
│   ├── core/                           # Shared utilities (@<project>/core)
│   │   └── src/
│   │       └── auth/
│   │           ├── client.ts           # createServerClient factory
│   │           ├── server.ts           # Server-side auth utils
│   │           ├── middleware.ts       # Next.js middleware
│   │           └── services/
│   │               └── user.ts         # Fetch user attributes from Cognito
│   │
│   └── email-templates/                # Maizzle templates (@<project>/email-templates)
│       ├── emails/                     # HTML templates (one per email type)
│       ├── components/                 # Reusable email components
│       ├── layouts/
│       │   └── base.html
│       ├── partials/
│       │   ├── headers/
│       │   └── footers/
│       ├── static/                     # Images, logos
│       ├── config.js                   # Maizzle dev config
│       └── config.production.js        # Maizzle production config
│
├── config/
│   └── amplify_outputs.json            # Generated by Amplify CLI — consumed by both apps
│
├── amplify.yml                         # Amplify Hosting CI/CD
├── nx.json
├── tsconfig.base.json
└── package.json
```

### Package dependency rules

```
apps/* → packages/*                            ✓  apps import from packages
packages/* → packages/*                        ✗  packages are independent of each other
packages/* → apps/*                            ✗  never
amplify/functions/* → amplify/functions/shared/ ✓
amplify/functions/shared/ → node_modules only  ✓
```

---

## Step 1 — Scaffolding with Nx

### 1.1 Create the workspace

```bash
npx create-nx-workspace@latest <project-name> \
  --preset=empty \
  --packageManager=npm \
  --workspaceType=integrated
cd <project-name>
```

### 1.2 Install required Nx plugins

```bash
npm install -D @nx/next @nx/react @nx/eslint @nx/jest @nx/playwright @nx/js
```

### 1.3 Create the admin app (backend owner)

```bash
npx nx generate @nx/next:application admin \
  --directory=apps/admin \
  --style=tailwind \
  --linter=eslint \
  --e2eTestRunner=none \
  --unitTestRunner=jest \
  --appDir=true \
  --srcDir=true
```

### 1.4 Create the public web app

```bash
npx nx generate @nx/next:application web \
  --directory=apps/web \
  --style=tailwind \
  --linter=eslint \
  --e2eTestRunner=none \
  --unitTestRunner=jest \
  --appDir=true \
  --srcDir=true
```

### 1.5 Create the E2E app

```bash
npx nx generate @nx/playwright:configuration web-e2e \
  --project=web \
  --directory=apps/web-e2e
```

### 1.6 Create the UI package (design system)

```bash
npx nx generate @nx/react:library ui \
  --directory=packages/ui \
  --style=tailwind \
  --linter=eslint \
  --unitTestRunner=jest \
  --bundler=none \
  --component=false
```

### 1.7 Create the Core package (shared auth)

```bash
npx nx generate @nx/js:library core \
  --directory=packages/core \
  --linter=eslint \
  --unitTestRunner=jest \
  --bundler=none
```

### 1.8 Create the email-templates package (manual)

`packages/email-templates/` has no native Nx generator. Create it manually:

```bash
mkdir -p packages/email-templates/{emails,components,layouts,partials/{headers,footers},static}
```

Install Maizzle:

```bash
npm install -D @maizzle/framework tailwindcss-preset-email juice
```

### 1.9 Configure path aliases in tsconfig.base.json

```json
{
  "compilerOptions": {
    "paths": {
      "$amplify/*": ["apps/admin/.amplify/generated/*"],
      "@amplify/*": ["apps/admin/amplify/*"],
      "@config/*": ["config/*"],
      "@<project>/core/*": ["packages/core/src/*"],
      "@<project>/ui/*": ["packages/ui/src/*"]
    }
  }
}
```

---

## Step 2 — AWS Amplify Gen 2 backend

### 2.1 Install Amplify

```bash
npm install aws-amplify @aws-amplify/backend @aws-amplify/backend-cli
npm install -D aws-cdk-lib constructs
```

### 2.2 Initialize the backend in apps/admin/

```bash
cd apps/admin
npx ampx configure project
```

This generates `apps/admin/amplify/` with the base structure.

### 2.3 Auth — apps/admin/amplify/auth/resource.ts

Rules:

- Cognito is the single source of truth for users. Do not create a `User` model in DynamoDB.
- Lambda function permissions on the Cognito UserPool are declared in `access()`, not in the handlers.
- Cognito triggers are registered in `triggers`, not as standalone functions.

```typescript
import { defineAuth } from '@aws-amplify/backend';

export const auth = defineAuth({
  loginWith: {
    email: {
      otpLogin: true, // Email OTP instead of password
    },
  },
  senders: {
    email: {
      fromEmail: 'auth@<domain>',
      fromName: process.env.APP_NAME || '<AppName>',
    },
  },
  triggers: {
    customMessage: customMessageTrigger, // Customize Cognito emails
    preSignUp: preSignUpTrigger, // Validate document uniqueness
    postConfirmation: postConfirmationTrigger, // Assign initial group
  },
  groups: ['ADMIN', 'OPERATOR', 'USER'], // Adjust to domain roles
  userAttributes: {
    givenName: { mutable: true },
    familyName: { mutable: true },
    email: { required: true, mutable: true },
    phoneNumber: { mutable: true },
    // Domain-specific custom attributes
    'custom:document_type': { dataType: 'String', mutable: false },
    'custom:document_number': { dataType: 'String', mutable: false },
    'custom:subscription_level': { dataType: 'String', mutable: true },
  },
  accountRecovery: 'EMAIL_ONLY',
  access: (allow) => [
    // Declare each Lambda's permissions on the UserPool
    allow.resource(postConfirmationTrigger).to(['addUserToGroup']),
    allow.resource(preSignUpTrigger).to(['listUsers']),
    allow.resource(listUsersFunction).to(['listUsersInGroup']),
    allow.resource(setUserFunction).to(['getUser', 'createUser', 'addUserToGroup', 'updateUserAttributes']),
    allow.resource(toggleUserFunction).to(['disableUser', 'enableUser']),
  ],
});
```

### 2.4 Data schema — apps/admin/amplify/data/resource.ts

Rules:

- Authorization is declared in the schema, never in Lambda handlers.
- Cognito operations are exposed as custom `query` or `mutation` with `a.handler.function()`.
- Email notifications are exposed as async mutations (`.async()`).
- `defaultAuthorizationMode: 'apiKey'` enables public read without login (for catalogs, pricing).
- Resources visible only to authenticated users use `allow.authenticated()`.
- Single-owner resources use `allow.owner()` or `allow.ownerDefinedIn('fieldName')`.

```typescript
import { a, ClientSchema, defineData } from '@aws-amplify/backend';

const schema = a
  .schema({
    // ── Domain enums ──────────────────────────────────────────────
    MyEnum: a.enum(['VALUE_A', 'VALUE_B']),

    // ── Custom types for Lambda-backed mutations/queries ──────────
    MyCustomInput: a.customType({
      field: a.string().required(),
    }),

    // ── Data models ───────────────────────────────────────────────
    MyModel: a
      .model({
        name: a.string().required(),
        ownerSub: a.string().required(),
      })
      .authorization((allow) => [allow.ownerDefinedIn('ownerSub').to(['read']), allow.groups(['ADMIN']).to(['read', 'create', 'update', 'delete'])]),

    // ── Async mutations for email notifications ───────────────────
    sendNotificationEmail: a
      .mutation()
      .arguments({ to: a.string().required(), data: a.ref('MyCustomInput').required() })
      .handler(a.handler.function(myNotificationFunction).async())
      .authorization((allow) => [allow.authenticated()]),

    // ── Lambda-backed queries/mutations for Cognito operations ────
    listUsers: a
      .query()
      .returns(a.ref('UserItem').array())
      .handler(a.handler.function(listUsersFunction))
      .authorization((allow) => [allow.groups(['ADMIN'])]),
  })
  // Allow Lambda functions to access the schema without user authentication
  .authorization((allow) => [allow.resource(webhookFunction), allow.resource(setUserFunction)]);

export type Schema = ClientSchema<typeof schema>;

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'apiKey', // Public read (catalogs)
    apiKeyAuthorizationMode: { expiresInDays: 30 },
  },
});
```

### 2.5 Lambda functions — domain-based structure

Each domain has its own directory with `resource.ts` + handlers:

```
functions/
├── [domain]/
│   ├── resource.ts          # defineFunction(): name, entry, env, timeout, memoryMB
│   └── handler.ts           # Lambda handler (or separate files per operation)
├── cognito-triggers/        # pre-signup, post-confirmation, custom-message
├── notifications/           # One function per email type
├── [integration]/           # One folder per SaaS webhook integration (e.g. payments/, sms/, crm/)
│   ├── resource.ts          # Defines the main handler + Lambda authorizer
│   ├── handler.ts           # Processes the webhook event
│   └── authorizer.ts        # Verifies the provider signature/token (HMAC, Bearer, etc.)
├── services/                # Multi-step logic (e.g. orchestrating multiple operations)
├── users/                   # Cognito UserPool operations
└── shared/
    ├── common/              # Types, constants, audit, identity, security
    ├── email/               # Maizzle template renderer, SES send with retry
    └── [integration]/       # SaaS SDK client, types and utilities (e.g. payment/, sms/)
```

**resource.ts example for modular notifications:**

```typescript
import { defineFunction } from '@aws-amplify/backend';

const sharedEmailEnv = {
  NOREPLY_EMAIL: process.env.NOREPLY_EMAIL || 'noreply@<domain>',
  SUPPORT_EMAIL: process.env.SUPPORT_EMAIL || 'support@<domain>',
  APP_NAME: process.env.APP_NAME || '<AppName>',
  ADMIN_EMAIL: process.env.ADMIN_EMAIL || 'admin@<domain>',
};

export const sendWelcomeFunction = defineFunction({
  name: 'sendWelcome',
  entry: './sendWelcome.ts',
  environment: sharedEmailEnv,
  timeoutSeconds: 30,
  memoryMB: 512, // Higher because Maizzle renders at runtime
});

export const sendReceiptFunction = defineFunction({
  name: 'sendReceipt',
  entry: './sendReceipt.ts',
  environment: sharedEmailEnv,
  timeoutSeconds: 30,
  memoryMB: 512,
});
```

**Rules for functions:**

- One `resource.ts` per domain directory — defines all functions for that domain.
- Logic shared between functions goes in `functions/shared/`, never duplicated.
- Environment variables: injected in `resource.ts`, never hardcoded in the handler.
- Secrets (API keys, passwords): use `ampx secret set <NAME>` and reference with `secret('NAME')`.

### 2.6 backend.ts — composition root

```typescript
import { defineBackend } from '@aws-amplify/backend';
import * as lambda from 'aws-cdk-lib/aws-lambda';

import { auth } from './auth/resource';
import { data } from './data/resource';
import { sendWelcomeFunction, sendReceiptFunction } from './functions/notifications/resource';
import { LambdaLayerEmailTemplates } from './cdk/functions/email-templates-layer';
import { SESEmailIdentity } from './cdk/ses/email-identity';

export const backend = defineBackend({
  auth,
  data,
  sendWelcomeFunction,
  sendReceiptFunction,
  // ...all other functions
});

// CDK customization — only for resources not covered by Amplify (see Step 3)
const backendStack = backend.createStack('BackendStack');

const emailTemplatesLayer = new LambdaLayerEmailTemplates(backendStack, 'EmailTemplatesLayer');
const sesConfig = new SESEmailIdentity(backendStack, 'SESConfig', { domain: '<domain>' });

const notificationFunctions = [backend.sendWelcomeFunction, backend.sendReceiptFunction];

notificationFunctions.forEach((func) => {
  const lambdaFn = func.resources.lambda as lambda.Function;
  lambdaFn.addLayers(emailTemplatesLayer);
  lambdaFn.role?.addManagedPolicy(sesConfig.sendPolicy);
});
```

**Rules for backend.ts:**

- Single composition root — functions do not import from each other directly.
- No business logic here — only instantiate CDK resources and wire them together.

---

## Step 3 — CDK: Amplify-first decision process

**Before writing any CDK construct, check the official Amplify Gen 2 documentation:**

> <https://docs.amplify.aws/nextjs/build-a-backend/>

If the infrastructure resource is listed there, use the Amplify abstraction. Only fall back to raw CDK when Amplify does not cover the resource.

### Decision flowchart

```
Need an AWS resource?
        │
        ▼
Is it listed at docs.amplify.aws/nextjs/build-a-backend/?
        │
   YES ─┤─ NO
        │     └─► Use CDK via backend.createStack()
        ▼
Use the Amplify abstraction:
  - Auth      → defineAuth()
  - Data      → defineData() / a.schema()
  - Functions → defineFunction()
  - Hosting   → amplify.yml
  - Storage   → defineStorage()
```

### What Amplify Gen 2 covers (no CDK needed)

- Cognito UserPool, UserPoolClient, Identity Pool
- AppSync API + DynamoDB (via `defineData`)
- Lambda functions (via `defineFunction`)
- S3 storage (via `defineStorage`)
- Basic IAM policies between Amplify resources (via `access()` in auth and `.authorization()` in data)
- Amplify Hosting (via `amplify.yml`)

### What requires CDK directly

| Resource                         | Why Amplify does not cover it                                      |
| -------------------------------- | ------------------------------------------------------------------ |
| SES domain identity              | Amplify does not manage SES                                        |
| Lambda Layers                    | No `defineLayer` exists in Amplify Gen 2                           |
| API Gateway (external webhooks)  | Amplify does not expose public HTTP endpoints without Cognito auth |
| Lambda authorizer for webhooks   | Depends on custom API Gateway                                      |
| KMS key for custom SMS sender    | Non-standard Cognito configuration                                 |
| Any AWS service not listed above | Use `backend.createStack()` + CDK                                  |

### CDK construct — SES

```typescript
// apps/admin/amplify/cdk/ses/email-identity.ts
import * as iam from 'aws-cdk-lib/aws-iam';
import * as ses from 'aws-cdk-lib/aws-ses';
import { Construct } from 'constructs';

export class SESEmailIdentity extends Construct {
  public readonly sendPolicy: iam.ManagedPolicy;

  constructor(scope: Construct, id: string, props: { domain: string }) {
    super(scope, id);

    ses.EmailIdentity.fromEmailIdentityName(this, 'DomainIdentity', props.domain);

    this.sendPolicy = new iam.ManagedPolicy(this, 'SESPolicy', {
      statements: [
        new iam.PolicyStatement({
          actions: ['ses:SendEmail', 'ses:SendRawEmail'],
          resources: ['*'],
          conditions: { StringLike: { 'ses:FromAddress': `*@${props.domain}` } },
        }),
      ],
    });
  }
}
```

**Required manual setup:**

1. Verify the domain in AWS SES Console (sandbox → production).
2. Configure DNS records: DKIM, SPF, DMARC.
3. Request sandbox exit if sending to unverified emails.

### CDK construct — Lambda Layer for email templates

```typescript
// apps/admin/amplify/cdk/functions/email-templates-layer.ts
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Stack } from 'aws-cdk-lib';

export class LambdaLayerEmailTemplates extends lambda.LayerVersion {
  constructor(scope: Stack, id: string) {
    super(scope, id, {
      code: lambda.Code.fromAsset('../../packages/email-templates', {
        bundling: {
          local: {
            tryBundle(outputDir) {
              // Copy emails/, components/, layouts/, partials/, static/, config.js
              // to outputDir/opt/email-templates/
              // Return true on success, false to fall back to Docker
            },
          },
          image: lambda.Runtime.NODEJS_20_X.bundlingImage,
          command: ['bash', '-c', 'cp -r emails components layouts partials static config.js config.production.js /asset-output/opt/email-templates/'],
        },
      }),
      compatibleRuntimes: [lambda.Runtime.NODEJS_20_X, lambda.Runtime.NODEJS_22_X],
      description: 'Maizzle email templates',
    });
  }
}
```

### CDK — SaaS webhook integrations (API Gateway)

When an external SaaS (payment gateway, SMS provider, CRM) needs to push events to the app via HTTP and cannot authenticate with Cognito. The pattern is always the same: API Gateway + Lambda handler + Lambda authorizer with provider signature verification.

```typescript
// In backend.ts, inside createStack():
import * as apigateway from 'aws-cdk-lib/aws-apigateway';

// One RestApi can have multiple resources, one per integration
const webhooksApi = new apigateway.RestApi(backendStack, 'WebhooksApi', {
  restApiName: 'webhooks-api',
});

const webhooksRoot = webhooksApi.root.addResource('webhooks');

// Example: payment gateway integration
const paymentsResource = webhooksRoot.addResource('payments');

const paymentsAuthorizer = new apigateway.RequestAuthorizer(backendStack, 'PaymentsAuthorizer', {
  handler: backend.paymentsAuthorizerFunction.resources.lambda,
  // The header carrying the token/signature depends on the provider
  identitySources: [apigateway.IdentitySource.header('x-webhook-token')],
});

paymentsResource.addMethod('POST', new apigateway.LambdaIntegration(backend.paymentsFunction.resources.lambda), {
  authorizationType: apigateway.AuthorizationType.CUSTOM,
  authorizer: paymentsAuthorizer,
});

// Add more integrations following the same pattern
// const smsResource = webhooksRoot.addResource('sms');

new cdk.CfnOutput(backendStack, 'PaymentsWebhookUrl', {
  value: `${webhooksApi.url}webhooks/payments`,
});
```

**Lambda authorizer — generic pattern:**

The authorizer verifies that the event comes from the legitimate provider. The mechanism depends on the SaaS:

- **HMAC-SHA256** (Stripe, GitHub, most payment gateways): compare header signature with payload hash
- **Secret Bearer token** (Twilio, SendGrid): compare header with stored secret
- **IP allowlist**: validate origin IP from API Gateway headers

```typescript
// functions/[integration]/authorizer.ts
import { verifyHmacSignature } from '../shared/common/security';

export const handler = async (event: APIGatewayAuthorizerEvent) => {
  const signature = event.headers['x-signature'] ?? '';
  const secret = process.env.WEBHOOK_SECRET!;
  const isValid = verifyHmacSignature(event.body ?? '', signature, secret);

  return {
    principalId: 'webhook',
    policyDocument: {
      Statement: [
        {
          Action: 'execute-api:Invoke',
          Effect: isValid ? 'Allow' : 'Deny',
          Resource: event.methodArn,
        },
      ],
    },
  };
};
```

---

## Step 4 — Consuming the backend from the apps

### 4.1 packages/core — createServerClient

```typescript
// packages/core/src/auth/client.ts
import { generateServerClientUsingCookies } from '@aws-amplify/adapter-nextjs/api';

export const createServerClient =
  <Schema extends Record<string | number | symbol, unknown> = never>(outputs: NextServer.CreateServerRunnerInput['config'], cookies: () => Promise<ReadonlyRequestCookies>) =>
  (config?: { authMode?: GraphQLAuthMode; authToken?: string }) =>
    generateServerClientUsingCookies<Schema>({ ...config, config: outputs, cookies });
```

### 4.2 Client factories per app

Each app has its own `lib/client.ts` marked with `'use server'`:

```typescript
// apps/web/src/lib/client.ts  (same pattern in apps/admin/src/lib/client.ts)
'use server';

import { cookies } from 'next/headers';
import { createServerClient } from '@<project>/core';
import config from '@config/amplify_outputs.json';
import { Schema } from '@amplify/data/resource';

export async function getGuestClient() {
  return createServerClient<Schema>(config, cookies)();
}

export async function getAuthenticatedClient() {
  return createServerClient<Schema>(config, cookies)({ authMode: 'userPool' });
}
```

### 4.3 Server Actions — apps/\*/src/actions/

All backend operations (queries and mutations) live in `actions/` as Next.js Server Actions.
No network calls in components or in `useEffect`.

**Use `actions/` in both the web and admin apps:**

```typescript
// apps/web/src/actions/payment.ts
'use server';

import { getAuthenticatedClient } from '../lib/client';

/** Finalizes the payment and registers the transaction. */
export async function finalizePayment(input: FinalizePaymentInput): Promise<Result<Transaction, AppError>> {
  const client = await getAuthenticatedClient();
  // 1. Fetch payment plan
  // 2. Create UserPlan (PENDING)
  // 3. Create Transaction
  // 4. Fire async notification mutation
  // 5. Return Result<T, E>
}
```

**Error pattern — Result<T, E>:**

```typescript
type Result<T, E = AppError> = { ok: true; value: T } | { ok: false; error: E };

type AppError = { type: 'validation'; fields: Record<string, string> } | { type: 'not_found'; resource: string } | { type: 'unauthorized' } | { type: 'unexpected'; cause: unknown };
```

**Rule:** AppSync mutations are called from Server Actions, never from Client Components directly. Client Components call Server Actions via form `action` props or `startTransition`.

---

## Step 5 — Transactional email with Maizzle

### 5.1 packages/email-templates structure

```
emails/
├── welcome.html              # One file per email type
├── otp-code.html
├── payment-receipt.html
├── payment-failed.html
└── admin-notification.html
```

### 5.2 Base layout

```html
<!-- layouts/base.html -->
<x-root>
  <x-head>
    <title>{{ page.title }}</title>
  </x-head>
  <body>
    <x-partial src="partials/headers/brand.html"></x-partial>
    <content></content>
    <x-partial src="partials/footers/brand.html"></x-partial>
  </body>
</x-root>
```

### 5.3 Email template with variables

```html
<!-- emails/welcome.html -->
--- layout: layouts/base.html title: "Welcome" ---
<p>Hello {{ page.userName }},</p>
<p>Your session is on {{ page.sessionDate }} at {{ page.sessionTime }}</p>
```

### 5.4 Renderer in Lambda (shared/email/template-renderer.ts)

```typescript
import { Config, render } from '@maizzle/framework';
import emailPreset from 'tailwindcss-preset-email';

const LAMBDA_TEMPLATES_PATH = '/opt/email-templates';

export async function renderEmailTemplate<T extends Record<string, any>>(templateName: string, data: T): Promise<string> {
  const template = fs.readFileSync(path.join(LAMBDA_TEMPLATES_PATH, 'emails', `${templateName}.html`), 'utf8');
  const config = require(path.join(LAMBDA_TEMPLATES_PATH, 'config.production.js'));

  const { html } = await render(template, {
    ...config,
    components: { ...config.components, root: LAMBDA_TEMPLATES_PATH },
    locals: { currentYear: new Date().getFullYear(), ...data },
    css: {
      ...config.css,
      tailwind: {
        content: (config.css?.tailwind?.content ?? []).map((p: string) => path.join(LAMBDA_TEMPLATES_PATH, p)),
        presets: [emailPreset],
      },
    },
  });

  return html;
}
```

### 5.5 i18n in emails — pattern to implement

Email templates are currently single-language. Two patterns for multi-language support:

**Option A — Translation variables in `locals`** (recommended for few strings):

```typescript
// In the Lambda notification function:
const translations = loadTranslations(locale); // Load from JSON in the Layer
const html = await renderEmailTemplate('welcome', {
  ...data,
  locale,
  t: translations,
});
```

```html
<!-- emails/welcome.html -->
<p>{{ page.t.greeting }}, {{ page.userName }}</p>
```

**Option B — Separate templates per locale** (recommended when templates vary significantly):

```
emails/
├── welcome.es.html
├── welcome.en.html
└── welcome.pt.html
```

```typescript
const html = await renderEmailTemplate(`welcome.${locale}`, data);
```

**The locale** is taken from the Cognito user attribute or passed explicitly in the mutation that triggers the notification.

---

## Step 6 — i18n in Next.js apps

### 6.1 Install next-intl

```bash
npm install next-intl
```

### 6.2 Message structure

```
apps/<app>/messages/
├── es/
│   ├── common.json
│   └── [page].json
├── en/
│   └── ...
└── pt/
    └── ...
```

Messages are **not** shared between apps. Each app has its own.

### 6.3 Configuration

```typescript
// apps/<app>/src/i18n/routing.ts
import { defineRouting } from 'next-intl/routing';
export const routing = defineRouting({
  locales: ['es', 'en', 'pt'] as const,
  defaultLocale: 'es' as const,
});
```

```typescript
// apps/<app>/src/i18n/request.ts
import { getRequestConfig } from 'next-intl/server';
import { routing } from './routing';

export default getRequestConfig(async ({ requestLocale }) => {
  const locale = (await requestLocale) || routing.defaultLocale;
  return {
    locale,
    messages: (await import(`../../messages/${locale}/common.json`)).default,
  };
});
```

### 6.4 Usage in Server Components

```typescript
import { getTranslations } from 'next-intl/server';

export default async function Page() {
  const t = await getTranslations('common');
  return <h1>{t('title')}</h1>;
}
```

---

## Step 7 — Design system (packages/ui)

Rules:

- No business logic — UI only: styles, variants, layout.
- Components typed with explicit interfaces.
- Variants with CVA (class-variance-authority).
- All components exported from `src/index.ts`.

### CVA component pattern

```typescript
// packages/ui/src/atoms/button.tsx
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: 'bg-primary text-white hover:bg-primary/90',
        secondary: 'bg-transparent border border-primary text-primary',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4',
        lg: 'h-12 px-6 text-lg',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  },
);

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

/** Primary interactive element. */
export function Button({ variant, size, className, ...props }: ButtonProps) {
  return <button className={buttonVariants({ variant, size, className })} {...props} />;
}
```

### Required installations in packages/ui

```bash
npm install class-variance-authority clsx tailwind-merge
npm install react-hook-form @hookform/resolvers zod   # For molecules/form.tsx
```

---

## Step 8 — CI/CD with amplify.yml

The admin app owns the backend. The web app only consumes it.

```yaml
version: 1
applications:
  # Admin: deploys backend + its frontend
  - appRoot: apps/admin
    backend:
      phases:
        preBuild:
          commands:
            - cd ../..
            - npm ci --include=optional --cache .npm --prefer-offline
        build:
          commands:
            - cd $AMPLIFY_MONOREPO_APP_ROOT
            - npx ampx pipeline-deploy --outputs-out-dir ../../config --outputs-format json --branch $AWS_BRANCH --app-id $AWS_APP_ID
    frontend:
      phases:
        build:
          commands:
            - cd ../..
            - npx nx build admin --configuration=production
      artifacts:
        baseDirectory: .next
        files: ['**/*']
      cache:
        paths: [node_modules/**, .nx/cache/**, .next/cache/**]

  # Web: frontend only, consumes the admin backend
  - appRoot: apps/web
    frontend:
      phases:
        preBuild:
          commands:
            - cd ../..
            - npm ci --include=optional --cache .npm --prefer-offline
        build:
          commands:
            - cd $AMPLIFY_MONOREPO_APP_ROOT
            # Download outputs generated by the admin app
            - npx ampx generate outputs --out-dir ../../config --format json --branch $AWS_BRANCH --app-id $BACKEND_APP_ID
            - cd ../..
            - npx nx build web --configuration=production
      artifacts:
        baseDirectory: .next
        files: ['**/*']
      cache:
        paths: [node_modules/**, .nx/cache/**, .next/cache/**]
```

### Environment variables — three levels

**Level 1 — Local development (`.env.local`, never committed):**

```bash
# apps/admin/.env.local
APP_NAME="My App (Dev)"
NOREPLY_EMAIL="dev@example.com"
SUPPORT_EMAIL="support@example.com"
ADMIN_EMAIL="admin@example.com"
WEBHOOK_SECRET="dev-secret-local"

# apps/web/.env.local
NEXT_PUBLIC_APP_URL="http://localhost:4200"
```

`.env.local` is git-ignored. Each developer sets it up manually from a committed `.env.example` file that contains keys but no real values.

**Level 2 — Sensitive secrets (`ampx secret`, encrypted in AWS):**

For third-party API keys that must not appear in code or plain environment variables:

```bash
npx ampx secret set WEBHOOK_SECRET
npx ampx secret set PAYMENT_API_KEY
```

Reference them in `resource.ts` with `secret('NAME')`, not `process.env`.

**Level 3 — Amplify Console environment variables (non-sensitive, per environment):**

Build and runtime variables are configured in Amplify Console > App settings > Environment variables. **This configuration is not in the code — it must be documented in the project README or operations documentation:**

```
# Admin app — required variables in Amplify Console
AWS_APP_ID          → (set automatically)
APP_NAME            → "My App"
NOREPLY_EMAIL       → "noreply@domain.com"
SUPPORT_EMAIL       → "support@domain.com"
ADMIN_EMAIL         → "admin@domain.com"

# Web app — required variables in Amplify Console
BACKEND_APP_ID      → App ID of the admin app (to download amplify_outputs.json)
NEXT_PUBLIC_APP_URL → Public URL of the web app
```

---

## New project checklist

**Architecture decision:**

- [ ] The project has 3+ user roles with distinct permissions
- [ ] There are transactional operations, PII data, or SaaS integrations with webhooks
- [ ] Attack surface separation is required (admin on a separate domain)

**Scaffolding:**

- [ ] Nx workspace created
- [ ] `admin` and `web` apps generated with Nx
- [ ] `ui`, `core`, `email-templates` packages created
- [ ] Path aliases configured in `tsconfig.base.json`
- [ ] `.env.example` committed (no real values) and `.env.local` in `.gitignore`

**Amplify backend:**

- [ ] `apps/admin/amplify/` initialized with `ampx configure project`
- [ ] Each infrastructure need checked against <https://docs.amplify.aws/nextjs/build-a-backend/> before writing CDK
- [ ] `auth/resource.ts` with groups, triggers, and `access()` permissions
- [ ] `data/resource.ts` with schema, models, group/owner authorization, mutations, queries
- [ ] Lambda functions organized by domain in `functions/`
- [ ] `[integration]/` folder with handler + authorizer for each external SaaS webhook
- [ ] `shared/` with constants, types, email renderer, SaaS SDK client per integration
- [ ] Secrets configured with `ampx secret set` (not as plain environment variables)

**CDK:**

- [ ] `SESEmailIdentity` construct created
- [ ] `LambdaLayerEmailTemplates` construct created
- [ ] `backend.ts` wires layers and SES policies to notification functions
- [ ] API Gateway with Lambda authorizer for each webhook integration (if applicable)

**Frontend:**

- [ ] `lib/client.ts` in each app with `getGuestClient` / `getAuthenticatedClient`
- [ ] `actions/` with Server Actions typed with `Result<T, E>`
- [ ] `packages/ui` with atoms, molecules, and organisms
- [ ] `packages/core` with `createServerClient` and `middleware.ts`
- [ ] i18n configured with next-intl in both apps

**Email:**

- [ ] `packages/email-templates/` configured with Maizzle
- [ ] SES: domain verified, DNS configured (DKIM, SPF, DMARC), sandbox exit requested for production

**CI/CD and operations:**

- [ ] `amplify.yml` with admin as backend owner and web as consumer
- [ ] Amplify Console environment variables documented in the project README
- [ ] `BACKEND_APP_ID` configured in the web app in Amplify Console
