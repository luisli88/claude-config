---
name: iac-template-specify
description: |
  Guides /speckit.specify for a client project using @iac-template/core.
  Use this skill when running /speckit.specify on a project that will consume the IaC template library.
  Ensures the spec captures scenario, layers, external systems, and client constraints
  using the controlled vocabulary required by /speckit.plan.
---

# IaC Template — Specify Guide

When running `/speckit.specify` on a project that uses `@iac-template/core`, follow this guide to produce a `spec.md` that the `/speckit.plan` skill can process automatically.

---

## Step 1: Identify the Scenario

Ask these questions in order. The first match wins.

| Question | If YES |
|---|---|
| Does the client control their own server (VPS, on-premise, government infrastructure)? | **E3** |
| Does the project integrate with external systems not owned or deployed by this project? | **E2** |
| Is all infrastructure owned and managed by this project on AWS? | **E1** |

Document the scenario and its justification at the top of `spec.md` under a `## Scenario` section.

---

## Step 2: Capture Layer Requirements

### For E1 and E2

For each layer, identify the required variant using the controlled vocabulary:

**Data Layer**
- Does the project need structured, relational data with transactions? → `data layer: relational engine`
- Does the project need key-value, document, or high-throughput data? → `data layer: non-relational engine`
- Both may be active simultaneously.

**Integration Layer**
- Does the project execute business logic as discrete, event-driven functions? → `integration layer: functions mode`
- Does the project run long-lived services or multi-step workloads in containers? → `integration layer: containers mode`
- Both may be active simultaneously.
- If functions mode: list the functional domains (e.g., `orders`, `payments`, `notifications`). A shared Lambda Layer is created automatically when there are 2 or more domains.

**Presentation Layer**
- Does the project expose a public or authenticated API / BFF? → `presentation layer: BFF mode`
  - What is the security level? `public`, `authenticated`, or `internal-network`
  - If `authenticated`: an existing Cognito User Pool must be provided as a parameter (it is never created by the template).
- Does the project serve a static frontend? → `presentation layer: static site`

**External Systems (E2 only)**
- For each external system: `external system integration: [name]`
- Document the access method: endpoint URL stored in Secrets Manager, or direct ARN.
- The system is never described as a CDK resource — only as a reference.

### For E3

- What is the runtime version? (e.g., `node20`, `python3.12`, `go1.22`, `swift5.10`)
- What libraries are allowed? (exact npm/pip/go module names as approved by the client)
- Are there additional constraints? (port restrictions, environment variables, file system limits)

---

## Step 3: Spec Structure

The `spec.md` produced by `/speckit.specify` must include:

```markdown
## Scenario
[E1 / E2 / E3] — [one-sentence justification]

## Layers (E1/E2 only)
- Data: [relational engine / non-relational engine / both]
- Integration: [functions mode / containers mode / both]
  - Functional domains: [list if functions mode]
- Presentation: [BFF mode / static site / both]
  - Security level: [public / authenticated / internal-network]

## External Systems (E2 only)
- [name]: [access method]

## Client Constraints (E3 only)
- Runtime: [version]
- Allowed libraries: [list]

## User Stories
[Standard spec kit user stories, each referencing the layer/construct it exercises]
```

---

## Step 4: User Story Guidelines

- Each user story must map to one or more layers from the controlled vocabulary.
- User stories for E3 describe application behavior, not infrastructure behavior.
- Include acceptance criteria that can be validated without a real AWS deployment (tests use `Template.fromStack` assertions).
- Do not write user stories for Cognito User Pool creation — it is always external to the project.
