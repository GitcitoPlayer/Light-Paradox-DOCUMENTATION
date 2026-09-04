# 21 — Sistema: Day/Night Cycle · Migración UE 5.5.4 → 5.4.4
### Base Asset: [pendiente nombre del asset original — ver discrepancia de nombre abajo]
### Proyecto: Light Paradox · UE 5.4.4
### Fuente: Migración manual + inspección directa del asset original — sesión Light Paradox
### Fuente adicional (esta sesión): `Day_Night_Module.pdf` — documento original del programador (fuente vendor, no ESRPGv5 oficial)

---

## Contexto

Sistema de ciclo Día/Noche migrado manualmente desde un proyecto en UE 5.5.4
(el desarrollador original usó la versión incorrecta de motor). Se replicó
Blueprint por Blueprint, asset por asset, en Light Paradox (UE 5.4.4).
Servirá como base para disparar eventos de ciclos lunares y tipos de eclipse.

**Estado general al cierre de la sesión anterior:** el ciclo día/noche ya es
**funcional** — corre lógicamente y produce movimiento visual del sol y la
luna. Quedan pendientes: un misterio sin resolver sobre el mecanismo real
del original (ver Bug 5), problemas visuales conocidos a corregir, una
tarea de optimización de rendimiento reportada por el cliente, validación
del sistema replicado contra el documento original del programador (nuevo
esta sesión), y el diseño de fases lunares/eclipse vinculadas a Estados
(ver `22_SYSTEM_LunarPhases_EclipseStates.md`, nuevo archivo).

---

## ⚠️ Discrepancia de nombre detectada esta sesión — BP_DayNight vs BP_DayNightCycle

El documento original del programador (`Day_Night_Module.pdf`) nombra al
actor principal como **`BP_DayNightCycle`**. Toda la documentación de este
proyecto basada en inspección directa de la migración (este archivo)
usa **`BP_DayNight`**.

**No confirmado cuál es el nombre real actual en el proyecto Light Paradox.**

- **Prioridad para nombrar el asset TAL COMO EXISTE en Light Paradox:** la
  inspección directa del proyecto migrado tiene prioridad sobre el PDF,
  porque el PDF describe el proyecto original en 5.5.4, no la migración —
  pudo haber existido un rename intencional o no intencional durante el
  proceso manual de migración documentado en este archivo.
- **Acción pendiente (próxima sesión):** abrir el Content Browser y
  confirmar el nombre exacto de la clase Blueprint actual. Si el nombre
  real es `BP_DayNightCycle`, corregir este archivo y todas las
  referencias cruzadas (`16_SYSTEM_RuneBinding_WeaponCosmetic.md` y
  `22_SYSTEM_LunarPhases_EclipseStates.md` no lo referencian aún, pero
  futuras sesiones sí lo harán).
- Hasta confirmar, este archivo sigue usando `BP_DayNight` por
  consistencia con las secciones ya escritas, marcado explícitamente como
  **pendiente de verificación**, no como hecho confirmado.

---

## Comparación contra el documento original del programador (PDF) — nuevo esta sesión

El PDF `Day_Night_Module.pdf` es la única documentación vendor disponible
para este sistema. Se usa como referencia de validación, no como fuente
más autoritativa que la inspección directa del proyecto migrado (ver regla
de prioridad de fuentes al inicio de este proyecto). Resumen de lo que
confirma/amplía:

### Variables de configuración (confirmado por PDF)
| Variable (PDF) | Descripción (PDF) | Coincide con este .md |
|---|---|---|
| `Time` | Hora del día | ✅ Coincide — variable `Time` ya documentada arriba |
| `Total Day Night Time` | Índice de velocidad del tiempo | ✅ Coincide — ya documentada, valor `1` confirmado en ambos proyectos |

### UI Widget (confirmado por PDF)
El PDF confirma que la lógica de creación del widget de día/hora vive en el
**Level Blueprint**, no en `BP_DayNight`/`BP_DayNightCycle`:
```
Event BeginPlay → Create WBP Time Widget (Class: WBP_Time, Owning Player: ...) → Add to Viewport
```
Esto es información nueva no capturada antes en la sección "WBP_Time" de
este archivo (antes solo se había confirmado que `WBP_Time` lee variables
vía `Get All Actors Of Class`, sin documentar quién lo instancia). **Acción
pendiente:** confirmar si el Level Blueprint de Light Paradox tiene esta
misma lógica de creación, o si `WBP_Time` nunca llegó a instanciarse en la
migración (dato relevante ya que este .md marcaba a `WBP_Time` como "sin
replicar" — puede que la falta de este nodo en el Level Blueprint sea
justamente la razón).

