---
name: iac-template-plan
description: |
  Guides /speckit.plan for a client project using @iac-template/core.
  Use when running /speckit.plan on a project whose spec.md was produced with the
  iac-template-specify skill. Reads the scenario and layer requirements, maps them
  to constructs using the controlled vocabulary, and generates the CDK composition
  code in infra/app.ts using @iac-template/core.
---

# IaC Template — Plan Guide

When running `/speckit.plan` on a project using `@iac-template/core`, follow this guide to produce a `plan.md` that activates the correct constructs and generates the composition code.

---

## Step 1: Read the Scenario

Open `spec.md` and locate the `## Scenario` section. The scenario determines the entire plan structure:

- **E1**: all infrastructure is owned by this project → activate constructs, compose in `TemplateStack(mode: OperationMode.E1)`
- **E2**: same as E1, plus at least one integration construct declares `externalIntegrations` → compose in `TemplateStack(mode: OperationMode.E2)`
- **E3**: no AWS infrastructure → call `E3Generator.generate()`, no `TemplateStack`, no constructs

---

## Step 2: Map Requirements to Constructs

Use this mapping table. Apply it to every layer requirement in `spec.md`:

| Vocabulary term in spec.md | Import from `@iac-template/core` | Key parameters |
|---|---|---|
| `data layer: relational engine` | `DataLayerRelational` | `engine?: 'postgres' \| 'mysql'` (default: `postgres`) |
| `data layer: non-relational engine` | `DataLayerNonRelational` | — |
| `integration layer: functions mode` | `IntegrationLayerLambda` | `functionalDomains: string[]`, `dataLayers: DataLayerConnection[]`, `externalIntegrations?: ExternalIntegration[]` |
| `integration layer: containers mode` | `IntegrationLayerContainer` | `dataLayers: DataLayerConnection[]`, `externalIntegrations?: ExternalIntegration[]` |
| `presentation layer: BFF mode` | `PresentationLayerBFF` | `securityLevel: 'public' \| 'authenticated' \| 'internal-network'`, `integrationLayer: IntegrationEntryPoint`, `userPool?: IUserPool` (required if authenticated) |
| `presentation layer: static site` | `PresentationLayerStaticSite` | — |
| `external system integration: [name]` | — | `externalIntegrations: [{ name: '[name]', endpointSecretArn: '...' }]` on integration construct |
| `no template infrastructure` | `E3Generator` | `runtimeVersion: string`, `allowedLibraries: string[]` |

---

## Step 3: Generate the Composition Code

### E1 / E2 — `infra/app.ts`

```typescript
import { App } from 'aws-cdk-lib';
import { TemplateStack, OperationMode } from '@iac-template/core';
// Import only the constructs activated by spec.md

const app = new App();

new TemplateStack(app, 'MyProjectStack', {
  mode: OperationMode.E1, // or E2
  env: { account: process.env.CDK_ACCOUNT, region: process.env.CDK_REGION },
});
```

Inside `TemplateStack`, instantiate only the constructs from Step 2:

```typescript
// Example: E1 with relational data + functions + BFF (authenticated)
const data = new DataLayerRelational(this, 'Data', { engine: 'postgres' });

const integration = new IntegrationLayerLambda(this, 'Integration', {
  functionalDomains: ['orders', 'payments'], // from spec.md
  dataLayers: [data],
});

new PresentationLayerBFF(this, 'Presentation', {
  securityLevel: 'authenticated',
  integrationLayer: integration,
  userPool: existingUserPool, // always a parameter, never created here
});
```

**E2 addition** — declare external systems on the integration construct:

```typescript
const integration = new IntegrationLayerLambda(this, 'Integration', {
  functionalDomains: ['orders'],
  dataLayers: [data],
  externalIntegrations: [
    { name: 'payment-gateway', endpointSecretArn: 'arn:aws:secretsmanager:...' },
  ],
});
```

### E3 — `infra/generate.ts`

```typescript
import { E3Generator } from '@iac-template/core';

E3Generator.generate({
  runtimeVersion: 'node20',           // from spec.md client constraints
  allowedLibraries: ['express', 'pg'], // from spec.md client constraints
  outputDir: './docker',
});
```

No `TemplateStack`, no constructs, no `aws-cdk-lib` stack instantiation.

---

## Step 4: Plan Structure in `plan.md`

The plan must include:

1. **Scenario confirmation** — restate E1/E2/E3 and the justification from `spec.md`
2. **Construct mapping table** — which constructs are activated and why (one row per layer)
3. **Generated `app.ts` (or `generate.ts` for E3)** — the full composition code, ready to copy
4. **Parameters per construct** — explicit values for every non-default parameter
5. **What is NOT in scope** — any responsibility explicitly deferred (e.g., Cognito User Pool creation, VPC strategy, WAF)
6. **Constitution check** — confirm no raw CDK is written for responsibilities covered by the library

---

## Step 5: Constraints and Guardrails

- **Never create a Cognito User Pool** inside the plan. If `securityLevel: 'authenticated'` is required, document that `userPool` must be provided as a parameter from an existing pool.
- **Never describe an external system as a CDK resource.** External systems appear only as `endpointSecretArn` strings.
- **Lambda Layer is automatic.** Do not plan a manual Layer — `IntegrationLayerLambda` creates a shared Layer when `functionalDomains` has 2 or more entries.
- **E3 and CDK are mutually exclusive.** If the scenario is E3, no CDK import or construct appears in the plan.
- **`OperationMode` is set once.** Do not plan conditional logic that changes the mode at runtime.
