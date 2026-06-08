---
name: maizzle-email
description: Scaffold the email-templates package with Maizzle + Amplify Gen 2 + SES — directory structure, base layout, Lambda renderer helper, CDK SES construct, and Lambda Layer for template bundling.
argument-hint: "Package path (default: packages/email-templates) and SES domain (e.g. yourdomain.com)"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse:
- **Package path**: default `packages/email-templates`.
- **SES domain**: the verified domain for sending emails (e.g. `yourdomain.com`). Required — ask if not provided.

---

## INSTALLATION

```bash
npm install -D @maizzle/framework tailwindcss-preset-email juice
```

---

## DIRECTORY STRUCTURE

Create this structure manually (no Nx generator for email-templates):

```
{package-path}/
├── emails/                     # One HTML file per email type
│   ├── welcome.html
│   └── otp-code.html
├── components/                 # Reusable email components (header, button, etc.)
├── layouts/
│   └── base.html               # Master layout shared by all emails
├── partials/
│   ├── headers/
│   │   └── brand.html
│   └── footers/
│       └── brand.html
├── static/                     # Images and logos (referenced in templates)
├── config.js                   # Maizzle dev config
└── config.production.js        # Maizzle production config (inlines CSS, minifies)
```

---

## MAIZZLE CONFIGURATION

### `config.js` — Development

```javascript
export default {
  build: {
    templates: {
      source: 'emails',
      destination: { path: 'dist' },
    },
    components: { root: './' },
  },
};
```

### `config.production.js` — Production (used by Lambda)

```javascript
export default {
  build: {
    templates: {
      source: 'emails',
      destination: { path: 'dist' },
    },
    components: { root: './' },
  },
  css: {
    inline: true,      // Required for email clients
    purge: true,
    tailwind: {
      presets: ['tailwindcss-preset-email'],
    },
  },
  minify: true,
};
```

---

## BASE LAYOUT `layouts/base.html`

```html
<x-root>
  <x-head>
    <title>{{ page.title }}</title>
  </x-head>
  <body style="margin: 0; padding: 0; background-color: #f8f9fb;">
    <x-partial src="partials/headers/brand.html"></x-partial>
    <content></content>
    <x-partial src="partials/footers/brand.html"></x-partial>
  </body>
</x-root>
```

---

## EMAIL TEMPLATE PATTERN `emails/welcome.html`

```html
---
layout: layouts/base.html
title: "Bienvenido"
---

<table width="100%" cellpadding="0" cellspacing="0">
  <tr>
    <td style="padding: 32px 24px;">
      <p>Hola {{ page.userName }},</p>
      <p>Bienvenido a {{ page.appName }}.</p>
    </td>
  </tr>
</table>
```

Variables are injected via `page.*` in templates and passed from the Lambda renderer.

---

## LAMBDA RENDERER `amplify/functions/shared/email/template-renderer.ts`

```typescript
import * as fs from 'fs';
import * as path from 'path';
import { render } from '@maizzle/framework';

const LAMBDA_TEMPLATES_PATH = '/opt/email-templates';

/**
 * Renders a Maizzle email template with the provided data variables.
 * Templates are loaded from the Lambda Layer at /opt/email-templates.
 */
export async function renderEmailTemplate<T extends Record<string, unknown>>(
  templateName: string,
  data: T,
): Promise<string> {
  const templatePath = path.join(LAMBDA_TEMPLATES_PATH, 'emails', `${templateName}.html`);
  const template = fs.readFileSync(templatePath, 'utf8');
  // eslint-disable-next-line @typescript-eslint/no-var-requires
  const config = require(path.join(LAMBDA_TEMPLATES_PATH, 'config.production.js'));

  const { html } = await render(template, {
    ...config,
    components: { ...config.components, root: LAMBDA_TEMPLATES_PATH },
    locals: {
      currentYear: new Date().getFullYear(),
      ...data,
    },
    css: {
      ...config.css,
      tailwind: {
        content: (config.css?.tailwind?.content ?? []).map((p: string) =>
          path.join(LAMBDA_TEMPLATES_PATH, p),
        ),
        presets: [require('tailwindcss-preset-email')],
      },
    },
  });

  return html;
}
```

---

## i18n IN EMAIL TEMPLATES

Two patterns — choose based on content variation:

**Option A — Translation variables in `locals`** (recommended when strings are few):

```typescript
// In the Lambda notification function:
const translations = JSON.parse(
  fs.readFileSync(path.join(LAMBDA_TEMPLATES_PATH, `i18n/${locale}.json`), 'utf8'),
);
const html = await renderEmailTemplate('welcome', { ...data, t: translations });
```

