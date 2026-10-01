# 21 — Sistema: Day/Night Cycle · Migración UE 5.5.4 → 5.4.4
### Base Asset: sistema Day/Night migrado de proyecto externo (UE 5.5.4)
### Proyecto: Light Paradox · UE 5.4.4
### Fuente: Migración manual + inspección directa + capturas de Details/Outliner + `Day_Night_Module.pdf` (vendor)
### Actualizado: sesión Light Paradox — Calibración día/noche (Sunrise/Sunset, Sun Yaw, Start Time, SkyLight día/noche)

---

## Contexto

Sistema de ciclo Día/Noche migrado manualmente desde un proyecto en UE 5.5.4
(el desarrollador original usó la versión incorrecta de motor). Se replicó
Blueprint por Blueprint, asset por asset, en Light Paradox (UE 5.4.4).
Servirá como base para disparar eventos de ciclos lunares y tipos de eclipse
(ver `22_SYSTEM_LunarPhases_EclipseStates.md`).

**Estado general (esta sesión): ciclo día/noche funcional y calibrado.**
- Lógica (`Sunrise Time = 6`, `Sunset Time = 18`) y visual (sol/luna) ahora
  coinciden. Confirmado por el usuario en juego con la tabla de verificación
  por `Start Time`.
- El juego arranca de día (`Start Time`).
- La iluminación del `SkyLight` cambia de día a noche (intensidad y color).
- Pendientes principales: interpolar iluminación con `Time` (salto instantáneo
  actual), mover el incremento de `Day` fuera del tick por frame (necesario
  para el DataTable de ciclos lunares), verificar efecto de `Moon.Temperature`,
  optimización, y problemas visuales heredados menores.

---

## ✅ Nombre del actor — discrepancia RESUELTA

El PDF original nombra al actor `BP_DayNightCycle`. La captura del Outliner
de esta sesión muestra el actor `BP_DayNight` con link `Edit BP_DayNight`.
**Nombre real en Light Paradox confirmado: `BP_DayNight`.** (Inspección
directa tiene prioridad sobre el PDF.)

---

## Comparación contra el documento original del programador (PDF)

El PDF `Day_Night_Module.pdf` es la única documentación vendor disponible.
Se usa como referencia de validación, no como fuente más autoritativa que la
inspección directa del proyecto migrado.

### Variables de configuración (confirmado por PDF)
| Variable (PDF) | Descripción (PDF) | Coincide con este .md |
|---|---|---|
| `Time` | Hora del día | ✅ |
| `Total Day Night Time` | Índice de velocidad del tiempo | ✅ Valor `1` confirmado (captura Details esta sesión: `1.0`) |

### UI Widget (confirmado por PDF)
La creación del widget de día/hora vive en el **Level Blueprint**:
```
Event BeginPlay → Create WBP Time Widget (Class: WBP_Time, Owning Player: ...) → Add to Viewport
```
**Pendiente:** confirmar si el Level Blueprint de Light Paradox tiene esta
lógica (posible razón de que `WBP_Time` figure "sin replicar").

### MI_SkySpherePhases — lista completa de parámetros (confirmado por PDF)
Ubicación: `/Game/EasySurvivalRPG/Materials/Instances`.

| Grupo | Parámetro | Valor de ejemplo (PDF) | Notas |
|---|---|---|---|
| Global | `Eclipse Phase` | 0.5 | Máscara para simular un eclipse |
| Global | `SunBrightness` | 1.0 | |
| Global | `SunScale` | 0.1 | Tamaño del disco solar |
| Moon | `MoonBrightness` | 1.0 | |
| Moon | `MoonColor` | (color) | |
| Moon | `MoonGlowRotation` | 0.0 | Único parámetro de rotación del material |
| Moon | `MoonGlowSharpness` | 1.5 | |
| Moon | `MoonGlowSize` | 0.15 | |
| Moon | `MoonPhase` | 0.744966 | Fase visual de la luna |
| Moon | `MoonScale` | 0.1 | |
| Stars | `StarsBlendMinDarkness` | 35.0 | |
| Stars | `StarsBlendTransition` | 0.1 | |
| Stars | `StarsBrightness` | 4.0 | |
| Stars | `StarTexture` | `T_Stars` | Textura activa de estrellas |

> **Pista para Problema Visual 2 (estrellas rotando):** el grupo `Stars` no
> tiene parámetro de rotación. Evidencia (no confirmación) de que la rotación
> viene del transform del mesh `SM_SkySphere` (hijo de `Moon`), no del
> material. Pendiente inspeccionar el grafo de `M_SkySphere`.

