# Plantilla — Flujos de Usuario (SDD)

> **Plantilla reutilizable** para levantar flujos de usuario con un cliente antes del desarrollo de software. Usa los skills `user-flow-validator` y `user-flow-transformer` para validar y transformar este documento a formato Speckit.

---

# Flujos de usuario — [Nombre de la aplicación]

> **¿Qué es este documento?**
> Aquí definimos, junto al cliente, lo que los usuarios hacen dentro de la aplicación para lograr cosas concretas. Cada flujo describe una tarea de principio a fin. No necesitamos saber cómo se programa — solo qué hace el usuario y qué espera recibir.

> **¿Cómo se usa?**
> En cada sesión se define un flujo nuevo copiando el bloque de plantilla. Las preguntas en cada sección guían la conversación. Una vez completo, un agente de IA formaliza el documento para el proceso de especificación técnica.

---

## Flujo: [Nombre del flujo]

> *Ejemplos: "Registrar una venta", "Aprobar una solicitud"*

**ID:** `UF-001`

> *Formato UF-NNN. Incrementar por cada flujo nuevo.*

**Estado:** `Borrador`

> *Valores posibles: Borrador / Revisado / Aprobado*

### ¿Qué quiere lograr el negocio con este flujo?

> *¿Por qué existe este flujo? ¿Qué problema resuelve o qué oportunidad aprovecha?*

> [Una o dos frases. Ej: "Que los vendedores registren ventas desde el celular sin tener que llamar a nadie, para que el inventario se actualice en tiempo real."]

### ¿Quién hace qué en este flujo?

> *¿Qué tipo de persona lo inicia? ¿Hay otras personas que participan en algún momento?*

> [Describe libremente quiénes participan. Ej: "Lo inicia el vendedor desde su celular. En algún punto el supervisor tiene que dar el visto bueno antes de que la venta quede confirmada."]

### ¿Qué necesita el usuario para empezar?

> *¿Hay algo que debe estar listo antes? Si no hay nada especial, omite esta sección.*

- [Ej: El usuario debe haber iniciado sesión]
- [Ej: Debe existir al menos un producto registrado]

### ¿Desde dónde empieza?

> *¿En qué pantalla está el usuario y qué acción activa el flujo?*

> [Ej: "El vendedor está en la pantalla de inicio y presiona el botón 'Nueva venta'."]

### Pasos

> *Describe paso a paso lo que ocurre, en el orden en que sucede.*

> **Cómo escribir los pasos:**
> - Escribe cada paso como una oración natural: quién lo hace, dónde y qué resultado produce.
> - Si el paso muestra información al usuario, menciona qué datos aparecen en pantalla.
> - Si el paso tiene un formulario, menciona qué campos debe llenar el usuario.
> - Si un paso no puede ocurrir hasta que otra persona haga algo primero, indícalo con `↳ Espera que [persona] complete el paso N.`

1. [Ej: El vendedor abre la pantalla de ventas y presiona "Nueva venta".]
2. [Ej: El sistema muestra un formulario con los campos: producto, cantidad y precio unitario.]
3. [Ej: El vendedor llena el formulario y presiona "Enviar para aprobación".]
4. [Ej: El supervisor recibe una notificación con el resumen de la venta (producto, cantidad, total) y la aprueba.]
   - `↳ Espera que el supervisor complete este paso antes de continuar.`
5. [Ej: El vendedor ve la venta confirmada en su historial con el estado "Aprobada".]

### ¿Qué obtiene el usuario al terminar?

> *¿Qué queda guardado, enviado o disponible cuando todo sale bien?*

> [Ej: "La venta queda registrada con estado 'Aprobada', el inventario se descuenta y el cliente recibe un correo de confirmación."]

### ¿Qué pasa si algo sale diferente?

> *Solo si hay una variación importante que también genera valor al negocio.*

> **¿Cuándo incluir una variación?** Solo cuando el camino diferente también produce algo útil para el negocio, no cuando simplemente hay un error.

**Variación — [Nombre]:**

> *Duplicar este bloque por cada variación adicional.*

- **¿Cuándo ocurre?** [Ej: "Cuando el stock del producto es insuficiente."]
- **¿Qué cambia?** [Pasos que son distintos al flujo normal.]
- **¿Cómo termina?** [Resultado final de esta variación.]

### Notas del cliente

> *Reglas o restricciones del negocio que no quedaron claras en los pasos.*

- [Ej: "Solo los vendedores con más de 3 meses pueden registrar ventas superiores a $5.000.000."]

---

## Flujo: [Nombre del siguiente flujo]

> *Copia el bloque completo del flujo anterior y llénalo.*

**ID:** `UF-002`
**Estado:** `Borrador`

---

## Glosario de términos del negocio

> *¿Hay palabras propias de este negocio que puedan ser confusas para alguien de afuera?*

> **¿Por qué es importante?** Las palabras del cliente a veces significan cosas distintas en otros contextos. Este glosario asegura que el equipo de desarrollo entienda exactamente lo mismo que el cliente.

| Término     | Qué significa en este negocio           |
|-------------|-----------------------------------------|
| [Término 1] | [Definición en el contexto del cliente] |
| [Término 2] | [Definición en el contexto del cliente] |
