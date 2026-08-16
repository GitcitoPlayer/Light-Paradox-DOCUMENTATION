# 21 — Sistema: Day/Night Cycle · Migración UE 5.5.4 → 5.4.4
### Base Asset: [pendiente nombre del asset original — no confirmado]
### Proyecto: Light Paradox · UE 5.4.4
### Fuente: Migración manual + inspección directa del asset original — sesión Light Paradox

---

## Contexto

Sistema de ciclo Día/Noche migrado manualmente desde un proyecto en UE 5.5.4
(el desarrollador original usó la versión incorrecta de motor). Se replicó
Blueprint por Blueprint, asset por asset, en Light Paradox (UE 5.4.4).
Servirá como base para disparar eventos de ciclos lunares y tipos de eclipse.

**Estado general al cierre de esta sesión:** el ciclo día/noche ya es
**funcional** — corre lógicamente y produce movimiento visual del sol y la
luna. Quedan pendientes: un misterio sin resolver sobre el mecanismo real
del original (ver Bug 5), problemas visuales conocidos a corregir, y una
tarea de optimización de rendimiento reportada por el cliente.

---

## Assets confirmados (Content/LightParadox/Blueprints/DayNightCycle)

| Asset | Tipo | Notas |
|---|---|---|
| `BP_DayNight` | Blueprint Class | Blueprint principal del ciclo |
| `BPI_IsLight` | Blueprint Interface | Función `TurnOn/OffLight(TurnLightOn?: Boolean)`. Confirmado sin diferencias vs. original. |
| `MPC_IsLight` | Material Parameter Collection | Parámetro escalar `LightLevel`, Default Value `1.0`. Confirmado sin diferencias vs. original. |
| `WBP_Time` | Widget Blueprint | Widget de solo lectura — contador de Day/Time. **No controla nada de BP_DayNight**, solo lee sus variables vía `Get All Actors Of Class`. No bloqueante — pendiente de replicar cuando se quiera, ver sección dedicada abajo. |

## Assets confirmados (Content/LightParadox/Meshes)

| Asset | Tipo | Notas |
|---|---|---|
| `M_Procedural_Sky_MASTER02` | Material | Pendiente verificar uso real en escena — posible candidato para el material de estrellas, ver Problema Visual 2 |
| `M_SkySphere` | Material | Reparado (nodo `MF_SunEclipse` recreado, ver Bug 2). Conteo de instrucciones: **455** vs **483** del original — diferencia de 28, pendiente investigar si es relevante. |
| `MI_SkySpherePhases` | Material Instance | Parámetros `Eclipse Phase`, `SunScale`, `MoonPhase`, `MoonScale` confirmados completos tras reparar `M_SkySphere`. Valores reaplicados. Directamente responsable del visual de eclipse confirmado funcionando esta sesión — ver Comportamiento Visual Confirmado. |
| `SM_SkySphere` | Static Mesh | Transform confirmado idéntico al original. Collision Preset cambiado a `NoCollision` — cambio intencional del usuario. Attachado como hijo de `Moon`, no de `DirectionalLight`. |
| `dark_moon` | Texture | |
| `gradient_sphere` / `GradientSphere` | Texture | ⚠️ Posible duplicado — verificar cuál está realmente asignado en materiales |
| `Sun` | Texture | |
| `T_Stars` | Texture | Probable textura del material de estrellas — ver Problema Visual 2 (rotación de estrellas) |

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

**Aclarado esta sesión (relevante para Bug 5):** el usuario confirma que
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
| `Total Day Night Time` | Float | **Confirmado = `1` en ambos proyectos** (original y migrado, valores idénticos). Descartado como causa de cualquier bug — ver Bug 4. |
| `Time` | Float | Recibe salida del Float Track `Time`. Confirmado incrementando correctamente en runtime (Print String). |
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

**Estado:** ✅ Resuelto.

**Causa:** La variable `DayNightCycle_Timeline` fue creada manualmente
desde el panel Variables (`+`) en vez del flujo nativo My Blueprint →
Timelines → `+`, dejando el nodo sin tracks reales.

**Fix:** Timeline recreado correctamente con los 3 Float Tracks vía el
flujo nativo.

---

## Bug 2 — Material M_SkySphere incompleto tras migración

**Estado:** ✅ Resuelto (funcionalmente — diferencia menor de instrucciones
de shader pendiente de investigar, ver Deuda Técnica).