```html
<!-- In the template: -->
<p>{{ page.t.greeting }}, {{ page.userName }}</p>
```

**Option B — Separate templates per locale** (recommended when layout differs significantly):

```
emails/
├── welcome.es.html
├── welcome.en.html
└── welcome.pt.html
```

```typescript
const html = await renderEmailTemplate(`welcome.${locale}`, data);
```

The locale is taken from the Cognito user attribute or passed in the AppSync mutation that triggers the notification.

---

## CDK CONSTRUCT `amplify/cdk/ses/email-identity.ts`

```typescript
import * as iam from 'aws-cdk-lib/aws-iam';
import * as ses from 'aws-cdk-lib/aws-ses';
import { Construct } from 'constructs';

interface SESEmailIdentityProps {
  domain: string;
}

/** CDK construct: SES domain identity and send policy for transactional email. */
export class SESEmailIdentity extends Construct {
  public readonly sendPolicy: iam.ManagedPolicy;

  constructor(scope: Construct, id: string, props: SESEmailIdentityProps) {
    super(scope, id);

    ses.EmailIdentity.fromEmailIdentityName(this, 'DomainIdentity', props.domain);

    this.sendPolicy = new iam.ManagedPolicy(this, 'SESPolicy', {
      statements: [
        new iam.PolicyStatement({
          actions: ['ses:SendEmail', 'ses:SendRawEmail'],
          resources: ['*'],
          conditions: {
            StringLike: { 'ses:FromAddress': `*@${props.domain}` },
          },
        }),
      ],
    });
  }
}
```

---

## CDK CONSTRUCT `amplify/cdk/functions/email-templates-layer.ts`

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Stack } from 'aws-cdk-lib';

/** Lambda Layer that bundles Maizzle email templates for use in notification functions. */
export class LambdaLayerEmailTemplates extends lambda.LayerVersion {
  constructor(scope: Stack, id: string) {
    super(scope, id, {
      code: lambda.Code.fromAsset('../../packages/email-templates', {
        bundling: {
          image: lambda.Runtime.NODEJS_20_X.bundlingImage,
          command: [
            'bash', '-c',
            'cp -r emails components layouts partials static config.js config.production.js /asset-output/opt/email-templates/',
          ],
        },
      }),
      compatibleRuntimes: [lambda.Runtime.NODEJS_20_X, lambda.Runtime.NODEJS_22_X],
      description: 'Maizzle email templates',
    });
  }
}
```

---

## `amplify/backend.ts` — Wiring

```typescript
import { LambdaLayerEmailTemplates } from './cdk/functions/email-templates-layer';
import { SESEmailIdentity } from './cdk/ses/email-identity';
import * as lambda from 'aws-cdk-lib/aws-lambda';

const backend = defineBackend({ auth, data, /* notification functions */ });

const backendStack = backend.createStack('BackendStack');

const emailLayer = new LambdaLayerEmailTemplates(backendStack, 'EmailLayer');
const sesConfig = new SESEmailIdentity(backendStack, '{Domain}SES', { domain: '{ses-domain}' });

// Attach layer and SES policy to each notification function
const notificationFunctions = [/* backend.sendWelcomeFunction, etc. */];

notificationFunctions.forEach((fn) => {
  (fn.resources.lambda as lambda.Function).addLayers(emailLayer);
  fn.resources.lambda.role?.addManagedPolicy(sesConfig.sendPolicy);
});
```

---

## REQUIRED MANUAL STEPS (cannot be automated)

1. Verify the domain in AWS SES Console: Domain identities → Verify.
2. Configure DNS records: DKIM (3 CNAME records), SPF (`TXT` record), DMARC (`TXT` record).
3. Request SES sandbox exit if the app sends to unverified recipient emails.

---

## CHECKLIST

- [ ] `packages/email-templates/` structure created with all directories
- [ ] `config.js` and `config.production.js` configured
- [ ] `layouts/base.html` with header and footer partials
- [ ] At least one email template in `emails/`
- [ ] `amplify/functions/shared/email/template-renderer.ts` created
- [ ] `SESEmailIdentity` CDK construct created in `amplify/cdk/ses/`
- [ ] `LambdaLayerEmailTemplates` CDK construct created in `amplify/cdk/functions/`
- [ ] Both constructs wired in `backend.ts`
- [ ] SES domain verified and DNS records configured (manual)
- [ ] SES sandbox exit requested for production (manual)