### Ejemplos de valores — Eclipse (PDF, `DayNighCycleMap`)
| Test | EclipsePhase | SunBrightness | SunScale |
|---|---|---|---|
| 1 | 0.5 | 1.0 | 0.5 |
| 2 | 0.25 | 1.0 | 0.2 |
| 3 | 0.01 | 1.0 | 0.2 |

### Ejemplos de valores — Fases lunares (PDF, `DayNighCycleMap`)
| Test | MoonPhase | Descripción visual (PDF) |
|---|---|---|
| 1 | 0.5 | Luna llena / muy iluminada |
| 2 | 0.72 | Creciente/menguante (delgada) |
| 3 | 0.088 | Gibosa (ancha, no llena) |

> Ver discusión sobre fases discretas vs rango continuo en
> `22_SYSTEM_LunarPhases_EclipseStates.md`.

### Nota de diseño del PDF (paráfrasis)
Los eventos deben correlacionarse con el ciclo, y los efectos lunares afectan
a jugador y enemigos por igual, independientemente del personaje.

### Mapa de pruebas
`DayNighCycleMap`, en `/Game/EasySurvivalRPG/Maps`.

### ⏳ Validación restante
El PDF no cubre EventGraph, Timeline ni jerarquía de componentes. Pendiente
validar `EclipsePhase` y `MoonPhase` contra las capturas del PDF.

---

## Assets confirmados (Content/LightParadox/Blueprints/DayNightCycle)

| Asset | Tipo | Notas |
|---|---|---|
| `BP_DayNight` | Blueprint Class | Blueprint principal del ciclo. Nombre confirmado. |
| `BPI_IsLight` | Blueprint Interface | `TurnOn/OffLight(TurnLightOn?: Boolean)`. |
| `MPC_IsLight` | Material Parameter Collection | Escalar `LightLevel`, default `1.0`. |
| `WBP_Time` | Widget Blueprint | Solo lectura (Day/Time vía `Get All Actors Of Class`). Instanciación en Level Blueprint según PDF — pendiente confirmar. |

## Assets confirmados (Content/LightParadox/Meshes)

| Asset | Tipo | Notas |
|---|---|---|
| `M_Procedural_Sky_MASTER02` | Material | Pendiente verificar uso real en escena |
| `M_SkySphere` | Material | Reparado (`MF_SunEclipse` recreado). **455** instrucciones vs **483** del original — pendiente. |
| `MI_SkySpherePhases` | Material Instance | Parámetros en tabla PDF arriba. Responsable del visual de eclipse. |
| `SM_SkySphere` | Static Mesh | Transform idéntico al original. Collision `NoCollision` (intencional). Hijo de `Moon`. |
| `dark_moon` | Texture | |
| `gradient_sphere` / `GradientSphere` | Texture | No aparecen en `MI_SkySpherePhases`. Pendiente confirmar uso. |
| `Sun` | Texture | |
| `T_Stars` | Texture | `StarTexture` activa. |

## Material Functions

| Asset | Notas |
|---|---|
| `MF_SunEclipse` | Recreada manualmente (Migrate falló 5.5.4→5.4.4). Reconectada en `M_SkySphere`. Confirmada funcional. |

---

## Jerarquía de componentes confirmada en BP_DayNight

Confirmada otra vez por captura del panel Components esta sesión.

```
BP_DayNight (Self)
  DefaultSceneRoot
    ├── Billboard                    ← Hidden In Game intencional
    ├── VolumetricCloud
    ├── ExponentialHeightFog
    ├── SkyAtmosphere
    └── DirectionalLight            ← "Sol"
          ├── SkyLight              ← hijo del DirectionalLight (hereda su rotación)
          └── Moon                  ← luz direccional de la luna
                └── SM_SkySphere
```

Sol, luna y estrellas se perciben con timings distintos entre sí en el
original. Consistente con que `SM_SkySphere` dependa del transform de `Moon`.

---

## Valores de iluminación confirmados (capturas Details, esta sesión)

### `Moon` (componente de luz direccional)
| Propiedad | Valor en captura | Notas |
|---|---|---|
| Rotation | (0, 0, 180) | Override de instancia (flecha de reset) |
| Mobility | Movable | |
| Intensity | `0.26 lux` | Override de instancia |
| Light Color | blanco | |
| Use Temperature | ✅ activo | |
| Temperature | `9000.0` en la captura | El usuario aplicó ajustes tras la captura (se sugirió `12000` o Light Color azul). **Valor final no re-capturado — pendiente documentar.** |

