# Preferencias Personales de Luis Li

## Identidad

- **Email:** <luislicardo1307@gmail.com>
- **Rol:** arquitecto de software, desarrollador de aplicaciones web, desarrollador de aplicaciones móviles
- **Stack principal:** frontend (React, NextJS, SwiftUI)

---

## Estilo de comunicación

- Respuestas cortas y directas. Deja un resumen al final de las tareas que ejecutaste.
- No usar emojis a menos que yo los pida explícitamente.
- Idioma: español por defecto, inglés en el código y en la documentación.
- Si algo es ambiguo, pregunta antes de implementar.

---

## Estilo de código

### General

- Documentar todas las funciones, estructuras y definiciones, salvo que el proyecto indique lo contrario. Los atributos o propiedades solo se documentan si su función no es clara.
- Agregar documentación inline cuando el proceso o el algoritmo es complejo, o en el código no es explícito.
- No agregar manejo de errores, validaciones ni fallbacks para escenarios imposibles.
- No introducir abstracciones prematuras. Tres líneas similares son mejor que una abstracción innecesaria.
- No agregar features extra ni refactors que no fueron pedidos.
- No usar feature flags ni shims de compatibilidad hacia atrás sin razón explícita.

### Naming y estructura

- Nombres descriptivos y explícitos. El nombre debe decir qué hace, sin necesitar un comentario.
- Preferir editar archivos existentes antes de crear nuevos.
- Nunca crear archivos `.md` de documentación o README a menos que yo lo pida.

### Seguridad

- Nunca introducir vulnerabilidades: SQL injection, XSS, command injection, secretos en código.
- Validar solo en los límites del sistema (input del usuario, APIs externas).

---

## Git y PRs

- Commits en inglés, mensaje conciso enfocado en el _por qué_, no en el _qué_.
- Usa Conventional Commits como estructura para los mensajes de los commits.
- Nunca hacer `git push --force` a main/master sin confirmación explícita.
- Nunca saltarse hooks (`--no-verify`) sin que yo lo pida.
- Preguntar antes de cualquier operación destructiva: `reset --hard`, `branch -D`, `rm -rf`.
- No hacer commit de archivos sensibles: `.env`, credenciales, tokens.

---

## Flujo de trabajo

- Para tareas exploratorias ("¿qué podríamos hacer con X?"), responder en 2-3 oraciones con una recomendación y el trade-off principal. No implementar hasta que yo confirme.
- Usar herramientas paralelas cuando las llamadas son independientes entre sí.
- Acciones que afectan sistemas compartidos (push, PRs, mensajes) requieren confirmación previa.
- Para cambios en UI/frontend, probar el flujo dorado antes de reportar como completo.

---

## Preferencias de herramientas

- Preferir `Read`/`Edit`/`Write` sobre comandos Bash cuando aplique.
- No usar `cat`, `head`, `tail`, `sed`, `awk` salvo que no haya alternativa.
- Usar rutas absolutas siempre.

---

## Lo que NO quiero

- Explicaciones de _qué_ hace el código (los nombres y los comentarios lo dicen).
- Referencia al número de issue, PR o tarea dentro del código.
- Código a medias ni implementaciones incompletas.
- Cambios en `git config`.