**Causa raíz:** El nodo `MF_SunEclipse` (Material Function Call) y su pin
de entrada `Sun Eclipse Phase In (S)` no existían en el grafo de
`M_SkySphere` migrado — confirmado por búsqueda "Eclipse" nodo por nodo
contra el original. El parámetro `Eclipse Phase` y su Named Reroute
existían, pero nada consumía el dato.

**Fix aplicado:** Migrate nativo falló por incompatibilidad de versión.
Se recreó `MF_SunEclipse` manualmente como asset nuevo y se reconectó en
`M_SkySphere`. Parámetros `Eclipse Phase`, `SunScale`, `MoonPhase`,
`MoonScale` ahora aparecen en `MI_SkySpherePhases` con valores reaplicados:

| Parámetro | Valor |
|---|---|
| Eclipse Phase | `0.5871` |
| SunBrightness | `6.496162` |
| SunScale | `0.052` |
| MoonPhase | `0.744966` |

**Confirmado esta sesión:** el efecto visual de eclipse descrito por el
usuario en el comportamiento del original ya se reproduce correctamente
en la copia migrada — el fix fue necesario y suficiente para este aspecto.

---

## Bug 3 — Sin cielo ni luz visible en runtime (PIE)

**Estado:** ✅ Resuelto.

**Causa raíz confirmada:** El Actor `BP_DayNight` completo tenía
`Hidden In Game` activado. No relacionado a ningún componente individual.

**Verificación:** Todos los componentes comparados uno a uno contra el
original — sin diferencias de visibilidad, excepto `Billboard` (Hidden In
Game intencional, igual en ambos proyectos).

---

## Bug 4 — Ciclo Día/Noche corría lógicamente pero sin resultado visual

**Estado:** ✅ Resuelto (fix práctico aplicado y confirmado funcionando).
Ver Bug 5 para la incógnita relacionada que queda abierta.

**Descartado como causa:** `Total Day Night Time` — confirmado idéntico
entre original y migrado (`= 1` en ambos), fórmula de Play Rate idéntica.
Subir el valor manualmente a `120` no produjo ningún cambio visual,
reforzando que la velocidad del Timeline nunca fue el problema real.

**Causa raíz confirmada:** El Float Track `Sun Yaw` no tenía ningún
keyframe. Un Float Track vacío devuelve un valor constante (0.0) en cada
evaluación — la cadena `Sun Yaw → Make Rotator → Set World Rotation
(Directional Light)` aplicaba siempre la misma rotación sin importar el
avance de `Time`, dejando la escena congelada en la rotación default del
Directional Light (Pitch -39.0°, ángulo de ocaso).

**Fix aplicado y confirmado:** se agregaron dos keyframes de prueba al
track `Sun Yaw` (Time=0 → 0°, Time=24 → 360°). Tras esto, **el ciclo
día/noche completo se volvió visible y funcional** — ver sección
"Comportamiento Visual Confirmado" abajo con la descripción completa del
usuario.

**Pendiente:** los valores de prueba (0° → 360°) son provisionales.
Falta definición de diseño final de la curva (interpolación, rango exacto
de grados, si debe ser lineal o con easing) — ver Checklist de Pendientes.

---

## Bug 5 — INCÓGNITA ABIERTA: el original funciona pese a Sun Yaw vacío

**Estado:** 🟣 Sin resolver — misterio confirmado, no bloqueante (el
sistema ya funciona en la copia gracias al fix práctico del Bug 4), pero
pendiente de investigar para entender el mecanismo real del original.

**La contradicción exacta:**
- El usuario confirmó, mediante inspección directa del Curve Editor, que
  el track `Sun Yaw` **no tiene ningún keyframe en el proyecto original**
  (5.5.4) — mismo estado vacío que se encontró en el proyecto migrado.
- Sin embargo, el usuario **también confirmó haber ejecutado el proyecto
  original y observado el ciclo funcionando visualmente por completo**:
  sol moviéndose por el cielo, eclipse visible, ocaso ~t=18, sol saliendo
  de pantalla y luna apareciendo del lado opuesto con fase lunar, estrellas
  apareciendo de noche.

Esto es lógicamente imposible si el `Sun Yaw` del Timeline fuera el único
mecanismo de rotación — un track sin keyframes no puede producir el
movimiento descrito.

**Hipótesis para investigar en sesión futura (no urgente, sistema ya
funcional con el fix práctico):**
1. El original podría tener la rotación real del sol/luna calculada en
   otro lugar completamente distinto al `Sun Yaw` del Timeline — por
   ejemplo dentro de la función `Update Data` (cuyo interior documentado
   hasta ahora solo muestra el incremento de `Day`, pero podría tener más
   lógica no capturada), o en un Tick separado, o usando un Sun Position
   Calculator nativo de Unreal.
