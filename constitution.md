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
- Mantenibilidad > Escalabilidad
- Código y archivos legibles > estructuras rigidas en archivos

### Development Principles

- YAGNI — No construir lo que no se necesita hoy
- DRY — Sin duplicación de lógica de negocio
- KISS — La solución más simple que resuelve el problema
- Separation of Concerns — Cada módulo con una responsabilidad clara
- Fail Fast — Errores visibles cuanto antes, no silenciados
- Dependency Inversion Principle - Usa abstracciones y no entidades concretas
- Boy Scout Rule - Deja el código limpio.

---

## 2. Tech Stack Standards

### Frontend Web

**Required:**

- React + Next.js (App Router)
- TypeScript strict mode (`"strict": true` en tsconfig)
- TailwindCSS, CSS Modules, SASS/SCSS
- Uso de estados usando las opciones por defecto de React: useState, useReduce, createContext.
- useForm Hooks, useTable Hooks.

**Prohibited:**

- `any` en TypeScript sin justificación explícita
- Lógica de negocio en componentes UI
- Exponer API para consumo externo por medio de NextJS.

### Frontend Mobile — iOS

**Required:**

- SwiftUI (no UIKit salvo integración puntual justificada)
- MVVM — ViewModel como única fuente de verdad de la vista
- SwiftData para persistencia local
- Alamofire para networking (tipado con `Decodable` siempre)
- CocoaPods para gestión de dependencias
- `async/await` para operaciones asíncronas

**Prohibited:**

- Lógica de negocio en Views (`body` solo para layout y binding)
- Llamadas directas a Alamofire desde una View
- Operaciones de SwiftData fuera del ViewModel o Repository
- Forzar unwrap (`!`) sin guard previo documentado

### Backend — AWS Amplify Gen 2

Stack por defecto para proyectos nuevos. Si el proyecto indica otro stack, ese tiene precedencia.

**Required:**

- AWS Amplify Gen 2 (`@aws-amplify/backend`) como framework de infraestructura
- TypeScript en todas las Lambda functions y recursos CDK
- DynamoDB vía Amplify Data (`a.schema()` con AppSync GraphQL)
- Cognito para autenticación y grupos de usuarios
- Lambda functions organizadas por dominio con `resource.ts` + `handler.ts`
- CDK para recursos AWS que Amplify no cubre (SES, API Gateway custom, layers)

**Estructura de backend:**

```
amplify/
├── backend.ts                  # Composition root (defineBackend)
├── auth/
│   └── resource.ts             # Configuración de Cognito
├── data/
│   └── resource.ts             # Schema DynamoDB (AppSync)
├── functions/
│   ├── [domain]/
│   │   ├── resource.ts         # Definición de la Lambda (defineFunction)
│   │   └── handler.ts          # Handler de la Lambda
│   ├── cognito-triggers/       # Triggers de Cognito (pre-signup, post-confirmation, etc.)
│   ├── notifications/          # Funciones de notificación por email
│   ├── services/               # Funciones de servicios de negocio
│   └── shared/                 # Utilidades compartidas entre funciones
│       ├── common/             # Tipos, validaciones, identidad, auditoría
│       ├── email/              # Helpers de envío de email (SES)
│       ├── payment/            # Lógica de pagos compartida
│       └── [domain]/           # Tipos y helpers por dominio
└── cdk/                        # Constructs CDK personalizados
    ├── ses/                    # Configuración de SES
    └── functions/              # Lambda Layers, etc.
```

**Reglas:**

- `backend.ts` es el único punto de composición — no importar funciones entre sí directamente
- Lógica compartida entre funciones va en `functions/shared/`, nunca duplicada
- Autorización declarada en el schema (`allow.groups()`, `allow.owner()`) — no en los handlers
- Variables de entorno inyectadas por Amplify o CDK, nunca hardcodeadas
- Un `resource.ts` por función — define nombre, runtime y permisos IAM necesarios

**Prohibited:**

