---
name: user-flow-validator
description: Validates that a user flows document is complete and consistent before passing it to the Speckit transformation skill. Always run this before user-flow-transformer.
argument-hint: "Path to the user flows document"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

The file path is **required**. If not provided, ask the user before continuing.

# Skill: User Flow Validator

## Purpose

Reviews the provided user flows document across two dimensions:

1. **Completeness:** all required fields (`[req]`) are present and non-empty.
2. **Consistency:** information across flows and the glossary is coherent.

If there are blocking errors, the skill **stops** and does not invoke the transformation skill.
If there are only warnings, the skill reports them and lets the user decide whether to continue.

---

## Step 1 — Parse the document

Identify the document structure:

- **General header:** lines before the first `## Flow:`
- **Flows:** each block delimited by `## Flow:` up to the next `## Flow:` or `## Glossary`
- **Glossary:** the `## Business terms glossary` block

Extract from each flow:
- ID (`UF-NNN`), Status, Name
- Text of each `###` section
- Step list (numbered items under `### Steps`)
- Step dependencies: lines with `↳ Waits for` — extract the actor name and the referenced step number
- Variations (if any)
- Client notes (if any)
- People and roles mentioned in the steps and in the "Who does what?" section

---

## Step 2 — Completeness validations

Run these checks on each flow. Classify each failure as **[BLOCKS]** or **[WARNS]**.

### Flow fields

| Check | Type |
|---|---|
| Flow name is not the literal placeholder `[Flow name]` | BLOCKS |
| ID has the format `UF-NNN` with an integer | BLOCKS |
| Status is one of: `Draft`, `Reviewed`, `Approved` | BLOCKS |
| "What does the business want to achieve?" has real content | BLOCKS |
| "Who does what?" has real content | BLOCKS |
| "Where does it start?" has real content | BLOCKS |
| "Steps" has at least 2 items with real content | BLOCKS |
| "What does the user get at the end?" has real content | BLOCKS |
| Status is `Approved` or `Reviewed` (not `Draft`) | WARNS |

### Flow steps

For each numbered item in the `### Steps` section:

| Check | Type |
|---|---|
| Step is not a literal placeholder (does not contain `[Ex:` unreplaced) | BLOCKS |
| Step mentions who performs the action (an actor or "the system") | WARNS |
| If the step involves a form, it mentions at least one field | WARNS |
| If the step shows a screen with data, it mentions what data is displayed | WARNS |
| `↳ Waits for` lines reference a step number that exists in the same list | BLOCKS |
| `↳ Waits for` lines mention a different actor than the current step | WARNS |

### Variations (if any)

| Check | Type |
|---|---|
| Variation has a name (not the placeholder `[Name]`) | BLOCKS |
| All three fields (When?, What changes?, How does it end?) have real content | BLOCKS |

---

## Step 3 — Consistency validations

### Across flows

| Check | Type |
|---|---|
| No two flows share the same ID (`UF-NNN`) | BLOCKS |
| No two flows share the same name | BLOCKS |
| IDs are sequential with no gaps (UF-001, UF-002, UF-003…) | WARNS |

### Role consistency

1. Extract all person/role names mentioned in the steps of all flows (excluding "the system" and "the user").
2. Verify that each extracted name is described in the "Who does what?" section of at least one flow.
3. If a name appears in steps but is never described: **[BLOCKS]**.
4. If the same role appears in different forms (e.g. "Seller", "the seller", "seller"): **[WARNS]** with a suggestion to unify.

### Glossary

1. Extract all terms defined in the glossary.
2. Search through all flow text for domain-specific nouns: capitalized words not at the start of a sentence, quoted words, terms that repeat frequently and are not common words.
3. If a potential business term is not in the glossary: **[WARNS]** with the candidate list for the user to decide.

---

## Step 4 — Generate the report

```
# Validation report — [filename]

## Summary
- Flows analyzed: N
- Blocking errors: N
- Warnings: N
- Ready to transform?: YES / NO

---

## Blocking errors
> These must be fixed before continuing.

### Flow UF-001 — [Name]
- [Error description] → [Specific section or step where it occurs]

---

## Warnings
> These do not block the process, but are recommended for review.

### General
- [Warning description]

### Flow UF-002 — [Name]
- [Warning description]

---

## Glossary candidates
The following terms appear in the flows and may need a definition:
- "[Term]" — appears in: UF-001 step 3, UF-002 step 1
```

---

## Step 5 — Continuation decision

- If there are **blocking errors:** show the report and stop. Do not invoke the transformation skill.
- If there are **only warnings:** show the report and ask: *"Do you want to fix the warnings before continuing, or shall we proceed with the transformation?"*
- If there are no errors or warnings: inform the user the document is ready and invoke the `user-flow-transformer` skill.