2. El Curve Editor podría estar mostrando una **Named Reroute o External
   Curve** que apunta a datos reales fuera del Timeline (el original usa
   Named Reroutes en otras partes del Material — ver Bug 2 — así que no
   sería inconsistente que el Blueprint también use un patrón similar de
   indirección que no se refleja como "keyframe" visible en este track
   específico).
3. Podría haber una diferencia de comportamiento entre versiones de motor
   (5.5.4 vs 5.4.4) en cómo se evalúa un Float Track vacío — poco probable,
   pero no descartado sin evidencia.

**Por qué no es urgente resolver esto ahora:** el fix práctico del Bug 4
(agregar keyframes reales al track) ya deja el sistema funcional y
controlable de forma explícita y fácil de ajustar — que es, de hecho, un
enfoque más simple y mantenible que replicar un mecanismo oculto/indirecto
si ese fuera el caso. Se documenta como curiosidad técnica pendiente, no
como bloqueante.

---

## Comportamiento Visual Confirmado (descripción del usuario — Asset Original)

Documentado esta sesión como referencia de diseño para futuras
correcciones visuales y para el desarrollo del sistema de ciclos lunares
y eclipses:

- A lo largo de un ciclo (Time 0→24), el cielo se mueve de forma continua.
- Aparece un efecto de **eclipse solar** sobre el sol (controlado por el
  parámetro `Eclipse Phase` de `MI_SkySpherePhases` — ver Bug 2).
- Aproximadamente en `Time ≈ 18`, ocurre el **ocaso**: el sol llega al
  límite de la pantalla (se oculta), y del lado opuesto **sale la luna**,
  con un visual de **fase lunar** (controlado por `MoonPhase`).
- Cuando la luna está en el cielo, **aparecen las estrellas**, que son
  parte de un material separado y **están en rotación**.
- El usuario percibe que sol, luna y estrellas se mueven a **timings
  distintos entre sí** — posible efecto de la diferencia de distancia/
  escala entre los assets (consistente con la jerarquía de componentes:
  Sol = `DirectionalLight` directo, Luna+Estrellas = dependientes de
  `Moon` → `SM_SkySphere`, un componente distinto). No confirmado como
  bug o diseño intencional — pendiente de definición.

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
pero no la iluminación atmosférica del cielo en sí. **No investigado a
fondo todavía — pendiente para sesión de corrección visual.**

**Problema Visual 2 — Las estrellas rotan**
El material/textura de estrellas (`T_Stars`, uso exacto pendiente de
confirmar en el grafo de `M_SkySphere`) se ve rotando en el cielo, lo cual
es visualmente incorrecto respecto a cómo se comportaría un cielo
estrellado real (las estrellas deberían percibirse fijas entre sí, salvo
rotación muy lenta de la esfera celeste completa). Pendiente confirmar si
esto es un bug de UV/rotación heredado del original tal cual, o si se
puede introducir como mejora en Light Paradox.

**Nota importante:** ambos problemas visuales existen en el **original**,
no son introducidos por la migración — se heredan tal cual. Se registran
aquí como tareas de mejora futura, no como bugs de migración a corregir
contra un "estado correcto" de referencia (porque el original ya los tenía).

---

## Optimización — Pendiente de comprobar

El cliente reportó que el sistema de ciclo día/noche **baja
significativamente el framerate**. Esto **no ha sido verificado
directamente por el equipo todavía** — se registra como tarea pendiente,
no como hallazgo confirmado.

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

**Confirmado por Reference Viewer:** la relación entre `WBP_Time` y
`BP_DayNight` es de dependencia de lectura únicamente — no hay ninguna
llamada de `WBP_Time` hacia `BP_DayNight` que modifique su estado.