### `SkyLight`
| Propiedad | Valor en captura | Notas |
|---|---|---|
| Location | (-48.296291, 0, -87.059048) | |
| Rotation | (0, 75, 0) | |
| Mobility | Movable | |
| Real Time Capture | ✅ activo | Necesario para que el ambiente se actualice con el ciclo |
| Source Type | SLS Captured Scene | |
| Intensity | `0.3` | Valor base de día. De noche se sobrescribe vía `CheckLight` (ver abajo). |
| Light Color | blanco | De noche se sobrescribe vía `CheckLight`. |

### `DirectionalLight` (sol)
`Intensity = 2.75 lux` (documentado en sesión anterior, **no revisado** esta
sesión). Si de noche aún se ve luz solar, revisar.

---

## Variables confirmadas en BP_DayNight

| Variable | Tipo | Notas |
|---|---|---|
| `DayNightCycle_Timeline` | Timeline | Creado vía My Blueprint → Timelines. |
| `Total Day Night Time` | Float | `1.0` (confirmado por captura y PDF). |
| `Time` | Float | Salida del Float Track `Time`. En instancia: `0.0`. |
| `Day` | Integer (inferido) | Captura muestra `1` sin decimales (vs `1.0` en Float). Confirmar en panel Variables. |
| `Sunrise Time` | Float | **`6.0`** (Instance Editable). |
| `Sunset Time` | Float | **`18.0`** (Instance Editable). |
| `Start Time` | Float | **Nueva.** Instance Editable. Default `8.0`. Hora a la que arranca el Timeline. |
| `Night Only Items` | Array de `BP_Item_NightOnly` | Instance Editable. **3 elementos** en la instancia (`BP_Item_NightOnly`, `_2`, `_3`). Ver `23_SYSTEM_ItemSpawner_Investigacion.md`. |
| `All Lights in Level` | Array | Llenado en BeginPlay vía `Get All Actors with Interface (BPI_IsLight)`. |
| `SkyLight Intensity Day` | Float | **Nueva.** Instance Editable. Default `0.3`. |
| `SkyLight Intensity Night` | Float | **Nueva.** Instance Editable. Default `0.1` (rango de prueba `0.05`–`0.15`). |
| `SkyLight Color Day` | Linear Color | **Nueva.** Instance Editable. Default blanco (`FFFFFF`). |
| `SkyLight Color Night` | Linear Color | **Nueva.** Instance Editable. Punto de partida `7090FF` (azul frío), ajustado a ojo por el usuario — valor final no documentado. |

> Los defaults indicados son los sugeridos al crear las variables. El usuario
> pudo ajustarlos después; valores finales pendientes de re-captura.

---

## Timeline — `DayNightCycle_Timeline` — Tracks

**Length:** `24.00` — **Tick Group:** `TG_PrePhysics` — **Looping:** activado

| Track | Tipo | Keyframes |
|---|---|---|
| `Time` | Float Track | (0, 0) y (24, 24). Lineal. |
| `DayEnded` | Float Track | 1 keyframe: (0, 0). Sin uso. **Es Float Track, no Event Track: su pin es un valor, no un pin de ejecución.** |
| `Sun Yaw` | Float Track | ✅ **Recalibrado esta sesión:** (0, **90**) y (24, **450**), interpolación **Linear** en ambos. |

### Calibración de `Sun Yaw` (esta sesión)
La curva anterior (0→360) dejaba el cielo 6 h atrasado respecto a
`Sunrise = 6` / `Sunset = 18` (el usuario había puesto temporalmente
`Sunrise = 12` / `Sunset = 24` para compensar visualmente). Fórmula aplicada:
`Pitch = 15 × Time + 90`. El valor 450 equivale a 90° para el `Make Rotator`.

Tabla de verificación (confirmada por el usuario: "ya cuadra la tabla"):

| Start Time | Visual | Lógica |
|---|---|---|
| 6 | Sol en el horizonte (amanecer) | Cambia a día |
| 12 | Sol en el cenit | Día |
| 18 | Sol en el horizonte, entra la luna | Cambia a noche |
| 0 | Luna en el cenit | Noche |

Para recalibrar en el futuro: cada hora de desfase = 15°, se mueven los dos
keyframes juntos (ej. 90/450 → 105/465).

---

## EventGraph — flujo confirmado (actualizado)

