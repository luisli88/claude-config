---
name: webhook-integration
description: Add a CDK API Gateway + Lambda authorizer webhook integration to an Amplify Gen 2 backend — handler, authorizer with provider signature verification (HMAC/Bearer/IP), and API Gateway resource wired in backend.ts.
argument-hint: "Provider name (e.g. stripe, mercadopago, twilio) and auth mechanism (hmac|bearer|ip)"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse:
- **Provider name**: kebab-case. Required. Used as the directory and resource name. Ask if not provided.
- **Auth mechanism**: `hmac`, `bearer`, or `ip`. Required — ask if not provided.
  - `hmac`: provider sends a signature header with an HMAC-SHA256 of the payload (Stripe, MercadoPago, GitHub).
  - `bearer`: provider sends a static secret token in the Authorization or a custom header (Twilio, SendGrid).
  - `ip`: validate the request origin IP against a provider allowlist.

Derive `PascalCaseName` and `camelCaseName` from the provider name.

---

## FILES TO CREATE

### `amplify/functions/{provider}/resource.ts`

```typescript
import { defineFunction, secret } from '@aws-amplify/backend';

/** Lambda handler for {provider} webhook events. */
export const {camelCaseName}Function = defineFunction({
  name: '{provider}-webhook',
  timeoutSeconds: 30,
  environment: {
    WEBHOOK_SECRET: secret('{PROVIDER_WEBHOOK_SECRET}'),
  },
});

/** Lambda authorizer for {provider} webhook requests. */
export const {camelCaseName}AuthorizerFunction = defineFunction({
  name: '{provider}-authorizer',
  timeoutSeconds: 5,
  environment: {
    WEBHOOK_SECRET: secret('{PROVIDER_WEBHOOK_SECRET}'),
  },
});
```

Register the secret before deploying:
```bash
npx ampx secret set {PROVIDER_WEBHOOK_SECRET}
```

### `amplify/functions/{provider}/handler.ts`

```typescript
import type { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';

/**
 * Processes a verified {provider} webhook event.
 * The authorizer runs before this handler — assume the request is authentic.
 */
export const handler = async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
  const body = JSON.parse(event.body ?? '{}');
  const eventType: string = body.type ?? body.event ?? '';

  switch (eventType) {
    case '{EVENT_TYPE}':
      // Handle the event
      break;
    default:
      // Unhandled event type — acknowledge and ignore
      break;
  }

  return { statusCode: 200, body: JSON.stringify({ received: true }) };
};
```

### `amplify/functions/{provider}/authorizer.ts`

Select the template for the chosen auth mechanism.

#### HMAC (Stripe, MercadoPago, GitHub)

```typescript
import type { APIGatewayAuthorizerResult, APIGatewayRequestAuthorizerEvent } from 'aws-lambda';
import { createHmac, timingSafeEqual } from 'crypto';

/** Verifies the {provider} webhook signature using HMAC-SHA256. */
export const handler = async (
  event: APIGatewayRequestAuthorizerEvent,
): Promise<APIGatewayAuthorizerResult> => {
  const signature = event.headers?.['x-{provider}-signature'] ?? '';
  const secret = process.env.WEBHOOK_SECRET ?? '';
  const payload = event.body ?? '';

  const expected = createHmac('sha256', secret).update(payload).digest('hex');
  const isValid = timingSafeEqual(Buffer.from(signature), Buffer.from(`sha256=${expected}`));

  return buildPolicy('webhook', isValid ? 'Allow' : 'Deny', event.methodArn);
};

function buildPolicy(
  principalId: string,
  effect: 'Allow' | 'Deny',
  resource: string,
): APIGatewayAuthorizerResult {
  return {
    principalId,
    policyDocument: {
      Version: '2012-10-17',
      Statement: [{ Action: 'execute-api:Invoke', Effect: effect, Resource: resource }],
    },
  };
}
```

#### Bearer token (Twilio, SendGrid)

```typescript
import type { APIGatewayAuthorizerResult, APIGatewayRequestAuthorizerEvent } from 'aws-lambda';
import { timingSafeEqual } from 'crypto';

/** Verifies the {provider} webhook using a static Bearer token. */
export const handler = async (
  event: APIGatewayRequestAuthorizerEvent,
): Promise<APIGatewayAuthorizerResult> => {
  const authHeader = event.headers?.['authorization'] ?? '';
  const token = authHeader.replace(/^Bearer\s+/i, '');
  const secret = process.env.WEBHOOK_SECRET ?? '';

  const isValid =
    token.length === secret.length &&
    timingSafeEqual(Buffer.from(token), Buffer.from(secret));

  return buildPolicy('webhook', isValid ? 'Allow' : 'Deny', event.methodArn);
};
```