No es necesario para que el ciclo funcione (ya confirmado funcionando sin
él). Útil como herramienta de diagnóstico visual en pantalla y como
utilidad de UI a futuro. Pendiente de replicar cuando se priorice —
documentar su Hierarchy y estructura completa en ese momento.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Bug 5 — Sun Yaw vacío en original pero el original funciona visualmente | Contradicción sin resolver. No bloqueante — sistema ya funcional con fix práctico del Bug 4 | 🟣 Pendiente investigación, sin prioridad urgente |
| Problema Visual 1 — cielo se ilumina de noche | Heredado del original. Sospecha: Intensity del Directional Light o falta de control de iluminación atmosférica por hora | 🟡 Pendiente sesión de corrección visual |
| Problema Visual 2 — estrellas rotando | Heredado del original. Pendiente confirmar si es bug de UV o diseño a mejorar | 🟡 Pendiente sesión de corrección visual |
| Optimización — framerate reportado por cliente | No verificado aún por el equipo. Candidatos: VolumetricCloud, Update Data por frame, CheckLight con Total Day Night Time=1 (dispara cada 1s) | 🔴 Pendiente perfilar antes de optimizar |
| Valores definitivos de keyframes Sun Yaw | Actualmente 0°→360° como prueba. Falta definición de diseño final (interpolación, rango, timing de ocaso ~18h) | 🟡 Pendiente definición de diseño |
| Diferencia de 28 instrucciones de shader en M_SkySphere (455 vs 483) | Posible nodo/función menor faltante adicional a MF_SunEclipse | ⏳ Pendiente, prioridad menor |
| Duplicado `gradient_sphere` / `GradientSphere` | Verificar cuál está realmente en uso en los materiales | ⏳ Pendiente |
| Tipo exacto de variable `Day` (Integer vs Float) sin confirmar | Inferido del nodo Add, no confirmado desde panel Variables | ⏳ Pendiente |
| `Update Data` disparado cada frame en vez de por ciclo | Diseño del original, no bug de migración. Relevante también para optimización — ver sección Optimización | ⏳ Pendiente decisión de diseño |
| `Day Ended` sin conexión | Confirmado igual en original y migrado — no es bug. Candidato natural para mover `Update Data` fuera del tick por frame | ⏳ Sin prioridad — oportunidad de mejora futura |
| WBP_Time sin replicar | No bloqueante — sistema funciona sin él. Replicar cuando se priorice UI | ⏳ Pendiente, baja prioridad |
| `MF_SunEclipse` recreada manualmente en vez de migrada | Confirmado funcionalmente correcto esta sesión (efecto de eclipse visible y funcionando) | ✅ Verificado funcional |

---

## Checklist de Pendientes — próxima sesión (en orden sugerido)

1. 🟡 **Definir valores finales de la curva `Sun Yaw`** — reemplazar los
   keyframes de prueba (0°→360°) por la curva de diseño real, considerando
   el timing de ocaso ~Time=18 descrito por el usuario.
2. 🟡 **Investigar Problema Visual 1** (cielo iluminado de noche) —
   revisar si `CheckLight`/`MPC_IsLight` debe también controlar
   intensidad del `SkyAtmosphere`/`DirectionalLight`, no solo las luces
   del nivel vía `BPI_IsLight`.
3. 🟡 **Investigar Problema Visual 2** (estrellas rotando) — ubicar en el
   grafo de `M_SkySphere` qué nodo controla la rotación de `T_Stars` y
   confirmar si es bug de UV heredado o comportamiento a rediseñar.
4. 🔴 **Perfilar rendimiento** con el ciclo activo (`stat unit`, `stat
   gpu`, o Unreal Insights) antes de optimizar cualquier candidato a
   ciegas — confirmar si `VolumetricCloud`, `Update Data` por frame, o
   `CheckLight` cada 1s son la causa real del framerate reportado por el
   cliente.
5. 🟣 **(Sin urgencia)** Investigar Bug 5 — intentar entender el mecanismo
   real de rotación del original pese al track `Sun Yaw` vacío, por
   curiosidad técnica y para no repetir el mismo patrón de "pieza oculta
   no documentada" en otros sistemas migrados a futuro.
6. ⏳ Resolver diferencia de 28 instrucciones de shader en `M_SkySphere`.
7. ⏳ Confirmar cuál de `gradient_sphere` / `GradientSphere` está en uso real.

---

*Archivo actualizado — sesión Light Paradox. Ciclo Día/Noche confirmado
FUNCIONAL: Bug 1 (Timeline), Bug 2 (Material/MF_SunEclipse), Bug 3 (Hidden
In Game), y Bug 4 (Sun Yaw sin keyframes) resueltos. Bug 5 (incógnita del
mecanismo real del original) documentado como pendiente no bloqueante.
Comportamiento visual completo del original documentado como referencia de
diseño. Problemas visuales heredados (cielo iluminado de noche, estrellas
rotando) y tarea de optimización de framerate registrados como pendientes
para próximas sesiones.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 (sistema Day/Night migrado de proyecto externo UE 5.5.4)*
