---
name: new-project
description: Decision skill for new web projects — gathers requirements, applies architecture decision criteria, and routes to the appropriate scaffolding skill (amplify-nextjs or amplify-monorepo).
argument-hint: "Project name and a brief description of what it does"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Parse the user input and extract:
- **Project name**: kebab-case identifier. Required.
- **Description**: What the project does in one sentence. Required.

If either is missing, ask for both before continuing. Do not start Phase 1 until you have both.

---

## PHASE 1 — Requirements Interview

Ask all the questions below in **a single message**, numbered. Do not proceed to Phase 2 until all answers are received.

> "Para elegir la arquitectura correcta necesito entender el proyecto. Responde estas preguntas:"

1. **Frontends** — ¿El proyecto necesita dos aplicaciones separadas: una pública (usuarios finales) y una de administración con su propio dominio? _(sí / no)_

2. **Roles de usuario** — ¿Cuántos roles distintos necesita el proyecto y tienen reglas de acceso a datos radicalmente diferentes a nivel de base de datos (no solo visibilidad de UI)? _(ejemplo: "solo usuarios autenticados", "Admin + User", "Admin + Operator + User")_

3. **Email transaccional** — ¿El proyecto necesita enviar emails HTML con diseño corporativo: bienvenidas, recibos, notificaciones? _(sí / no)_

4. **Integraciones SaaS** — ¿El proyecto necesita recibir webhooks de servicios externos como pasarelas de pago, SMS, CRM? _(sí / no — si sí, ¿cuáles?)_

5. **Tamaño del equipo** — ¿Cuántos desarrolladores trabajarán en el proyecto? _(1–4 / 5–10 / 10+)_

6. **Backend existente** — ¿Existe un backend previo al que el frontend deba conectarse (Spring Boot, Rails, Django, Laravel, API de terceros)? _(sí / no)_

7. **Complejidad de datos** — ¿El modelo de datos requiere consultas relacionales complejas con múltiples JOINs entre entidades? _(sí / no)_

8. **Procesos largos** — ¿Hay tareas en segundo plano que corran por más de 15 minutos (procesamiento de video, generación masiva de reportes)? _(sí / no)_

---

## PHASE 2 — Architecture Decision

### Hard blockers — ninguna arquitectura aplica

Evalúa estos antes de aplicar el resto de la matriz. Si alguno es verdadero, reporta el bloqueador y sugiere la alternativa. No hagas scaffolding.

| Condición | Motivo | Alternativa |
|---|---|---|
| Backend existente (Spring Boot, Rails, etc.) | Amplify es un backend completo. Agregar uno en paralelo crea drift. | Next.js standalone + Server Actions llamando al API existente |
| Consultas relacionales complejas con muchos JOINs | DynamoDB está optimizado para patrones de acceso, no JOINs ad-hoc. | Nest.js + PostgreSQL + tRPC o REST |
| Procesos que corren más de 15 minutos | Lambda tiene ese límite hardware. | ECS/EC2 para esas cargas; Lambda para el resto |

### Decidir: amplify-monorepo

Usa `amplify-monorepo` si **dos o más** de los siguientes son verdaderos:

- [ ] Se necesitan dos frontends separados (web pública + admin)
- [ ] Tres o más roles con reglas de acceso radicalmente distintas a nivel de schema
- [ ] Se requieren emails HTML transaccionales con templates de diseño (Maizzle)
- [ ] Una o más integraciones con webhooks de SaaS externos
- [ ] El equipo es de 5 o más desarrolladores

### Decidir: amplify-nextjs

Usa `amplify-nextjs` si **todos** los siguientes son verdaderos:

- [ ] Un solo frontend
- [ ] Cero a dos roles (acceso público, usuario autenticado, o admin+user con reglas simples de grupo)
- [ ] Sin webhooks de SaaS externos (o como máximo un formulario de contacto)
- [ ] Sin emails HTML transaccionales (SES en texto plano es suficiente)
- [ ] Equipo de 1–4 desarrolladores

### Auth pattern (solo para amplify-nextjs)

Determina el patrón de auth basado en los roles:

| Respuesta de roles | Patrón |
|---|---|
| Sin login, solo formulario de contacto o contenido público | `guest` |
| Usuarios autenticados con email/contraseña, datos propios | `cognito` |
| Admin + User con acceso a datos diferente a nivel de schema | `groups` |

---

## PHASE 3 — Decision Summary

Antes de invocar el skill, imprime el resumen de decisión con este formato exacto:

```
## Decisión de arquitectura: [amplify-nextjs | amplify-monorepo]

**Proyecto:** {project-name} — {description}
**Motivo:** {1–3 oraciones explicando por qué esta arquitectura}

**Factores decisivos:**
- {factor 1}
- {factor 2}
- ...

**Parámetros para el scaffolding:**
- Auth: {guest | cognito | groups}       ← solo si es amplify-nextjs
- Features: {contact-form | storage}     ← solo si aplica
```

---

## PHASE 4 — Skill Invocation

Después del summary, invoca el skill correspondiente con los argumentos correctos:

- **amplify-nextjs**: `{project-name}, {description}, auth: {pattern}[, features: {list}]`
- **amplify-monorepo**: `{project-name}, {description}`