#### IP allowlist

```typescript
import type { APIGatewayAuthorizerResult, APIGatewayRequestAuthorizerEvent } from 'aws-lambda';

// Update this list with the provider's published IP ranges
const ALLOWED_IPS = new Set(['x.x.x.x', 'y.y.y.y']);

/** Validates the {provider} webhook origin IP. */
export const handler = async (
  event: APIGatewayRequestAuthorizerEvent,
): Promise<APIGatewayAuthorizerResult> => {
  const sourceIp = event.requestContext?.identity?.sourceIp ?? '';
  const isValid = ALLOWED_IPS.has(sourceIp);

  return buildPolicy('webhook', isValid ? 'Allow' : 'Deny', event.methodArn);
};
```

---

## `amplify/backend.ts` — API Gateway wiring

Add inside `backend.createStack()`. One `RestApi` can hold multiple webhook integrations as separate resources.

```typescript
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as cdk from 'aws-cdk-lib';
import { {camelCaseName}Function, {camelCaseName}AuthorizerFunction } from './functions/{provider}/resource';

const backend = defineBackend({ auth, data, {camelCaseName}Function, {camelCaseName}AuthorizerFunction });

const backendStack = backend.createStack('BackendStack');

// Reuse existing webhooksApi if it already exists — add a new resource to it
const webhooksApi = new apigateway.RestApi(backendStack, 'WebhooksApi', {
  restApiName: 'webhooks-api',
});

const {provider}Resource = webhooksApi.root
  .addResource('webhooks')
  .addResource('{provider}');

const {provider}Authorizer = new apigateway.RequestAuthorizer(
  backendStack,
  '{PascalCaseName}Authorizer',
  {
    handler: backend.{camelCaseName}AuthorizerFunction.resources.lambda,
    identitySources: [
      // HMAC/Bearer: the header carrying the signature or token
      apigateway.IdentitySource.header('x-{provider}-signature'),
      // IP: use apigateway.IdentitySource.ip() instead
    ],
    resultsCacheTtl: cdk.Duration.seconds(0), // Disable caching — every request must be verified
  },
);

{provider}Resource.addMethod(
  'POST',
  new apigateway.LambdaIntegration(backend.{camelCaseName}Function.resources.lambda),
  {
    authorizationType: apigateway.AuthorizationType.CUSTOM,
    authorizer: {provider}Authorizer,
  },
);

new cdk.CfnOutput(backendStack, '{PascalCaseName}WebhookUrl', {
  value: `${webhooksApi.url}webhooks/{provider}`,
  description: 'Configure this URL in the {provider} dashboard as the webhook endpoint',
});
```

---

## RULES

- Always use `timingSafeEqual` for HMAC and Bearer comparisons — never `===`. Timing-safe comparison prevents timing attacks.
- Set `resultsCacheTtl: Duration.seconds(0)` on the authorizer — webhook requests must always be re-verified, never served from cache.
- HMAC signatures must be verified against the raw request body, not a parsed object. Ensure API Gateway passes the raw body.
- Secrets go in `ampx secret set` — never hardcoded in resource.ts or handler.ts.
- Acknowledge all webhook events with HTTP 200 even if the event type is not handled. Returning 4xx causes providers to retry indefinitely.
- One `RestApi` per project. Add new integrations as additional `addResource()` calls on the existing `webhooksApi`.

---

## CHECKLIST

- [ ] `amplify/functions/{provider}/resource.ts` with handler + authorizer functions
- [ ] `amplify/functions/{provider}/handler.ts` with event type routing
- [ ] `amplify/functions/{provider}/authorizer.ts` with the correct auth mechanism
- [ ] Secret registered with `npx ampx secret set {PROVIDER_WEBHOOK_SECRET}`
- [ ] Both functions registered in `defineBackend()` in `backend.ts`
- [ ] API Gateway resource added to `webhooksApi` with custom authorizer
- [ ] `timingSafeEqual` used for all token/signature comparisons
- [ ] `resultsCacheTtl` set to 0 on the authorizer
- [ ] Webhook URL output via `CfnOutput` — configure it in the provider dashboard