### MI_SkySpherePhases — lista completa de parámetros (confirmado por PDF)
El PDF confirma la lista **completa** de parámetros del Material Instance,
ampliando lo que este archivo tenía documentado (antes solo se listaban 4
parámetros con los valores reaplicados tras el fix de Bug 2). Ubicación
confirmada: `/Game/EasySurvivalRPG/Materials/Instances`.

| Grupo | Parámetro | Valor de ejemplo (PDF) | Notas |
|---|---|---|---|
| Global | `Eclipse Phase` | 0.5 | Máscara para simular un eclipse |
| Global | `SunBrightness` | 1.0 | Brillo del sol |
| Global | `SunScale` | 0.1 | Tamaño del disco solar |
| Moon | `MoonBrightness` | 1.0 | |
| Moon | `MoonColor` | (color) | |
| Moon | `MoonGlowRotation` | 0.0 | ⚠️ Ver nota Problema Visual 2 abajo |
| Moon | `MoonGlowSharpness` | 1.5 | |
| Moon | `MoonGlowSize` | 0.15 | |
| Moon | `MoonPhase` | 0.744966 | Controla la fase visual de la luna — ver `22_SYSTEM_LunarPhases_EclipseStates.md` |
| Moon | `MoonScale` | 0.1 | |
| Stars | `StarsBlendMinDarkness` | 35.0 | |
| Stars | `StarsBlendTransition` | 0.1 | |
| Stars | `StarsBrightness` | 4.0 | |
| Stars | `StarTexture` | `T_Stars` | Confirma que `T_Stars` es la textura activa (resuelve la duda de "posible duplicado" con `gradient_sphere`/`GradientSphere` — esos dos no aparecen en este material) |

> **Nota importante — posible pista para Problema Visual 2 (estrellas
> rotando):** el grupo `Stars` **no tiene ningún parámetro de rotación
> propio** en esta lista. El único parámetro de rotación visible en todo
> `MI_SkySpherePhases` es `MoonGlowRotation`, que pertenece al grupo
> **Moon**, no Stars. Esto es evidencia (no confirmación) de que la
> rotación visible de las estrellas **no viene de un parámetro de Material
> Instance dedicado**, sino que probablemente es un efecto secundario del
> `SetWorldRotation` que ya aplica el Timeline sobre el `DirectionalLight`/
> `Moon` (ver jerarquía de componentes — `SM_SkySphere` está attachado como
> hijo de `Moon`, y si el material de estrellas usa las UVs del mesh en
> vez de World Space, rotar el mesh rota visualmente las estrellas junto
> con él). **No confirmado — pendiente inspección del grafo de
> `M_SkySphere` para confirmar si `T_Stars` usa Texture Coordinate ligado
> al mesh o a un Panner independiente.**

### Ejemplos de valores — Eclipse (confirmado por PDF, `DayNighCycleMap`)
| Test | EclipsePhase | SunBrightness | SunScale |
|---|---|---|---|
| 1 | 0.5 | 1.0 | 0.5 |
| 2 | 0.25 | 1.0 | 0.2 |
| 3 | 0.01 | 1.0 | 0.2 |

> Nota: `SunScale` varía entre los tres ejemplos (0.5 → 0.2 → 0.2), lo cual
> sugiere que estos valores fueron ajustados manualmente por captura, no
> que representen 3 "fases" discretas predefinidas del sistema. Ver
> discusión completa en `22_SYSTEM_LunarPhases_EclipseStates.md`.

### Ejemplos de valores — Fases lunares (confirmado por PDF, `DayNighCycleMap`)
| Test | MoonPhase | Descripción visual (PDF) |
|---|---|---|
| 1 | 0.5 | Luna llena / muy iluminada |
| 2 | 0.72 | Luna creciente/menguante (forma delgada) |
| 3 | 0.088 | Luna gibosa (forma ancha, no llena) |

> Ver discusión completa sobre si esto implica 3 fases discretas o un
> rango continuo en `22_SYSTEM_LunarPhases_EclipseStates.md`.

### Nota de diseño confirmada por el PDF (cita paraprasada, no textual)
El documento original indica que los eventos deben correlacionarse con el
ciclo, y que los efectos lunares afectan tanto al jugador como a los
enemigos, de forma independiente de la personalización del personaje. Esta
es la base de diseño para el nuevo sistema documentado en
`22_SYSTEM_LunarPhases_EclipseStates.md`.

