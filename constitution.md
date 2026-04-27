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
- [COMPLETAR: herramienta/estrategia de auth — NextAuth, Clerk, etc.]
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
7. [COMPLETAR: agrega tus reglas innegociables]

---

> **Nota:** Este documento aplica a todos los proyectos salvo que el `CLAUDE.md` del proyecto indique explícitamente lo contrario.
