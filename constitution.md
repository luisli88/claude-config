# Development Constitution

**Author:** Luis Ricardo Ruiz
**Version:** 2.0
**Last Updated:** 2026-06-28

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

### Backend — Dos stacks en transición activa

> **Nota de versión (2.0):** a partir de esta versión, el backend tiene **dos stacks válidos en convivencia**, no uno solo por defecto. Esto refleja la realidad operativa del autor: existen 3 proyectos en producción sobre Amplify Gen 2 que se seguirán manteniendo, y Amplify Gen 2 sigue siendo una opción válida para proyectos _nuevos_ mientras la plantilla IaC propia madura. El detalle completo de la decisión de construir la plantilla vive en el proyecto "Plantilla de Montaje Rápido de Proyectos (IaC + SDD)" — ver `00-vision-y-alcance.md` y `02-arquitectura-por-capas.md` de ese proyecto.

**Criterio de selección para proyectos nuevos:**

| Situación                                                                                               | Stack a usar                                                               |
| ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Proyecto nuevo, MVP rápido, antes de que la plantilla IaC alcance el criterio de transición (ver abajo) | **AWS Amplify Gen 2** — sigue siendo válido y probado para este caso       |
| Proyecto nuevo, después de que la plantilla IaC alcance el criterio de transición                       | **Plantilla IaC propia (AWS CDK)** — pasa a ser el default                 |
| Mantenimiento de los 3 proyectos existentes en producción                                               | **AWS Amplify Gen 2** — se mantiene indefinidamente, sin presión de migrar |
| El proyecto indica explícitamente otro stack                                                            | Ese stack tiene precedencia sobre ambos defaults                           |

**Criterio de transición (cuándo la plantilla IaC pasa a ser el default para proyectos nuevos):**

- **Disparador práctico:** el escenario **E1 (implementación completa)** de la plantilla IaC queda validado de punta a punta en el repositorio — es decir, se puede montar un MVP completo con la misma confianza que hoy da Amplify Gen 2. Ver `01-roadmap-general.md`, Fase B en adelante, criterio de cierre ADR-07.
- **Madurez completa del proyecto plantilla** (distinto del disparador de arriba): requiere los tres escenarios resueltos — E1, E2 (híbrida con integración) y E3 (infraestructura ajena con restricciones del cliente) — porque la plantilla se diseñó para cubrir los tres desde el inicio, y avanzan en paralelo. E1 decide cuándo _dejar de iniciar_ proyectos nuevos en Amplify; E1+E2+E3 deciden cuándo la plantilla se considera _completa_ como proyecto.

Hasta que el criterio de transición se cumpla, esta sección de la constitución debe mantenerse con ambos stacks documentados — no se elimina la sección de Amplify Gen 2 prematuramente.

#### Backend — AWS Amplify Gen 2

**Required (mientras sea el stack elegido, según el criterio de selección de arriba):**

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
│       ├── email/               # Helpers de envío de email (SES)
│       ├── payment/              # Lógica de pagos compartida
│       └── [domain]/              # Tipos y helpers por dominio
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

**Limitación conocida (motivo de la transición):** la integración de Amplify Gen 2 con el ecosistema de Next.js introduce fragilidad de CI/CD — cambios en dependencias suelen requerir ajustes en cascada para que el pipeline no falle. Esta limitación es la motivación original del proyecto de plantilla IaC propia.

#### Backend — Plantilla IaC propia (AWS CDK)

**Required (mientras sea el stack elegido, según el criterio de selección de arriba):**

- AWS CDK como única herramienta de Infraestructura como Código — no se usa Terraform ni alternativas (ver ADR-03 de la plantilla)
- La infraestructura se activa importando y parametrizando **Constructs de la librería reutilizable** de la plantilla (`DataLayerRelational`, `DataLayerNonRelational`, `IntegrationLayerLambda`, `IntegrationLayerContainer`, `PresentationLayerBFF`, `PresentationLayerStaticSite`) — no se escribe infraestructura nueva para responsabilidades que la librería ya cubre
- TypeScript en todo el código de Constructs, funciones Lambda, y composición de stacks
- Cognito para autenticación, cuando el escenario del proyecto lo requiera
- Funciones organizadas por dominio, siguiendo el mismo patrón `resource.ts` + `handler.ts` que ya usa Amplify Gen 2 (se conserva esa convención porque funciona bien y no depende de Amplify en sí)

**Los tres escenarios de trabajo (ver `00-vision-y-alcance.md`, sección 1.0.1):**

- **E1 — Implementación completa:** el autor controla toda la infraestructura (greenfield en AWS). Se activan los Constructs necesarios de las tres capas (Datos, Integración, Presentación).
- **E2 — Híbrida con integración:** parte de la infraestructura la gestiona la plantilla (AWS); parte es un sistema externo ajeno, al que se accede vía API/webhooks desde el Construct de Integración — nunca describiendo el sistema externo como recurso de infraestructura propio.
- **E3 — Infraestructura ajena con restricciones del cliente:** no se activa ningún Construct de infraestructura de la plantilla. El entorno se emula vía Docker Compose, replicando las restricciones de software que el cliente impone (versiones de runtime, librerías permitidas).