### Mapa de pruebas confirmado
`DayNighCycleMap`, ubicado en `/Game/EasySurvivalRPG/Maps` — mapa de
pruebas del asset original, útil como referencia visual para validar que
la migración a Light Paradox reproduce el mismo comportamiento en cada
parámetro.

### ⏳ Pendiente — validación sistemática restante
Este PDF cubre variables de configuración y parámetros de material. **No
cubre** el EventGraph, el Timeline, ni la jerarquía de componentes — esas
partes de este .md siguen basadas únicamente en inspección directa de la
migración (Bug 1 a Bug 5). **Acción pendiente:** validar, uno por uno a
medida que avancen las sesiones, si el comportamiento reproducido coincide
con lo que el PDF documenta, especialmente los rangos de `EclipsePhase` y
`MoonPhase` de la tabla de ejemplos arriba, replicándolos en
`DayNighCycleMap` (o el mapa equivalente en Light Paradox) y comparando
visualmente contra las capturas del PDF.

---

## Assets confirmados (Content/LightParadox/Blueprints/DayNightCycle)

| Asset | Tipo | Notas |
|---|---|---|
| `BP_DayNight` | Blueprint Class | Blueprint principal del ciclo. ⚠️ Ver discrepancia de nombre vs PDF (`BP_DayNightCycle`) arriba. |
| `BPI_IsLight` | Blueprint Interface | Función `TurnOn/OffLight(TurnLightOn?: Boolean)`. Confirmado sin diferencias vs. original. |
| `MPC_IsLight` | Material Parameter Collection | Parámetro escalar `LightLevel`, Default Value `1.0`. Confirmado sin diferencias vs. original. |
| `WBP_Time` | Widget Blueprint | Widget de solo lectura — contador de Day/Time. **No controla nada de BP_DayNight**, solo lee sus variables vía `Get All Actors Of Class`. El PDF confirma que su instanciación (Create Widget + Add to Viewport) vive en el Level Blueprint — ver sección de comparación arriba. Pendiente confirmar si esa lógica existe en el Level Blueprint de Light Paradox. |

## Assets confirmados (Content/LightParadox/Meshes)

| Asset | Tipo | Notas |
|---|---|---|
| `M_Procedural_Sky_MASTER02` | Material | Pendiente verificar uso real en escena — posible candidato para el material de estrellas, ver Problema Visual 2 |
| `M_SkySphere` | Material | Reparado (nodo `MF_SunEclipse` recreado, ver Bug 2). Conteo de instrucciones: **455** vs **483** del original — diferencia de 28, pendiente investigar si es relevante. |
| `MI_SkySpherePhases` | Material Instance | Lista completa de parámetros confirmada por PDF esta sesión — ver tabla arriba. Directamente responsable del visual de eclipse confirmado funcionando esta sesión — ver Comportamiento Visual Confirmado. |
| `SM_SkySphere` | Static Mesh | Transform confirmado idéntico al original. Collision Preset cambiado a `NoCollision` — cambio intencional del usuario. Attachado como hijo de `Moon`, no de `DirectionalLight`. |
| `dark_moon` | Texture | |
| `gradient_sphere` / `GradientSphere` | Texture | Ninguna de las dos aparece en la lista de parámetros de `MI_SkySpherePhases` confirmada por el PDF — la textura de estrellas activa es `T_Stars` (parámetro `StarTexture`). Sigue pendiente confirmar en qué material/uso está cada una de estas dos texturas duplicadas. |
| `Sun` | Texture | |
| `T_Stars` | Texture | Confirmado por PDF como el parámetro `StarTexture` de `MI_SkySpherePhases` — textura activa del material de estrellas. |

## Material Functions confirmadas

| Asset | Tipo | Notas |
|---|---|---|
| `MF_SunEclipse` | Material Function | No existía en el proyecto migrado — causaba el Bug 2. Recreada manualmente (Migrate nativo falló por incompatibilidad 5.5.4→5.4.4). Reconectada en `M_SkySphere`, pin `Sun Eclipse Phase In (S)`. **Confirmado funcional esta sesión** — el visual de eclipse descrito por el usuario ya se reproduce correctamente. |

---

## Jerarquía de componentes confirmada en BP_DayNight

