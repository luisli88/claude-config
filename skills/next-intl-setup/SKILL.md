---
name: next-intl-setup
description: Add next-intl i18n to a Next.js App Router project — routing config, middleware, request config, locale-aware navigation helpers, root layout update, and per-locale message directory structure.
argument-hint: "App source path (default: src/) and optional locales (default: es,en,pt)"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse:
- **App source path**: default `src/`. In a monorepo, this will be `apps/web/src/` or `apps/admin/src/`.
- **Locales**: comma-separated list. Default `es,en,pt`. First locale is the `defaultLocale`.

Do not ask if not provided — apply defaults.

---

## INSTALLATION

```bash
npm install next-intl
```

---

## FILES TO CREATE OR UPDATE

All paths below are relative to the provided app source path.

### 1. `next.config.ts` (project root or app root)

```typescript
import createNextIntlPlugin from 'next-intl/plugin';

const withNextIntl = createNextIntlPlugin('./src/i18n/request.ts');

const nextConfig = {};

export default withNextIntl(nextConfig);
```

### 2. `src/i18n/routing.ts`

```typescript
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['es', 'en', 'pt'],
  defaultLocale: 'es',
});
```

### 3. `src/i18n/request.ts`

```typescript
import { getRequestConfig } from 'next-intl/server';
import { routing } from './routing';

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale;

  if (!locale || !routing.locales.includes(locale as never)) {
    locale = routing.defaultLocale;
  }

  return {
    locale,
    messages: (await import(`../../messages/${locale}/common.json`)).default,
  };
});
```

### 4. `src/i18n/navigation.ts`

```typescript
import { createNavigation } from 'next-intl/navigation';
import { routing } from './routing';

export const { Link, redirect, usePathname, useRouter, getPathname } =
  createNavigation(routing);
```

### 5. `src/middleware.ts`

```typescript
import createMiddleware from 'next-intl/middleware';
import { routing } from './i18n/routing';

export default createMiddleware(routing);

export const config = {
  matcher: ['/((?!_next|_vercel|.*\\..*).*)'],
};
```

### 6. `src/app/[locale]/layout.tsx` — Update root layout

Rename `src/app/layout.tsx` to `src/app/[locale]/layout.tsx` if it hasn't been done. Wrap children with `NextIntlClientProvider`:

```typescript
import { NextIntlClientProvider } from 'next-intl';
import { getMessages } from 'next-intl/server';

interface Props {
  children: React.ReactNode;
  params: Promise<{ locale: string }>;
}

/** Root layout: injects i18n context and shared UI wrappers. */
export default async function LocaleLayout({ children, params }: Props) {
  const { locale } = await params;
  const messages = await getMessages();

  return (
    <html lang={locale}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

### 7. Message files

Create the per-locale directory structure. Spanish is always the source — create English and Portuguese as copies with translated values.

```
messages/
├── es/
│   └── common.json
├── en/
│   └── common.json
└── pt/
    └── common.json
```

Seed `messages/es/common.json` with the project's initial namespaces. Namespace by page or domain:

```json
{
  "nav": {
    "home": "Inicio"
  },
  "common": {
    "loading": "Cargando...",
    "error": "Ocurrió un error"
  }
}
```

---

## USAGE IN COMPONENTS

**Server Components** (async):
```typescript
import { getTranslations } from 'next-intl/server';

export default async function Page() {
  const t = await getTranslations('nav');
  return <h1>{t('home')}</h1>;
}
```

**Client Components** (`'use client'`):
```typescript
import { useTranslations } from 'next-intl';

export function NavBar() {
  const t = useTranslations('nav');
  return <nav>{t('home')}</nav>;
}
```

**Navigation** — always import from `@/i18n/navigation`, never from `next/link` or `next/navigation`:
```typescript
import { Link, useRouter, usePathname } from '@/i18n/navigation';
```

---

## RULES

- All locale directories must have identical file and key structures. Missing keys cause runtime errors.
- Spanish (`es`) is always the source language. English and Portuguese are derived from it.
- `Link`, `useRouter`, `usePathname` must be imported from `@/i18n/navigation` — not from `next/link` or `next/navigation`.
- Add new message files (e.g., `dashboard.json`) when a page's translations outgrow `common.json`. Update all three locales simultaneously.
- In a monorepo, each app has its own `messages/` directory — translations are not shared between apps.

---

## CHECKLIST

- [ ] `next-intl` installed
- [ ] `next.config.ts` wraps the config with `withNextIntl`
- [ ] `src/i18n/routing.ts` defines `locales` and `defaultLocale`
- [ ] `src/i18n/request.ts` loads from `messages/${locale}/common.json`
- [ ] `src/i18n/navigation.ts` exports locale-aware `Link`, `useRouter`, `usePathname`
- [ ] `src/middleware.ts` uses `createMiddleware(routing)`
- [ ] Root layout is under `src/app/[locale]/` and wraps with `NextIntlClientProvider`
- [ ] `messages/es/`, `messages/en/`, `messages/pt/` directories created with `common.json`
- [ ] All three locale files have identical key structures