```
Event BeginPlay
  → Get All Actors with Interface (BPI_IsLight) → SET All Lights in Level
  → Set Play Rate (Target: DayNightCycle_Timeline, New Rate: (60*24)/Total Day Night Time)
  → Sequence
      Then 0 → Set New Time (Target: DayNightCycle_Timeline, New Time: GET Start Time)   ← NUEVO
                 → DayNightCycle_Timeline.Play
      Then 1 → Set Timer by Event (Event: CheckLight, Time: Total Day Night Time, Looping: true)

DayNightCycle_Timeline (pines: Time, Day Ended, Sun Yaw)
  Update    → Update Data (Target: self)          ← incrementa Day cada frame (pendiente corregir)
  Time      → SET Time
  Sun Yaw   → Make Rotator (Y=Pitch) → Set World Rotation (Target: Directional Light)
  Day Ended → sin conexión (Float Track, no es exec)

CheckLight (Custom Event)
  → Sequence
      Then 0 → In Range (Time, Sunrise Time, Sunset Time) → NOT
                 → Select Float (Pick A: 1.0 / B: 0.0) → Set Scalar Parameter Value (MPC_IsLight, LightLevel)
                 → For Each Loop (All Lights in Level) → Turn on/Off Light (BPI_IsLight)
      Then 1 → Branch (NOT) → Get Player Character → Cast To BP_Character_Base → Apply State (State_Poison)
                 (ver 22_SYSTEM_LunarPhases_EclipseStates.md)
      Then 2 → For Each Loop (Night Only Items) → Set Night Visible (bVisible ← NOT)
                 (ver 23_SYSTEM_ItemSpawner_Investigacion.md)
      Then 3 → ✅ NUEVO esta sesión — iluminación del SkyLight (abajo)
```

> **Convención:** el booleano `NOT(In Range(Time, Sunrise Time, Sunset Time))`
> significa **"es de noche"** y es compartido por todas las ramas. No crear un
> segundo `In Range` para nuevas ramas.

### `Then 3` — SkyLight día/noche (✅ implementado y confirmado en juego)

```
Sequence → Then 3
  → Set Intensity (Target: SkyLight, New Intensity ← Select Float)
        Select Float: Pick A ← "es de noche" (NOT compartido)
                      A ← GET SkyLight Intensity Night
                      B ← GET SkyLight Intensity Day
  → Set Light Color (Target: SkyLight, New Light Color ← Select Linear Color, SRGB marcado)
        Select Linear Color: Pick A ← "es de noche" (mismo NOT)
                             A ← GET SkyLight Color Night
                             B ← GET SkyLight Color Day
```

Motivo: `SkyLight` es un valor único que afecta día y noche. Con `0.3` fijo la
noche se veía casi de día; con `0.1` fijo el día perdía luz y las sombras se
endurecían (observación del usuario). Con este cambio cada valor se aplica
solo en su momento.

### Update Data (interior confirmado)
```
Entry (Update Data) → Get Day → Add (Day, 1) → SET Day
```
Disparado por `Update` del Timeline: incrementa `Day` **cada frame**, no una
vez por ciclo. Ver pendiente "Day por frame" — necesario para el DataTable de
ciclos lunares/eclipses.

---

## Bugs resueltos

| # | Bug | Estado |
|---|---|---|
| 1 | Timeline con ERROR tras migración (creado desde Variables en vez de My Blueprint → Timelines) | ✅ Resuelto |
| 2 | `M_SkySphere` incompleto (`MF_SunEclipse` faltante) | ✅ Resuelto |
| 3 | Sin cielo ni luz en PIE (`Hidden In Game` activo en `BP_DayNight`) | ✅ Resuelto |
| 4 | Ciclo corría sin resultado visual (`Sun Yaw` sin keyframes) | ✅ Resuelto — curva ahora calibrada (ver arriba) |
| 5 | Incógnita: el original funciona pese a `Sun Yaw` vacío | 🟣 Sin resolver, no bloqueante |
| 6 | Poison "permanente" / luces siempre de noche (`Sunrise`/`Sunset` en `0.0` sin Instance Editable) | ✅ Resuelto — variables expuestas |
| 7 | Desincronización lógica día/noche vs curva visual `Sun Yaw` | ✅ **Resuelto esta sesión** — `Sunrise 6` / `Sunset 18` + `Sun Yaw` 90→450 Linear |
| 8 | Juego arrancaba de noche | ✅ **Resuelto esta sesión** — causas: `Sunrise 12`/`Sunset 24` temporales (workaround visual) + Timeline siempre arrancando en `Time = 0`. Fix: valores `6`/`18` + variable `Start Time` + `Set New Time` antes de `Play` |
| 9 | Noche demasiado clara (SkyLight fijo) | ✅ **Resuelto esta sesión** — `Then 3` con intensidad y color día/noche |