```
BP_DayNight (Self)
  DefaultSceneRoot
    ├── Billboard                    ← Hidden In Game intencional, igual al original
    ├── VolumetricCloud
    ├── ExponentialHeightFog
    ├── SkyAtmosphere
    └── DirectionalLight            ← "Sol"
          ├── SkyLight
          └── Moon                  ← componente de luz direccional para la luna
                └── SM_SkySphere    ← mesh de la esfera de cielo, attachado a Moon, no al Sol
```

**Aclarado sesión anterior (relevante para Bug 5):** el usuario confirma que
sol, luna y estrellas se perciben moviéndose a **timings distintos entre
sí** en el original — el usuario atribuye esto a la diferencia de distancia
entre los assets, lo cual es consistente con esta jerarquía (el Sol rota
como `DirectionalLight` directamente, mientras que `SM_SkySphere` —que
probablemente contiene tanto la luna como las estrellas en su material—
depende del transform de `Moon`, un componente distinto). Esto refuerza que
puede haber **más de un mecanismo de rotación operando en paralelo**, no
solo el `Sun Yaw` del Timeline. Ver Bug 5.

---

## Variables confirmadas en BP_DayNight

| Variable | Tipo | Notas |
|---|---|---|
| `DayNightCycle_Timeline` | Timeline (auto-generada vía My Blueprint → Timelines) | ✅ Recreado correctamente tras Bug 1. |
| `Total Day Night Time` | Float | **Confirmado = `1` en ambos proyectos** (original y migrado, valores idénticos, y confirmado nuevamente por el PDF esta sesión). Descartado como causa de cualquier bug — ver Bug 4. |
| `Time` | Float | Recibe salida del Float Track `Time`. Confirmado incrementando correctamente en runtime (Print String). Confirmado por PDF como "hour of day". |
| `Day` | Integer/Float (tipo exacto no confirmado) | Incrementado en `Update Data`, disparado por `Update` del Timeline (cada frame, no por ciclo — ver nota de diseño). |
| `Sunrise Time` / `Sunset Time` | Float | Valores simples de instancia. Sin relación a Data Table. |

---

## Timeline — `DayNightCycle_Timeline` — Tracks confirmados

**Length:** `24.00` — **Tick Group:** `TG_PrePhysics` — **Looping:** activado

| Track | Tipo | Keyframes confirmados |
|---|---|---|
| `Time` | Float Track | 2 keyframes: (0, 0) y (24, 24). Curva lineal. |
| `DayEnded` | Float Track | 1 keyframe: (0, 0). Sin uso — pin sin conectar en el EventGraph, confirmado en original y migrado. |
| `Sun Yaw` | Float Track | **0 keyframes — confirmado vacío en AMBOS proyectos** (original y migrado). Ver Bug 4 y Bug 5. |

---

## EventGraph — flujo confirmado

```
Event BeginPlay
  → Get All Actors with Interface (BPI_IsLight) → SET All Lights in Level
  → Set Play Rate (Target: DayNightCycle_Timeline, New Rate: (60*24)/Total Day Night Time)
  → Sequence
      Then 0 → DayNightCycle_Timeline.Play
      Then 1 → Set Timer by Event (Event: CheckLight, Time: Total Day Night Time, Looping: true)

DayNightCycle_Timeline (pines: Time, Day Ended, Sun Yaw)
  Update    → Update Data (Target: self)
  Time      → SET Time → alimenta los In Range(Float) de CheckLight
  Sun Yaw   → Make Rotator (Y=Pitch) → Set World Rotation (Target: Directional Light)
  Day Ended → SIN CONEXIÓN, confirmado en original y en migración.

CheckLight (Custom Event)
  → In Range (Float) [Time, Sunrise Time, Sunset Time] → NOT → Select Float (Pick A: 1.0 / B: 0.0) → Set Scalar Parameter Value (MPC_IsLight, Parameter: LightLevel)
  → In Range (Float) [Time, Sunrise Time, Sunset Time] → NOT → For Each Loop (All Lights in Level) → Turn on/Off Light (Target: BPI_IsLight, TurnLightOn?)
```

### Update Data (interior confirmado)
```
Entry (Update Data) → Get Day → Add (Day, 1) → SET Day
```

**Nota de diseño (no es bug):** disparado por `Update` del Timeline —
incrementa `Day` cada frame, no una vez por ciclo de 24h. Sin modificar
todavía — fiel al original. Revisar al diseñar el sistema de días/lunas,
probablemente debería usar el pin `Day Ended` (actualmente sin uso) en
vez de `Update`.

---