**Estructura de backend:**

```
infra/
├── app.ts                       # Composition root (instancia los Constructs activados)
├── constructs/                  # Constructs de la librería (versionados, reutilizables)
│   ├── data/
│   │   ├── DataLayerRelational.ts
│   │   └── DataLayerNonRelational.ts
│   ├── integration/
│   │   ├── IntegrationLayerLambda.ts
│   │   └── IntegrationLayerContainer.ts
│   └── presentation/
│       ├── PresentationLayerBFF.ts
│       └── PresentationLayerStaticSite.ts
├── functions/
│   ├── [domain]/
│   │   ├── resource.ts          # Definición de la Lambda (parámetros del Construct)
│   │   └── handler.ts           # Handler de la Lambda
│   ├── cognito-triggers/        # Triggers de Cognito (pre-signup, post-confirmation, etc.)
│   ├── notifications/           # Funciones de notificación por email
│   ├── services/                # Funciones de servicios de negocio
│   └── shared/                  # Utilidades compartidas entre funciones
│       ├── common/               # Tipos, validaciones, identidad, auditoría
│       ├── email/                 # Helpers de envío de email (SES)
│       ├── payment/                # Lógica de pagos compartida
│       └── [domain]/                # Tipos y helpers por dominio
└── docker/                      # Solo presente en escenario E3
    └── docker-compose.yml        # Emulación del entorno restringido del cliente
```

**Reglas:**

- `app.ts` es el único punto de composición — instancia los Constructs activados, no importa funciones entre sí directamente
- Lógica compartida entre funciones va en `functions/shared/`, nunca duplicada
- Autorización declarada a nivel del Construct de Presentación/Integración (ej. autorizador de API Gateway) — no en los handlers
- Variables de entorno inyectadas por CDK, nunca hardcodeadas
- Un `resource.ts` por función — define nombre, runtime y permisos IAM necesarios
- El contrato de salida de cada capa (outputs del stack) sigue lo definido en `02-arquitectura-por-capas.md` — nunca se inventa un formato de output distinto por proyecto

**Prohibited:**

- Lógica de negocio en `app.ts` (solo composición)
- Escribir infraestructura nueva para una responsabilidad que ya cubre un Construct de la librería — se extiende o parametriza el Construct existente, no se duplica su lógica
- Secrets en código — usar Parameter Store, Secrets Manager, o variables de entorno de CDK
- Usar Terraform o cualquier herramienta de IaC alternativa a CDK, salvo que un caso real documentado lo justifique (ver ADR-03 de la plantilla)
- Usarla todavía como default automático para un proyecto nuevo si el criterio de transición no se ha cumplido — en ese caso, Amplify Gen 2 sigue siendo la elección correcta salvo justificación explícita

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
│       ├── services/      # Llamadas a APIs / backend de la plantilla
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
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };
```

- Servicios retornan `Result<T, AppError>` — nunca lanzan excepciones para errores de negocio
- Errores inesperados (red, crashes) se propagan como excepciones y son capturados por Error Boundaries
- Error Boundaries en capas superiores del árbol de componentes para evitar duplicación de manejo de UI
- Errores esperados (validación, 404, conflictos) se tipan con discriminated unions — no `string` ni `any`

```ts
type AppError =
  | { type: "validation"; fields: Record<string, string> }
  | { type: "not_found"; resource: string }
  | { type: "unauthorized" }
  | { type: "unexpected"; cause: unknown };
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
- La autenticación depende del stack definido por cada proyecto, según el criterio de selección de la sección 2 (Amplify Gen 2 o la plantilla IaC propia). En ambos casos, la verificación de sesión sigue las mismas reglas de Server Components / middleware de Next.js
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

---

## Historial de versiones

| Versión | Fecha      | Cambio                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.0     | 2026-04-26 | Versión original, con AWS Amplify Gen 2 como único stack de backend por defecto.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| 2.0     | 2026-06-28 | Se agrega "Plantilla IaC propia (AWS CDK)" como segundo stack de backend válido, en **transición activa** con Amplify Gen 2 — no como reemplazo inmediato. Amplify Gen 2 sigue siendo el default para proyectos nuevos hasta que el escenario E1 de la plantilla quede validado (criterio de transición); se mantiene indefinidamente para los 3 proyectos en producción que dependen de él. Se documenta el criterio de transición y el criterio de madurez completa de la plantilla (E1+E2+E3). El resto del documento (Philosophy, Frontend Web, Frontend Mobile, Architecture Patterns, Code Quality, Testing, Non-Negotiables) se mantiene sin cambios respecto a la versión 1.0. |