### Historial — workaround `Sunrise 12` / `Sunset 24`
Antes de esta sesión el usuario había puesto `Sunrise = 12` y `Sunset = 24`
porque el sol visual salía en `Time 12` y se ocultaba en `Time 24` (curva
desfasada). Era un parche sobre la lógica; el fix correcto fue mover la
curva visual. Si en el futuro se vuelve a ver desfase, **no** tocar
`Sunrise`/`Sunset`: recalibrar `Sun Yaw`.

---

## Comportamiento Visual (Asset Original)

- Durante el ciclo el cielo se mueve de forma continua.
- Efecto de **eclipse solar** sobre el sol (`Eclipse Phase`).
- Al ocaso el sol se oculta y del lado opuesto sale la luna con fase
  (`MoonPhase`).
- Con luna en el cielo aparecen estrellas (material separado, en rotación).
- Sol, luna y estrellas se mueven a timings distintos entre sí.

### Problema Visual 1 — Cielo iluminado de noche
**Estado: 🟢 Mayormente resuelto esta sesión** (pendiente validación final del
usuario con valores definitivos). Causas contribuyentes confirmadas:
`Sunrise`/`Sunset` en `0.0` (resuelto antes) y `SkyLight` con intensidad fija
(resuelto con `Then 3`). Sin revisar: `Intensity` del `DirectionalLight`
(2.75 lux) por si de noche aportara luz.

### Problema Visual 2 — Las estrellas rotan
Heredado del original. A velocidad de debug (`Total Day Night Time = 1`) es
esperable que el movimiento sea visible. Pendiente:
1. Re-evaluar a velocidad de juego real.
2. Si persiste, inspeccionar el grafo de `M_SkySphere` (UV ligadas al mesh vs
   Panner independiente).

Ambos problemas visuales existen en el **original**, no los introdujo la
migración.

---

## Optimización — Pendiente de comprobar

El cliente reportó caída de framerate con el ciclo activo. **No verificado
por el equipo.** Candidatos (sin confirmar):
- `VolumetricCloud` (costoso históricamente).
- `Update Data` por frame (el incremento es trivial, pero hay lógica por frame).
- `CheckLight` cada `Total Day Night Time` segundos: ahora ejecuta además
  Poison, `Night Only Items` y `Set Intensity`/`Set Light Color` del
  `SkyLight`. Con muchas luces, `For Each Loop (All Lights in Level)` suma costo.
- `SkyLight` con **Real Time Capture** activo (recaptura la escena
  periódicamente; costo de render). Candidato nuevo a perfilar.
- Instrucciones de shader de `M_SkySphere` (455 vs 483).

**Acción:** perfilar con `stat unit`, `stat gpu` o Unreal Insights antes de
optimizar.

---

## WBP_Time — Widget de referencia (no bloqueante)

