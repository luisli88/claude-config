# Development Constitution

**Author:** Luis Ricardo Ruiz
**Version:** 1.0
**Last Updated:** 2026-04-26

---

## 1. Philosophy

### Core Values

- Simplicidad > Cleverness
- Explicit > Implicit
- Type Safety > Flexibilidad
- Código legible > Código inteligente
- [COMPLETAR: agrega 1-2 valores propios]

### Development Principles

- YAGNI — No construir lo que no se necesita hoy
- DRY — Sin duplicación de lógica de negocio
- KISS — La solución más simple que resuelve el problema
- Separation of Concerns — Cada módulo con una responsabilidad clara
- Fail Fast — Errores visibles cuanto antes, no silenciados
- [COMPLETAR: agrega principios propios si aplica]

---

## 2. Tech Stack Standards

### Frontend Web

**Required:**

- React + Next.js (App Router)
- TypeScript strict mode (`"strict": true` en tsconfig)
- [COMPLETAR: librería de estilos — Tailwind, CSS Modules, etc.]
- [COMPLETAR: librería de estado — Zustand, Jotai, React Query, etc.]
- [COMPLETAR: librería de formularios/validación — React Hook Form + Zod, etc.]

**Prohibited:**

- `any` en TypeScript sin justificación explícita
- Lógica de negocio en componentes UI
- [COMPLETAR: otras prohibiciones]

### Frontend Mobile

**Required:**

- SwiftUI
- [COMPLETAR: patrón de arquitectura — MVVM, TCA, etc.]
- [COMPLETAR: gestión de dependencias — SPM, etc.]

**Prohibited:**

- [COMPLETAR]

### Herramientas

**Required:**

- ESLint + Prettier para Web
- [COMPLETAR: herramienta de tests — Jest, Vitest, Testing Library, etc.]
- [COMPLETAR: herramienta de tests E2E — Playwright, Cypress, etc.]
- Git con Conventional Commits

---

## 3. Architecture Patterns

### Frontend Web — Next.js App Router

```
app/
├── (routes)/          # Grupos de rutas
├── _components/       # Componentes compartidos de la app
└── _lib/              # Utilidades, hooks, helpers

src/
├── components/        # Componentes reutilizables (UI puro)
├── features/          # Módulos de feature completos
│   └── [feature]/
│       ├── components/
│       ├── hooks/
│       └── types.ts
├── lib/               # Clientes externos, configuración
└── types/             # Tipos globales
```

[COMPLETAR: ajusta la estructura si la tuya es diferente]

**Regla de dependencias:**
- Componentes UI no conocen lógica de negocio
- Server Components por defecto, Client Components solo cuando necesario
- Fetch en Server Components, no en useEffect salvo casos justificados

### Frontend Mobile — SwiftUI

```
[COMPLETAR: estructura de carpetas que usas]
```

[COMPLETAR: patrón — MVVM, separación View/ViewModel/Model, etc.]

---

## 4. Code Quality Standards

### Naming Conventions

**TypeScript / JavaScript:**

- Variables y funciones: `camelCase`
- Componentes React y tipos: `PascalCase`
- Constantes globales: `UPPER_SNAKE_CASE`
- Archivos de componentes: `PascalCase.tsx`
- Archivos de utilidades/hooks: `camelCase.ts`
- Hooks: prefijo `use` — `useUserData`, `useCart`

**Swift:**

- [COMPLETAR: convenciones que usas]

### Function Guidelines

- Componentes: responsabilidad única, sin lógica de negocio mezclada con UI
- Funciones puras cuando sea posible
- Max líneas por función: [COMPLETAR — ej. 50]
- Props de componentes tipadas con interface explícita, nunca con `any`

### Documentación

- Documentar todas las funciones, estructuras y definiciones.
- Los atributos o propiedades solo se documentan si su función no es clara.
- Agregar documentación inline cuando el proceso o algoritmo es complejo.
- Salvo que el proyecto indique lo contrario.

---

## 5. Testing Standards

### Estrategia

[COMPLETAR: ¿qué priorizas? Unit, integration, E2E]

**Required:**

- [COMPLETAR: cobertura mínima — ej. 80% en lógica de negocio]
- Happy path + al menos un error path por función crítica
- Tests unitarios sin dependencias externas reales
- [COMPLETAR: mocking strategy]

**Herramientas:**

- [COMPLETAR: Jest / Vitest + Testing Library para Web]
- [COMPLETAR: XCTest para iOS]

### Qué se testea siempre

- Lógica de negocio (hooks, utils, servicios)
- Formularios con validación
- [COMPLETAR: otros]

### Qué NO se testea

- Estilos visuales
- Librerías de terceros
- [COMPLETAR]

---

## 6. Security Standards

- Nunca secretos en código: usar variables de entorno (`.env.local`, nunca commiteado)
- Validar input del usuario siempre en el servidor, no solo en el cliente
- Sanitizar output renderizado dinámicamente (prevención XSS)
- Autenticación verificada en Server Components / middleware de Next.js
- [COMPLETAR: herramienta/estrategia de auth — NextAuth, Clerk, etc.]
- Dependencias auditadas: `npm audit` antes de releases
- [COMPLETAR: otras prácticas de seguridad relevantes]

---

## 7. Non-Negotiables

Estas reglas NO se negocian NUNCA:

1. **TypeScript strict mode activo** — Sin `any` sin justificación documentada
2. **Conventional Commits** — Todo commit sigue el formato `type(scope): descripción`
3. **Sin secrets en código** — Variables de entorno siempre, sin excepción
4. **Tests antes de merge** — Las pruebas de lógica de negocio deben pasar
5. **Validación en el servidor** — Nunca confiar solo en validación del cliente
6. **Documentar funciones y estructuras** — Toda función/struct pública tiene doc
7. [COMPLETAR: agrega tus reglas innegociables]

---

> **Nota:** Este documento aplica a todos los proyectos salvo que el `CLAUDE.md` del proyecto indique explícitamente lo contrario.
