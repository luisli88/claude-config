---
name: ui-component
description: Generate a React component following the atomic design pattern with Tailwind v4 @theme tokens — typed props interface, variant map, Server vs Client decision, and JSDoc.
argument-hint: "Component name (PascalCase), layer (atom|molecule|organism), and optional variants (comma-separated, e.g. 'primary,secondary,ghost')"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse:
- **Component name**: PascalCase. Required. Ask if not provided.
- **Layer**: `atom`, `molecule`, or `organism`. Required. Ask if not provided.
- **Variants**: comma-separated list. Optional. Default: none.

Determine the output path:
- Standalone project: `src/components/{atoms|molecules|organisms}/{ComponentName}.tsx`
- Monorepo shared package: `packages/ui/src/{atoms|molecules|organisms}/{ComponentName}.tsx`

---

## LAYER RULES

### Atom

- Base HTML element with variant and size props.
- No children composition from other atoms.
- No side effects, no data fetching, no Server Action calls.
- Prefer Server Component. Add `'use client'` only if the atom has event handlers (`onClick`, `onChange`, etc.).

```typescript
// src/components/atoms/{ComponentName}.tsx

interface {ComponentName}Props {
  variant?: {variant-union | 'default'};
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
  className?: string;
  // Add element-specific HTML props (e.g. onClick, disabled, type)
}

/** {One-line description of the atom's purpose}. */
export function {ComponentName}({ variant = '{default-variant}', size = 'md', className, children, ...props }: {ComponentName}Props) {
  const base = '{shared base classes}';
  const variants: Record<string, string> = {
    // Map variant names to Tailwind token classes — no raw hex values
    {variant-map}
  };
  const sizes: Record<string, string> = {
    sm: '',
    md: '',
    lg: '',
  };

  return (
    <{element} className={`${base} ${variants[variant ?? '{default-variant}']} ${sizes[size]} ${className ?? ''}`} {...props}>
      {children}
    </{element}>
  );
}
```

### Molecule

- Composes atoms. May contain local state (`useState`, `useForm`).
- **Never** calls Server Actions directly — receives callbacks from the parent.
- Always `'use client'` if it manages state or uses hooks.

```typescript
// src/components/molecules/{ComponentName}.tsx
'use client';

import { useTranslations } from 'next-intl';
// Import atoms as needed

interface {ComponentName}Props {
  // Callbacks from parent — never call Server Actions here
  onSubmit?: (data: unknown) => Promise<void>;
  // Other props
}

/** {One-line description}. */
export function {ComponentName}({ onSubmit, ...props }: {ComponentName}Props) {
  const t = useTranslations('{namespace}');

  return (
    <div>
      {/* Compose atoms */}
    </div>
  );
}
```

### Organism

- Full page sections. Composes molecules and atoms.
- May be Server or Client Component depending on whether it fetches data.
- **May** call Server Actions directly.
- If it fetches data: Server Component (async). If it manages interactive state: `'use client'`.

```typescript
// src/components/organisms/{ComponentName}.tsx
// Add 'use client' only if this organism manages interactive state

import { getTranslations } from 'next-intl/server'; // or useTranslations if client
// Import molecules and atoms

interface {ComponentName}Props {
  // Props — may receive pre-fetched data as Server Component
}

/** {One-line description}. */
export async function {ComponentName}({ ...props }: {ComponentName}Props) {
  const t = await getTranslations('{namespace}');

  return (
    <section>
      {/* Compose molecules and atoms */}
    </section>
  );
}
```

---

## SERVER VS CLIENT DECISION TABLE

| Condition | Component type |
|---|---|
| Reads from DB or calls a Server Action | Server Component |
| Uses `useState`, `useEffect`, `useForm`, event handlers | `'use client'` |
| Uses `useTranslations` (next-intl client hook) | `'use client'` |
| Uses `getTranslations` (next-intl server function) | Server Component |
| Wraps a third-party animation or UI library | `'use client'` |
| Receives already-fetched data as props with no interaction | Server Component |

Default to Server Component. Add `'use client'` only when one of the above conditions requires it.

---

## RULES

- All colors in className values must reference `@theme` tokens (`bg-primary-700`, `text-neutral-0`). No raw hex values.
- Props interface uses `interface`, never `type alias`. Never use `any` — not even in optional props.
- Atoms: no business logic. Molecules: no Server Action calls. Organisms: may have both.
- Component file name matches the export name exactly: `Button.tsx` exports `Button`.
- One component per file. No barrel-exporting multiple components from the same file.
- JSDoc on the exported function — one line describing the component's purpose. No multi-line blocks.

---

## CHECKLIST

- [ ] Component created at the correct path for its layer
- [ ] Props typed with explicit `interface` — no `any`, no inline type literals
- [ ] `'use client'` present only when required by the decision table
- [ ] All Tailwind classes reference `@theme` tokens — no raw hex values
- [ ] JSDoc on the exported function
- [ ] Layer rules respected: atoms don't compose, molecules don't call actions, organisms may do both