Solo lectura. Consulta `Day` y `Time` vía `Get All Actors Of Class
(BP Day Night)` y formatea `"Day {Days} | Time {Hourse}:{Minutes}"`. El PDF
indica que su instanciación vive en el Level Blueprint. Útil como herramienta
de diagnóstico en pantalla (por ejemplo para validar el DataTable de ciclos
lunares). Pendiente confirmar si existe en el Level Blueprint de Light Paradox.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| **Interpolar iluminación con `Time`** | **Nota pendiente solicitada esta sesión.** Hoy `SkyLight` (intensidad y color) cambia de golpe en `Time = 6` y `Time = 18` porque `Then 3` corre en el timer de `CheckLight`. Mejora propuesta: interpolar con `Time` (por ejemplo `Lerp` con alpha derivada de `Time` y una ventana alrededor de 6 y 18, o curvas Float/Color dentro del Timeline) para amanecer/atardecer graduales. Considerar también el flash del primer disparo de `CheckLight`: si algún día se arranca de noche, el `SkyLight` usa su valor de clase (`0.3`) hasta el primer tick del timer. | 🟡 Pendiente — mejora visual, no bloqueante |
| **`Day` se incrementa cada frame** | `Update Data` se conecta al pin `Update` del Timeline. Bloquea el "cada cuántos días" del DataTable de ciclos lunares/eclipses. **Corrección a una sugerencia previa:** el pin `Day Ended` no sirve, porque `DayEnded` es Float Track (sin pin de ejecución). Propuesta: detectar el reinicio del loop (`Time` actual < `Previous Time`) y llamar `Update Data` solo entonces. Ver siguiente paso. | 🔴 Pendiente — siguiente tarea |
| **Verificar `Moon.Temperature`** | El usuario nota que cambiar la temperatura de `Moon` casi no afecta la mesh del personaje; mover `Light Color` del `SkyLight` sí. Inferencia (no confirmada): `Moon` tiene `Intensity 0.26 lux`, muy tenue, así que su tinte casi no se ve; el `SkyLight` aporta más luz ambiente. Prueba: subir `Moon.Intensity` temporalmente a `2.0`; si el tinte azul aparece, no hay error. | 🟡 Pendiente de verificar |
| Valores finales de `Moon` y `SkyLight` Night | Capturas anteriores a los ajustes del usuario. Re-capturar Details y actualizar tablas. | ⏳ Pendiente documentar |
| Intensity del `DirectionalLight` (2.75 lux) sin revisar | Verificar que de noche no aporte luz | ⏳ Pendiente |
| Problema Visual 2 — estrellas rotando | Ver sección | 🟡 Pendiente |
| Optimización — framerate | Ver sección | 🔴 Pendiente perfilar |
| Bug 5 (incógnita Sun Yaw vacío en el original) | No bloqueante | 🟣 Sin urgencia |
| Diferencia de 28 instrucciones en `M_SkySphere` | | ⏳ Prioridad menor |
| Duplicado `gradient_sphere` / `GradientSphere` | | ⏳ Pendiente |
| Tipo exacto de `Day` | Evidencia de Integer por captura; confirmar | ⏳ Pendiente |
| `WBP_Time` sin replicar | Ver sección | ⏳ Baja prioridad |
| `MF_SunEclipse` recreada manualmente | Verificada funcional | ✅ |
| Validación sistemática vs PDF | Valores de Eclipse/Moon pendientes de comparar | 🟡 En progreso |
| Fases lunares y eclipses vinculadas a Estados | Siguiente sistema, ver `22_SYSTEM_LunarPhases_EclipseStates.md`. Primer prototipo: DataTable con debug log. | 🔴 En curso |

---

## Checklist de pendientes — próxima sesión (orden sugerido)

1. 🔴 Mover el incremento de `Day` fuera del tick por frame (detección de reinicio de loop) y crear el DataTable de ciclos lunares/eclipses con debug log (`STR_CelestialEvent`, `DT_CelestialEvents`).
2. 🟡 Interpolar iluminación del `SkyLight` con `Time` (ver deuda técnica).
3. 🟡 Verificar `Moon.Temperature` (prueba con `Intensity 2.0`).
4. ⏳ Re-capturar Details de `Moon`/`SkyLight`/variables `SkyLight ... Night` y actualizar este archivo.
5. 🟡 Revisar `Intensity` del `DirectionalLight` de noche.
6. 🔴 Perfilar rendimiento (incluye `Real Time Capture` del `SkyLight`).
7. 🟡 Re-evaluar Problema Visual 2 a velocidad real.
8. ⏳ Diferencia de shader en `M_SkySphere`; duplicado `gradient_sphere`.
9. 🟡 Confirmar creación de `WBP_Time` en el Level Blueprint.
10. 🟣 Bug 5 (sin urgencia).

---

*Archivo actualizado — sesión Light Paradox (calibración día/noche).*
*Cambios: nombre `BP_DayNight` confirmado; `Sunrise/Sunset` = 6/18 con historial del workaround 12/24; `Sun Yaw` recalibrado (90→450, Linear) y tabla de verificación confirmada; variable `Start Time` + `Set New Time` en BeginPlay (juego arranca de día); `CheckLight` Sequence documentado completo con `Then 3` (SkyLight Intensity y Color día/noche) y variables nuevas; valores de Details de `Moon` y `SkyLight`; `Night Only Items` con 3 instancias; bugs 6–9 resueltos; pendientes nuevos: interpolación con `Time`, verificar `Moon.Temperature`, `Day` por frame (con corrección sobre el pin `Day Ended`), costo de `Real Time Capture`.*
*Project: Light Paradox · Base: EasySurvivalRPGv5 (Day/Night migrado de UE 5.5.4) · UE 5.4.4*