- Lógica de negocio en `backend.ts` (solo composición)
- Acceso directo a DynamoDB desde funciones que pueden usar el cliente de Amplify Data
- Secrets en código — usar Parameter Store o variables de entorno de Amplify

### Herramientas

**Required:**

- ESLint + Prettier para Web
- Jest, Testing Library
- Playwright
- XCTest para iOS
- SwiftLint para linting en iOS
- Git con Conventional Commits

---

## 3. Architecture Patterns

### Frontend Web — Next.js Monorepo (Nx)

Estructura de monorepo con Nx como build system:

```
apps/
├── web/               # App principal (Next.js App Router)
│   └── src/
│       ├── app/
│       │   └── (public)/  # Grupos de rutas públicas
│       ├── components/    # Componentes locales de la app
│       │   ├── atoms/
│       │   ├── molecules/
│       │   └── organisms/
│       ├── lib/           # Clientes externos, configuración
│       ├── services/      # Llamadas a APIs / AWS Amplify
│       └── i18n/          # Internacionalización
├── admin/             # App de administración (Next.js)
└── web-e2e/           # Tests E2E con Playwright

packages/
├── ui/                # Design system compartido
│   └── src/
│       ├── atoms/
│       ├── molecules/
│       └── organisms/
├── core/              # Lógica de negocio compartida (auth, etc.)
│   └── src/
│       └── [domain]/
│           └── services/
├── utils/             # Utilidades compartidas
└── email-templates/   # Templates de email (React Email)
```

**Regla de dependencias:**

- `apps/*` pueden importar de `packages/*`, nunca al revés
- Componentes UI (`packages/ui`) no contienen lógica de negocio
- Servicios en `apps/*/src/services/` o `packages/core/` — nunca en componentes
- Server Components por defecto; Client Components solo cuando hay interactividad
- Fetch en Server Components o services, no en `useEffect` salvo casos justificados

### Frontend Mobile — SwiftUI MVVM

```
Features/
├── [Feature]/
│   ├── Views/
│   │   └── FeatureView.swift
│   └── ViewModels/
│       └── FeatureViewModel.swift
│
Models/                    # SwiftData @Model compartidos
├── [Entity].swift
│
Services/                  # Capa de red con Alamofire
├── APIClient.swift        # Configuración base (baseURL, headers, interceptors)
└── [Feature]Service.swift # Endpoints por dominio
│
Repositories/              # Acceso a SwiftData
└── [Entity]Repository.swift
│
Shared/
├── Components/            # Views reutilizables
├── Extensions/            # Extensions de tipos nativos
└── Utils/                 # Helpers, formatters, constantes
```

**Regla de dependencias:**

```
View → ViewModel → Service / Repository → Model
```

- `View`: solo layout, bindings y navegación. Sin lógica de negocio.
- `ViewModel`: estado de UI, validaciones, coordinación entre Service y Repository. Marcado con `@Observable`.
- `Service`: requests Alamofire, decode de respuesta a tipos `Decodable`. Sin estado.
- `Repository`: operaciones CRUD sobre SwiftData. Sin lógica de red.
- `Model`: structs `Decodable` para red y clases `@Model` para persistencia. Sin comportamiento.

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

- Tipos, structs, clases, enums y protocolos: `PascalCase`
- Variables, funciones y parámetros: `camelCase`
- Constantes de módulo: `camelCase` (convención Swift — no `UPPER_SNAKE_CASE`)
- Archivos: mismo nombre que el tipo que contienen — `UserViewModel.swift`
- ViewModels: sufijo `ViewModel` — `LoginViewModel`
- Services: sufijo `Service` — `AuthService`
- Repositories: sufijo `Repository` — `UserRepository`
- Protocolos de comportamiento: sufijo `-ing` o `-able` — `Authenticating`, `Cacheable`

### Function Guidelines

- Componentes: responsabilidad única, sin lógica de negocio mezclada con UI
- Funciones puras cuando sea posible
- Max líneas por función: 50
- Props de componentes tipadas con interface explícita, nunca con `any`

