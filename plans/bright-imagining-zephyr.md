# Islas en la barra superior (pieza B)

## Contexto

Tercera pieza del rediseño de escritorio, siguiendo el panel izquierdo (A,
ya guardado). Cada widget de la barra pasa a ser su propia "isla" — un pill
redondeado independiente con separación del resto — en vez de compartir un
fondo continuo por sección, siguiendo la referencia visual de Moonveil ya
mostrada. También se reorganiza qué widget va en cada zona y se sacan varios
que ya no tienen lugar en la barra.

Confirmado con el usuario:
- `omarchy.power`, `omarchy.menu` y `omarchy.workspaces` se sacan de la
  barra por completo (menu/workspaces quedan redundantes con el panel
  izquierdo nuevo — pieza A).
- El widget de CPU/RAM/GPU/temp (pieza C) todavía no existe — la isla
  izquierda queda solo con el reloj por ahora; el widget de stats se agrega
  después sin tocar este trabajo.
- `omarchy.weather` y `omarchy.media` se sacan ya de la barra (van a
  widgets de escritorio, pieza D, todavía sin construir) — quedan sin
  mostrarse en ningún lado hasta esa pieza.

Layout final de `shell.json`:
- **left:** `omarchy.clock` (más el widget de stats cuando exista — pieza C)
- **center:** `omarchy.active-window`, `omarchy.system-update`, `omarchy.agents`
- **right:** `omarchy.indicators`, `omarchy.tray`, `omarchy.keyboard-layout`,
  `omarchy.bluetooth`, `omarchy.network`, `omarchy.monitor`
- `omarchy.spacer` también se saca — su función (separar visualmente) queda
  cubierta por el gap entre islas.
- `centerAnchor` pasa de `"omarchy.clock"` (ya no está en center) a `""`
  (sin anclaje, el grupo se centra como bloque).

## Investigación (agente Explore sobre
`/usr/share/omarchy/shell/plugins/bar/Bar.qml`, 1823 líneas)

Todo el renderizado de widgets pasa por un único punto: `Row`/`Column`
(`horizontalModuleList`/`verticalModuleList`, ~1487-1521, `spacing: 0` hoy)
→ `Repeater` → **`component ModuleSlot: Item`** (~1524-1572) → `Loader`
(`registryLoader` para widgets normales) → el widget en sí. Es el mismo
componente para las 3 secciones, ambas orientaciones (horizontal/vertical)
y el modo `centerAnchor` — **un solo edit cubre todo**, sin tocar el resto
del archivo (drag/reorder, tooltips, popouts, IPC — todo eso queda
intacto).

`ModuleSlot` hoy no tiene fondo propio: solo un `BorderSurface` visible
solo durante drag (ghost outline, 1576-1584) y un indicador de popout
abierto (1622-1648, un subrayado, no un fondo). El ancho/alto del slot está
atado 1:1 al `implicitWidth/implicitHeight` del widget, sin padding.

`omarchy.spacer`/`omarchy.indicators` no están hardcodeados en ningún lado
de Bar.qml — son widgets registrados normales. Confirmado: mover/sacar
widgets de sección es un cambio de `shell.json` únicamente, sin tocar el
plugin, EXCEPTO el estilo de isla en sí (eso sí requiere clonar y editar
`ModuleSlot`).

## Cambios

### 1. Clonar y editar `omarchy.bar`

```bash
omarchy plugin clone omarchy.bar   # -> ~/.config/omarchy/plugins/luisli.bar/
```

En el `Bar.qml` del clon:

- **`ModuleSlot`**: agregar un `BorderSurface` de fondo (mismo componente
  reusable que ya usan popups/notificaciones — `Ui/BorderSurface.qml`),
  dimensionado al slot con padding nuevo (ajustar
  `implicitWidth`/`implicitHeight` del slot + `anchors.margins` en los
  `Loader`s para dejar aire alrededor del widget). `radius: Style.cornerRadius`
  — a diferencia de los paneles custom (volume-edge/left-edge/OSD), que usan
  un radio fijo propio, esta ES la barra real del sistema, así que conviene
  que seguir sincronizada con el rounding global de Hyprland (13 hoy) como
  el resto de los popups compartidos.
- **Color por zona**: `border.color` (y un fill sutil) elegido por
  `slot.region` ("left"/"center"/"right") vía un mapa de 3 colores fijo —
  mismo patrón de "un color por sección" que ya usamos en `shell.toml`
  para popups/notifications/menu y en `luisli.left-edge` para los
  monitores. Paleta a definir (propongo reusar las mismas 3 que ya existen
  en el sistema para que todo el desktop comparta un lenguaje de color:
  azul/magenta/cyan de `shell.toml`, o los mismos que ya eligió el usuario
  en otro lado — confirmar color exacto durante la implementación con
  captura, como se hizo en todo lo demás).
- **Gap entre islas**: `horizontalModuleList`/`verticalModuleList` pasan de
  `spacing: 0` a un valor pequeño (ej. `Style.space(6)`) para que las islas
  no queden pegadas entre sí.

### 2. `shell.json`

Reescribir `bar.layout.{left,center,right}` según la lista de arriba,
sacar `omarchy.menu`, `omarchy.workspaces`, `omarchy.weather`,
`omarchy.media`, `omarchy.spacer`, `omarchy.power` de todas las secciones,
y `centerAnchor: ""`.

## Fuera de alcance

- Widget de CPU/RAM/GPU/temp (pieza C) — la isla izquierda queda con un
  solo widget (reloj) hasta esa pieza.
- Widgets de escritorio para weather/media/calendario (pieza D).
- Mover al repo `omarchy-config` — recién cuando esté validado (mismo
  flujo de siempre).

## Verificación

1. Enable del clon, `omarchy restart shell` (cambios de `ModuleSlot`
   afectan geometría — necesita restart completo, no alcanza con guardar,
   confirmado varias veces en esta sesión).
2. `journalctl --user _PID=<pid>` sin errores QML.
3. Screenshot de la barra en ambos monitores — confirmar visualmente islas
   separadas, colores por zona, sin regresión en widgets que ya
   funcionaban (tray, indicators, etc.).
4. Confirmar que los widgets sacados (menu, workspaces, weather, media,
   spacer, power) ya no aparecen en ningún lado de la barra.
5. Interacción real (clicks, popouts) queda para que el usuario confirme —
   no puedo probar eso sin mouse.
