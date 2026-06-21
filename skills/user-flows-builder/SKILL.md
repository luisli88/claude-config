---
name: user-flows-builder
description: >
  Completa el documento de flujos de usuario (SDD) para un proyecto nuevo o parcialmente documentado.
  Carga la plantilla de Bear, llena los flujos con IA usando la descripción que el usuario provee, y guarda
  el resultado como nota en Bear bajo el tag proyectos/[nombre_proyecto].
  Usar cuando el usuario quiere levantar requerimientos con un cliente, generar flujos de usuario para una nueva aplicación,
  completar un documento de flujos parcial, o describir una app y necesita un borrador de flujos de usuario.
  Invocar también cuando se mencionan flujos de usuario, levantamiento de requerimientos, SDD, o user flows.
argument-hint: "Descripción de la app [ruta-o-titulo-de-nota-parcial]"
user-invocable: true
---

## Argumentos

```text
$ARGUMENTS
```

El primer argumento es siempre la **descripción o instrucciones** para completar los flujos (obligatorio).
El segundo argumento, si existe, es una **referencia a un documento parcial**: puede ser:
- Título de una nota de Bear (texto sin extensión)
- Ruta absoluta a un archivo `.md` en el sistema de archivos

Si no se proporcionó descripción, pídela antes de continuar.

---

# Skill: User Flows Builder

## Propósito

Este skill genera o completa un documento de flujos de usuario basado en la plantilla estándar de Bear
(`Plantilla — Flujos de Usuario (SDD)`, tag `#templates/sdd`). El objetivo es producir un borrador
útil como base para el levantamiento de requerimientos con el cliente: flujos comunes entre muchas
aplicaciones (autenticación, perfil, onboarding, etc.) se completan automáticamente; los flujos
específicos del negocio se rellenan con la información que el usuario proveyó.

---

## Paso 1 — Obtener el contenido base

### Si NO hay documento parcial (proyecto nuevo):

Carga la plantilla de Bear usando la herramienta `mcp__Bear__get_note`:
- Busca por título: `Plantilla — Flujos de Usuario (SDD)`
- Si no la encuentra, busca con `mcp__Bear__search_notes` usando `#templates/sdd`
- Extrae el contenido de la plantilla para usarlo como estructura base

### Si HAY documento parcial:

**Detecta el tipo de referencia:**

- Si la referencia contiene `/` o termina en `.md` → es una ruta de archivo. Léela con la herramienta `Read`.
- Si no → es un título de nota de Bear. Búscala con `mcp__Bear__get_note` por título.
- Si el documento parcial no se encuentra, comunícalo y detente.

El documento parcial reemplaza a la plantilla como estructura base. Respeta todo lo que ya está escrito.
Solo completa lo que está vacío, usa placeholders `[...]`, o está marcado como incompleto.

---

## Paso 2 — Identificar el nombre del proyecto

**Prioridad para determinar el nombre del proyecto:**

1. Si el documento parcial existe y su título sigue el patrón `Flujos de usuario — [Nombre]`, extrae el nombre de ahí.
2. Si la descripción menciona explícitamente el nombre del proyecto o la app, úsalo.
3. Si no está claro, infiere un nombre corto y descriptivo a partir del contexto.

El nombre del proyecto se usará como tag en Bear: `proyectos/[nombre_proyecto]` (sin espacios, en minúsculas con guiones si aplica).

---

## Paso 3 — Generar los flujos

Completa el documento usando la descripción/instrucciones del usuario. Guíate por estos principios:

### Qué flujos incluir

Incluye siempre los flujos **comunes** que apliquen al tipo de app descrita:
- Registro e inicio de sesión (si la app tiene usuarios)
- Recuperación de contraseña (si aplica)
- Onboarding o configuración inicial (si aplica)
- Cierre de sesión

Agrega flujos **específicos del negocio** a partir de la descripción. Si la descripción menciona
entidades, acciones o procesos clave, convierte cada uno en un flujo o en una variación.

### Cómo completar cada sección

- **Objetivo del negocio**: explica el valor concreto que este flujo genera para el negocio o el usuario.
- **Actores**: identifica quién inicia el flujo y quién más participa.
- **Precondiciones**: lista solo las que son realmente necesarias (omite la sección si no hay ninguna).
- **Punto de entrada**: describe dónde está el usuario y qué acción dispara el flujo.
- **Pasos**: escribe oraciones naturales. Menciona qué datos ve el usuario en cada pantalla y qué debe llenar en formularios. Usa `↳ Espera que [persona] complete el paso N.` cuando hay dependencia entre actores.
- **Resultado esperado**: describe qué queda registrado, enviado o disponible al terminar.
- **Variaciones**: incluye solo las que producen un resultado de negocio distinto (no errores triviales).
- **Notas del cliente**: deja esta sección vacía o con un placeholder si no hay información específica.

### IDs de flujo

Asigna IDs correlativos: `UF-001`, `UF-002`, etc. Si el documento parcial ya tiene IDs asignados,
continúa desde el último usado.

### Estado

Todos los flujos generados por este skill tienen estado `Borrador`.

### Glosario

Completa el glosario con términos del dominio de negocio mencionados en la descripción que puedan
ser ambiguos fuera de ese contexto. Si no hay términos especiales, deja el glosario vacío con la
estructura de tabla intacta.

---

## Paso 4 — Detectar y resolver ambigüedades

Antes de guardar, revisa el borrador generado e identifica:

- **Definiciones incompletas**: flujos donde la descripción no da suficiente información para completar una sección.
- **Contradicciones**: dos partes del documento o de las instrucciones que se contradicen.
- **Ambigüedades bloqueantes**: donde hay dos interpretaciones igualmente válidas y la elección afecta el flujo.

Si encuentras alguna, **detente y pregunta al usuario** antes de guardar. Agrupa todas las preguntas
en un solo mensaje para no interrumpir más de una vez. Si las ambigüedades son menores o de estilo,
elige la opción más razonable y documéntala en "Notas del cliente" del flujo correspondiente.

---

## Paso 5 — Guardar en Bear

Una vez confirmado el contenido (o si no había ambigüedades bloqueantes), crea la nota en Bear con
`mcp__Bear__create_note`:

- **Título:** `Flujos de usuario — [Nombre del proyecto]`
- **Contenido:** el documento completo en Markdown
- **Tags:** `["proyectos/[nombre_proyecto]"]`

Confirma al usuario el título de la nota creada y el tag bajo el que fue guardada.

---

## Formato del documento generado

Respeta exactamente la estructura de la plantilla. No inventes secciones nuevas ni elimines las existentes.
El documento debe poder usarse directamente como input para `/user-flow-validator`.

Encabezado del documento (reemplaza los placeholders de la plantilla):

```markdown
# Flujos de usuario — [Nombre del proyecto]

> **¿Qué es este documento?**
> ...

> **¿Cómo se usa?**
> ...
```

Cada flujo sigue la estructura de la plantilla sin omitir ninguna sección (salvo "Precondiciones"
cuando genuinamente no hay ninguna).