## Bug 1 — Timeline con ERROR tras migración 5.5.4 → 5.4.4
**Estado:** ✅ Resuelto. Causa: `DayNightCycle_Timeline` creada manualmente
desde el panel Variables en vez del flujo nativo My Blueprint → Timelines.
Fix: Timeline recreado correctamente con los 3 Float Tracks vía el flujo
nativo.

## Bug 2 — Material M_SkySphere incompleto tras migración
**Estado:** ✅ Resuelto (funcionalmente — diferencia menor de instrucciones
de shader pendiente de investigar, ver Deuda Técnica). Causa raíz:
`MF_SunEclipse` no existía en el grafo migrado. Fix: recreado manualmente y
reconectado.

## Bug 3 — Sin cielo ni luz visible en runtime (PIE)
**Estado:** ✅ Resuelto. Causa: `BP_DayNight` completo tenía `Hidden In
Game` activado.

## Bug 4 — Ciclo Día/Noche corría lógicamente pero sin resultado visual
**Estado:** ✅ Resuelto (fix práctico aplicado y confirmado funcionando).
Causa raíz: el Float Track `Sun Yaw` no tenía ningún keyframe. Fix: dos
keyframes de prueba (Time=0 → 0°, Time=24 → 360°).
**Pendiente:** valores de prueba (0°→360°) son provisionales — falta
definición de diseño final de la curva.

## Bug 5 — INCÓGNITA ABIERTA: el original funciona pese a Sun Yaw vacío
**Estado:** 🟣 Sin resolver — misterio confirmado, no bloqueante. Ver
detalle completo en la sección de hipótesis (sin cambios esta sesión). El
PDF no aporta información nueva sobre este punto — no documenta el
EventGraph ni el Timeline.

---

## Comportamiento Visual Confirmado (descripción del usuario — Asset Original)

- A lo largo de un ciclo (Time 0→24), el cielo se mueve de forma continua.
- Aparece un efecto de **eclipse solar** sobre el sol (controlado por el
  parámetro `Eclipse Phase` de `MI_SkySpherePhases` — ver Bug 2, valores
  completos confirmados por PDF esta sesión).
- Aproximadamente en `Time ≈ 18`, ocurre el **ocaso**: el sol llega al
  límite de la pantalla (se oculta), y del lado opuesto **sale la luna**,
  con un visual de **fase lunar** (controlado por `MoonPhase`).
- Cuando la luna está en el cielo, **aparecen las estrellas**, que son
  parte de un material separado y **están en rotación**.
- El usuario percibe que sol, luna y estrellas se mueven a **timings
  distintos entre sí** — posible efecto de la diferencia de distancia/
  escala entre los assets. No confirmado como bug o diseño intencional.

### Problemas visuales conocidos del ORIGINAL (heredados, a corregir)