### Error Handling

**TypeScript / Web:**

Patrón `Result<T, E>` para errores esperados en servicios y use cases:

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E }
```

- Servicios retornan `Result<T, AppError>` — nunca lanzan excepciones para errores de negocio
- Errores inesperados (red, crashes) se propagan como excepciones y son capturados por Error Boundaries
- Error Boundaries en capas superiores del árbol de componentes para evitar duplicación de manejo de UI
- Errores esperados (validación, 404, conflictos) se tipan con discriminated unions — no `string` ni `any`

```ts
type AppError =
  | { type: 'validation'; fields: Record<string, string> }
  | { type: 'not_found'; resource: string }
  | { type: 'unauthorized' }
  | { type: 'unexpected'; cause: unknown }
```

**Swift:**

- `Result<T, Error>` con `async throws` para operaciones de red en Services
- `do/catch` en ViewModels — nunca en Views
- Errores de dominio tipados como `enum` conformando `Error`

```swift
enum AuthError: Error {
    case invalidCredentials
    case sessionExpired
    case networkUnavailable(underlying: Error)
}
```

### Documentación

- Documentar todas las funciones, estructuras y definiciones.
- Los atributos o propiedades solo se documentan si su función no es clara.
- Agregar documentación inline cuando el proceso o algoritmo es complejo.
- Salvo que el proyecto indique lo contrario.

---

## 5. Testing Standards

### Estrategia

Prioridad de pruebas: Integración, Unidad, E2E. Justificación: las pruebas de integración cubren también las pruebas de unidad. E2E requiere un escenario complejo y puede gastar más tokens en el proceso de pruebas.

**Required:**

- 90%
- Happy path + al menos un error path por función crítica
- Tests unitarios sin dependencias externas reales
- Los mocks para enmascarar consultas en bases de datos y en consultas de APIs

**Herramientas:**

- Jest + Testing Library para Web
- XCTest para iOS (unit tests de ViewModels y Services)

### Qué se testea siempre

- Lógica de negocio (hooks/utils en Web, ViewModels en iOS)
- Formularios con validación
- Services de red: respuesta correcta y manejo de errores (con mock de Alamofire)
- Repositories: operaciones CRUD sobre SwiftData en memoria

### Qué NO se testea

- Estilos visuales ni layout de SwiftUI
- Librerías de terceros (Alamofire, CocoaPods deps)
- Views directamente (se testea el ViewModel que las alimenta)

---

## 6. Security Standards

- Nunca secretos en código: usar variables de entorno (`.env.local`, nunca commiteado)
- Validar input del usuario siempre en el servidor, no solo en el cliente
- Sanitizar output renderizado dinámicamente (prevención XSS)
- Autenticación verificada en Server Components / middleware de Next.js
- La autenticación depende del stack definido por cada proyecto. Se da prioridad al uso de AWS Amplify Gen 2.
- Dependencias auditadas: `npm audit` (Web) y revisión de `Podfile.lock` antes de releases
- Credenciales iOS en Keychain, nunca en UserDefaults ni en código
- Headers de autenticación inyectados en `RequestInterceptor` de Alamofire, no en cada llamada
- SSL pinning en endpoints sensibles si el proyecto lo requiere

---

## 7. Non-Negotiables

Estas reglas NO se negocian NUNCA:

1. **TypeScript strict mode activo** — Sin `any` sin justificación documentada
2. **Conventional Commits** — Todo commit sigue el formato `type(scope): descripción`
3. **Sin secrets en código** — Variables de entorno siempre, sin excepción
4. **Tests antes de merge** — Las pruebas de lógica de negocio deben pasar
5. **Validación en el servidor** — Nunca confiar solo en validación del cliente
6. **Documentar funciones y estructuras** — Toda función/struct pública tiene doc

---

> **Nota:** Este documento aplica a todos los proyectos salvo que el `CLAUDE.md` del proyecto indique explícitamente lo contrario.
