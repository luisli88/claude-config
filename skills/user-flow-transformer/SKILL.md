---
name: user-flow-transformer
description: Transforms a user flows document into the input format required by /speckit-spec. ONLY run after user-flow-validator confirms the document is ready (no blocking errors).
argument-hint: "Path to the validated user flows document"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

The file path is **required**. If not provided, ask the user before continuing.

**Precondition:** this skill assumes `user-flow-validator` was already run on the same file and reported no blocking errors. If unsure, run `/user-flow-validator <path>` first.

# Skill: User Flow Transformer

## Purpose

Converts the conversational language of the user flows document into the structured format expected by `/speckit-spec`. During this process it also consolidates roles, extracts entities, screens, and business rules implicit in the steps.

---

## Step 1 — Consolidate roles

1. Extract all person/role names mentioned in the steps of all flows (excluding "the system").
2. Normalize names: unify casing and remove articles ("the seller" → "Seller").
3. For each normalized role, find its description in the "Who does what?" sections of the flows.
4. If a role lacks a sufficient description, generate a clarification question for the user. Do not assume.

Build the roles table:

```markdown
## System roles

| Role           | Description                                   | Flows where involved |
|----------------|-----------------------------------------------|----------------------|
| End user       | [Description extracted from the document]     | UF-001, UF-003       |
| [Role 2]       | [Description extracted from the document]     | UF-002               |
```

---

## Step 2 — Extract screens and forms

For each flow, analyze the steps in natural language and extract:

- **Unique screens:** every screen or app section name mentioned.
- **Forms:** steps that mention fields the user must fill in — extract the form name and the list of mentioned fields.
- **Displayed data:** steps where the system shows information — extract what data is presented to the user.

---

## Step 3 — Extract business entities

Identify the system objects that are created, modified, or queried during the flows. Use the glossary to correctly interpret domain terms.

Example entities: "Sale", "Request", "Order", "Product", "User".

For each entity, note which steps it appears in and what operation is performed (create, read, update, delete).

---

## Step 4 — Extract business rules

Look for business rules in these sources:

1. "Client notes" section of each flow → explicit rules.
2. "What does the user need to start?" section → formal preconditions.
3. `↳ Waits for` lines in the steps → role coordination rules.
4. Flow variations → conditional rules (`if [condition] → [alternative behavior]`).

---

## Step 5 — Build the /speckit-spec prompt

Generate one block per flow in this format. This is the direct input for `/speckit-spec`:

```
/speckit-spec

## Application context
Name: [Application name]
Domain: [Inferred from glossary and flows]

## Domain glossary
- [Term]: [Definition]

## Roles
- [Role]: [Description]

## Flow: [Flow name] (ID: UF-001)

**Business goal:** [Text from "What does the business want to achieve?"]

**Primary actor:** [Primary role identified]
**Secondary actors:** [Secondary roles, if any]

**Preconditions:**
- [Extracted from "What does the user need to start?"]

**Entry point:** [Text from "Where does it start?"]

**Identified screens:**
- [Screen name]: shows [identified data]
- [Form name]: fields [field1, field2, field3]

**Steps (Happy Path):**
1. [Actor] on [Screen/element]: [action] → [visible result]
2. System: [automatic action] → [visible result]
...
N. [Actor] on [Screen]: [action] → [result] [BLOCKED UNTIL: step M completed by Role X]

**Final result:** [Text from "What does the user get at the end?"]

**Alternative paths:**
- [Variation name]: If [condition] → [different steps] → [alternative result]

**Business rules:**
- [Extracted from client notes and identified conditions]

**Entities involved:**
- [Entity]: [operations: create / read / update / delete]
```

---

## Step 6 — Present and confirm

1. Show the consolidated roles table and ask: *"Do these roles and descriptions correctly represent the people who will use the application?"*
2. If the user adjusts anything, update the table and the generated prompt accordingly.
3. Show the final prompt and ask: *"Shall we proceed with spec generation in Speckit?"*
4. If confirmed, deliver the prompt ready to be executed.