**Problema Visual 1 — El cielo se ilumina de noche como si fuera de día**
Cuando la luna está en su punto máximo (medianoche visual), el cielo se
ve iluminado de forma similar al día — la única señal de que es de noche
es que el cielo está estrellado. El usuario atribuye esto probablemente a
la rotación del `DirectionalLight`: como el Directional Light sigue
siendo la fuente de luz primaria del `SkyAtmosphere` sin importar la hora,
es posible que su `Intensity` (2.75 lux, confirmado en sesión anterior)
no se reduzca ni se apague correctamente durante la noche, o que el
`CheckLight`/`MPC_IsLight` (`LightLevel`) controle las luces del nivel
pero no la iluminación atmosférica del cielo en sí.
**Confirmado nuevamente esta sesión como pendiente** — el usuario reporta
que en general el sistema ya funciona, pero la transición de iluminación
día/noche sigue viéndose extraña (da la sensación de que "siempre es de
día" aunque el cielo sí se altera ligeramente). Esto es consistente con la
hipótesis original: el cielo (SkyAtmosphere) cambia visualmente, pero la
intensidad de luz percibida no baja lo suficiente. **Sigue pendiente para
sesión de corrección visual — no investigado a fondo todavía.**

> **⚠️ Nueva pista, sesión de implementación del Poison nocturno (ver
> `22_SYSTEM_LunarPhases_EclipseStates.md`):** al conectar un `Branch`
> nuevo al mismo booleano `NOT(In Range(Time, Sunrise Time, Sunset Time))`
> que ya usa `For Each Loop (All Lights in Level) → Turn on/Off Light`,
> se confirmó por `Print String` que ese booleano **nunca cambia de
> valor** — siempre evalúa "es de noche" como `True`, sin importar cuántos
> ciclos de día/noche transcurran. Como esta misma condición alimenta la
> rama de luces del nivel, **es candidato directo a ser la causa raíz de
> este Problema Visual 1**: si `Sunrise Time`/`Sunset Time` están mal
> configurados (iguales, ambos en `0.0`, o en orden invertido), tanto el
> apagado/encendido de luces como la sensación de "siempre es de día"
> podrían tener el mismo origen. **No confirmado todavía — pendiente
> revisar los valores reales de `Sunrise Time`/`Sunset Time` en la
> instancia de `BP_DayNight` como primera acción de la próxima sesión.**
> Ver detalle completo del diagnóstico en
> `22_SYSTEM_LunarPhases_EclipseStates.md`.

**Problema Visual 2 — Las estrellas rotan**
El material/textura de estrellas (`T_Stars`, confirmado esta sesión como
el parámetro `StarTexture` de `MI_SkySpherePhases`) se ve rotando en el
cielo, lo cual es visualmente incorrecto respecto a cómo se comportaría un
cielo estrellado real.

**Aclaración importante añadida esta sesión:** la velocidad actual del
ciclo está configurada para debugging (`Total Day Night Time = 1`, ciclo de
24h completo en ~24 segundos reales). A esa velocidad de prueba, **es
esperable que las estrellas parezcan no moverse** si el mecanismo de
rotación fuera el correcto (una esfera celeste que rota una vez por noche
se movería de forma casi imperceptible en unos pocos segundos de juego).
El hecho de que el movimiento de las estrellas sea visible incluso a la
velocidad de debug es lo que hace sospechar que la rotación está atada
incorrectamente al mismo transform que mueve la luna (`Moon` →
`SM_SkySphere`, ver jerarquía de componentes) en vez de tener su propio
mecanismo, mucho más lento, independiente.

**Confirmado por PDF esta sesión:** ningún parámetro del grupo `Stars` en
`MI_SkySpherePhases` controla rotación — el único parámetro de rotación en
todo el material instance es `MoonGlowRotation` (grupo Moon). Esto apoya
la hipótesis de que la rotación de estrellas es un efecto colateral del
movimiento del mesh `SM_SkySphere` (hijo de `Moon`), no un parámetro de
material mal configurado.

**Pendiente de resolver, en orden sugerido:**
1. Volver a observar el comportamiento de las estrellas a la **velocidad
   de juego real** planeada (no la velocidad de debug actual) — es posible
   que a esa velocidad el movimiento sea aceptable o imperceptible, y que
   no sea necesario ningún fix.
2. Si sigue siendo visualmente incorrecto a velocidad real, inspeccionar
   el grafo de `M_SkySphere` para confirmar si las UVs de `T_Stars` están
   ligadas a coordenadas del mesh (World Position / Object rotation) o a
   un Panner/rotación independiente — esto no fue confirmado en esta
   sesión ni en el PDF.
3. Confirmar si esto es un bug heredado del original tal cual (dato
   registrado: el usuario confirma que ambos problemas visuales —1 y 2—
   ya existían en el proyecto original en 5.5.4, no fueron introducidos
   por la migración).

**Nota importante (sin cambios):** ambos problemas visuales existen en el
**original**, no son introducidos por la migración — se heredan tal cual.

---

## Optimización — Pendiente de comprobar

El cliente reportó que el sistema de ciclo día/noche **baja
significativamente el framerate**. Esto **no ha sido verificado
directamente por el equipo todavía** — se registra como tarea pendiente,
no como hallazgo confirmado. Sin cambios esta sesión — sigue pendiente de
perfilar antes de tocar código.

**Candidatos a revisar cuando se investigue (sin confirmar aún):**
- `VolumetricCloud` es históricamente uno de los componentes más costosos
  en tiempo de render dentro de sistemas de cielo en Unreal — candidato
  principal a perfilar primero.
- `Update Data` disparándose cada frame vía el pin `Update` del Timeline
  (ya documentado como nota de diseño) — el incremento de `Day` en sí es
  trivial, pero confirma que hay lógica ejecutándose cada frame sin
  necesidad real de esa frecuencia.
- `CheckLight` corre vía `Set Timer by Event` con `Total Day Night Time`
  como intervalo — con el valor actual (`1`), esto dispara **cada 1
  segundo**, iterando `For Each Loop (All Lights in Level)` cada vez. Con
  muchas luces en el nivel, esto podría ser una fuente real de costo
  recurrente. Candidato secundario a perfilar.
- Conteo de instrucciones de shader de `M_SkySphere` (455, con posible
  diferencia de 28 respecto al original) — no se considera prioritario
  para el problema de framerate reportado, pero queda relacionado.

**Acción pendiente:** correr el proyecto con el profiler de Unreal (`stat
unit`, `stat gpu`, o Unreal Insights) con el ciclo día/noche activo, para
confirmar o descartar cada candidato antes de optimizar nada a ciegas.

---

## WBP_Time — Widget de referencia (no bloqueante)

Confirmado como widget de **solo lectura**. No controla ni dispara nada en
`BP_DayNight` — únicamente consulta sus variables `Day` y `Time` vía
`Get All Actors Of Class (BP Day Night)` y las formatea como texto
(`"Day {Days} | Time {Hourse}:{Minutes}"`).

**Nuevo esta sesión (PDF):** el original confirma que la instanciación
(`Create WBP Time Widget` → `Add to Viewport`) vive en el **Level
Blueprint**, disparada desde `Event BeginPlay`. **Pendiente confirmar si
esta lógica existe en el Level Blueprint de Light Paradox** — si no
existe, es la razón más probable por la que `WBP_Time` está marcado como
"sin replicar" en este documento.

No es necesario para que el ciclo funcione (ya confirmado funcionando sin
él). Útil como herramienta de diagnóstico visual en pantalla y como
utilidad de UI a futuro.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Bug 5 — Sun Yaw vacío en original pero el original funciona visualmente | Contradicción sin resolver. No bloqueante — sistema ya funcional con fix práctico del Bug 4. PDF no aporta información nueva sobre esto. | 🟣 Pendiente investigación, sin prioridad urgente |
| Problema Visual 1 — cielo se ilumina de noche | Heredado del original. Nueva pista sesión de implementación de Poison: el booleano `NOT(In Range(Time, Sunrise/Sunset Time))` que alimenta el encendido/apagado de luces nunca cambia de valor — candidato directo a causa raíz compartida. Ver `22_SYSTEM_LunarPhases_EclipseStates.md`. | 🔴 Pendiente — revisar Sunrise/Sunset Time como primera acción de la próxima sesión |
| Problema Visual 2 — estrellas rotando | Heredado del original. PDF confirma que no hay parámetro de rotación en el grupo Stars de MI_SkySpherePhases — refuerza hipótesis de que la rotación viene del transform del mesh, no del material. Pendiente re-evaluar a velocidad de juego real antes de decidir si requiere fix. | 🟡 Pendiente sesión de corrección visual |
| Optimización — framerate reportado por cliente | No verificado aún por el equipo. Candidatos: VolumetricCloud, Update Data por frame, CheckLight con Total Day Night Time=1 (dispara cada 1s) | 🔴 Pendiente perfilar antes de optimizar |
| Valores definitivos de keyframes Sun Yaw | Actualmente 0°→360° como prueba. Falta definición de diseño final (interpolación, rango, timing de ocaso ~18h) | 🟡 Pendiente definición de diseño |
| Diferencia de 28 instrucciones de shader en M_SkySphere (455 vs 483) | Posible nodo/función menor faltante adicional a MF_SunEclipse | ⏳ Pendiente, prioridad menor |
| Duplicado `gradient_sphere` / `GradientSphere` | Ninguna de las dos aparece en la lista de parámetros de MI_SkySpherePhases confirmada por PDF esta sesión (la textura activa es T_Stars). Sigue sin confirmar dónde se usa cada una. | ⏳ Pendiente |
| Tipo exacto de variable `Day` (Integer vs Float) sin confirmar | Inferido del nodo Add, no confirmado desde panel Variables | ⏳ Pendiente |
| `Update Data` disparado cada frame en vez de por ciclo | Diseño del original, no bug de migración. Relevante también para optimización | ⏳ Pendiente decisión de diseño |
| `Day Ended` sin conexión | Confirmado igual en original y migrado — no es bug. Candidato natural para mover `Update Data` fuera del tick por frame | ⏳ Sin prioridad — oportunidad de mejora futura |
| WBP_Time sin replicar | Nuevo dato del PDF: su instanciación vive en el Level Blueprint (Create Widget + Add to Viewport en BeginPlay). Pendiente confirmar si esa lógica existe en el Level Blueprint de Light Paradox. | ⏳ Pendiente, baja prioridad |
| `MF_SunEclipse` recreada manualmente en vez de migrada | Confirmado funcionalmente correcto — efecto de eclipse visible y funcionando | ✅ Verificado funcional |
| **Nombre del actor: BP_DayNight vs BP_DayNightCycle (PDF)** | Discrepancia detectada esta sesión — ver sección dedicada arriba. No confirmado cuál es el nombre real actual en el proyecto. | 🔴 Pendiente verificación en el editor |
| **Validación sistemática vs PDF original** | El PDF cubre variables y parámetros de material, no EventGraph/Timeline. Falta comparar sistemáticamente cada valor de ejemplo (Eclipse/Moon) contra el comportamiento reproducido en Light Paradox. | 🟡 Pendiente, en progreso incremental |
| **Fases lunares y eclipse vinculadas a Estados** | Nuevo sistema de diseño — no implementado aún. Ver `22_SYSTEM_LunarPhases_EclipseStates.md`, sesión de hoy planea testear con State_Poison. | 🔴 Pendiente — inicio de diseño e implementación esta sesión |

---

## Checklist de Pendientes — próxima sesión (en orden sugerido)

1. 🔴 **(Nueva prioridad #1, confirmada esta sesión)** Revisar valores de
   `Sunrise Time` / `Sunset Time` en la instancia de `BP_DayNight` en el
   nivel — confirmado que el booleano "es de noche" nunca cambia a
   `False`. Candidato a causa raíz compartida con Problema Visual 1 y con
   el bug de Poison permanente. Ver `22_SYSTEM_LunarPhases_EclipseStates.md`.
2. 🔴 **Confirmar nombre real del actor** (`BP_DayNight` vs
   `BP_DayNightCycle`) directamente en el Content Browser — corregir este
   archivo si es necesario.
3. 🟡 **Definir valores finales de la curva `Sun Yaw`** — reemplazar los
   keyframes de prueba (0°→360°) por la curva de diseño real, considerando
   el timing de ocaso ~Time=18 descrito por el usuario.
3. 🟡 **Investigar Problema Visual 1** (cielo iluminado de noche) —
   revisar si `CheckLight`/`MPC_IsLight` debe también controlar
   intensidad del `SkyAtmosphere`/`DirectionalLight`, no solo las luces
   del nivel vía `BPI_IsLight`.
4. 🟡 **Re-evaluar Problema Visual 2** (estrellas rotando) a la velocidad
   de juego real (no velocidad de debug) antes de decidir si requiere fix.
   Si sigue siendo incorrecto, ubicar en el grafo de `M_SkySphere` qué
   nodo controla la rotación de `T_Stars`.
5. 🔴 **Perfilar rendimiento** con el ciclo activo (`stat unit`, `stat
   gpu`, o Unreal Insights) antes de optimizar cualquier candidato a
   ciegas.
6. 🟣 **(Sin urgencia)** Investigar Bug 5.
7. ⏳ Resolver diferencia de 28 instrucciones de shader en `M_SkySphere`.
8. ⏳ Confirmar cuál de `gradient_sphere` / `GradientSphere` está en uso real.
9. 🔴 **Iniciar sistema de fases lunares/eclipse vinculadas a Estados** —
   ver `22_SYSTEM_LunarPhases_EclipseStates.md`, testear con State_Poison.
10. 🟡 Confirmar si el Level Blueprint de Light Paradox ya crea/agrega
    `WBP_Time` al viewport (dato nuevo del PDF) — si no, decidir si se
    replica ahora o se pospone.

---

*Archivo actualizado — sesión Light Paradox. Se incorpora comparación
contra el documento original del programador (`Day_Night_Module.pdf`):
lista completa de parámetros de `MI_SkySpherePhases`, valores de ejemplo
de Eclipse/Moon Phase, confirmación de variables Time/Total Day Night
Time, y lógica de instanciación de WBP_Time en el Level Blueprint.
Se documenta discrepancia de nombre BP_DayNight vs BP_DayNightCycle como
pendiente de verificación. Se aclara Problema Visual 2 con la ausencia de
parámetro de rotación en el grupo Stars del material, y se añade la
consideración de volver a evaluar el problema a velocidad de juego real
antes de la velocidad de debug actual. Se registra el inicio del sistema
de fases lunares/eclipse vinculadas a Estados en archivo nuevo
`22_SYSTEM_LunarPhases_EclipseStates.md`.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 (sistema Day/Night migrado de proyecto externo UE 5.5.4)*
