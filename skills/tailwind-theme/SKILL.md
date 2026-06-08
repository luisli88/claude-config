---
name: tailwind-theme
description: Configure Tailwind v4 @theme design tokens in a Next.js project — CSS layer setup, color scale, typography, and spacing tokens following Luis's token conventions.
argument-hint: "Optional color overrides as 'key:hex' pairs, e.g. 'primary:#233369 secondary:#34508f'. Omit to use defaults."
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse optional `key:hex` color overrides. If none provided, use the defaults defined below. Do not ask — apply defaults and note which ones were used in the output.

Detect the target path:
- Standalone project: `src/app/globals.css`
- Monorepo shared package: `packages/ui/src/styles/globals.css`

If ambiguous, ask once before writing.

---

## `globals.css` — Full File Content

Write this file at the detected path. Do not preserve previous content inside `@theme` — replace it entirely.

```css
@import "tailwindcss";

@theme {
  /* ── Primary brand scale ─────────────────────────────────── */
  --color-primary-50:  #eef1f9;
  --color-primary-100: #d5dcf0;
  --color-primary-200: #aab9e1;
  --color-primary-300: #7f96d2;
  --color-primary-400: #5573c3;
  --color-primary-500: #3a57af;
  --color-primary-600: #2d4490;
  --color-primary-700: #233369;
  --color-primary-800: #192449;
  --color-primary-900: #111a3d;

  /* ── Secondary ────────────────────────────────────────────── */
  --color-secondary-600: #34508f;

  /* ── Neutral scale ────────────────────────────────────────── */
  --color-neutral-0:   #ffffff;
  --color-neutral-50:  #f8f9fb;
  --color-neutral-100: #f0f2f6;
  --color-neutral-200: #e2e6ee;
  --color-neutral-400: #9aa3b6;
  --color-neutral-600: #5a6480;
  --color-neutral-800: #2e3347;
  --color-neutral-900: #0f1520;

  /* ── Accent ───────────────────────────────────────────────── */
  --color-accent-gold:       #c9a84c;
  --color-accent-gold-light: #e8d5a0;
  --color-accent-sky:        #5b9bd5;

  /* ── Typography ───────────────────────────────────────────── */
  --font-family-display: "Inter", sans-serif;
  --font-family-body:    "Inter", sans-serif;

  /* ── Add spacing, radius, and shadow tokens below as needed ── */
}
```

Apply any provided overrides to the corresponding `--color-*` values before writing.

---

## RULES

- Token names map 1:1 to Tailwind utility classes: `--color-primary-700` → `bg-primary-700`, `text-primary-700`, `border-primary-700`.
- Never use raw hex values in component files. Always reference a `--color-*` token.
- `--color-neutral-0` is `#ffffff` — use `text-neutral-0` instead of `text-white` so all colors stay tokenized.
- Add tokens only when the project needs them. Do not predefine scales for values that will never be used.
- For a monorepo shared package, both apps import this file. Do not duplicate it per app.
- Replace `Inter` with the project's actual font family. If using Google Fonts, add the `<link>` to the root layout — do not `@import` inside a CSS file that Tailwind processes.
- Tailwind v4 requires `@import "tailwindcss"` — not the v3 `@tailwind base/components/utilities` directives.
- `next.config.ts` does not need a manual Tailwind config path — Tailwind v4 auto-detects via the CSS import.

---

## CHECKLIST

- [ ] `globals.css` uses `@import "tailwindcss"` (Tailwind v4 syntax)
- [ ] All project colors are defined as `--color-*` tokens inside `@theme`
- [ ] No raw hex values in any component file
- [ ] Font family matches the project's actual fonts
- [ ] For monorepo: file lives in `packages/ui/src/styles/globals.css` and is imported by both apps
