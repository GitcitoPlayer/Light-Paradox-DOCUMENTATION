# 21 — Sistema: Day/Night Cycle · Migración UE 5.5.4 → 5.4.4
### Base Asset: [pendiente nombre del asset original — no confirmado]
### Proyecto: Light Paradox · UE 5.4.4
### Fuente: Migración manual + inspección directa del asset original — sesión Light Paradox

---

## Contexto

Sistema de ciclo Día/Noche migrado manualmente desde un proyecto en UE 5.5.4
(el desarrollador original usó la versión incorrecta de motor). Se está
replicando Blueprint por Blueprint, asset por asset, en Light Paradox (UE 5.4.4).
Servirá como base para disparar eventos de ciclos lunares y tipos de eclipse.

---

## Assets confirmados (Content/LightParadox/Blueprints/DayNightCycle)

| Asset | Tipo | Notas |
|---|---|---|
| `BP_DayNight` | Blueprint Class | Blueprint principal del ciclo |
| `BPI_IsLight` | Blueprint Interface | Función `TurnOn/OffLight(TurnLightOn?: Boolean)`. Confirmado sin diferencias vs. original. |
| `MPC_IsLight` | Material Parameter Collection | Parámetro escalar `LightLevel`, Default Value `1.0`. Confirmado sin diferencias vs. original. |

## Assets confirmados (Content/LightParadox/Meshes)

| Asset | Tipo | Notas |
|---|---|---|
| `M_Procedural_Sky_MASTER02` | Material | Pendiente verificar uso real en escena |
| `M_SkySphere` | Material | ⚠️ Estaba incompleto tras migración — ver Bug 2. Reparado parcialmente (nodo `MF_SunEclipse` recreado). Conteo de instrucciones actual: **455** vs **483** del original — diferencia de 28, pendiente investigar si es relevante. |
| `MI_SkySpherePhases` | Material Instance | Parámetros `Eclipse Phase`, `SunScale`, `MoonPhase`, `MoonScale` ahora visibles tras reparar `M_SkySphere`. Valores reaplicados. |
| `SM_SkySphere` | Static Mesh | Transform (Rotation/Location) confirmado idéntico al original. Collision Preset cambiado a `NoCollision` — cambio intencional del usuario (evita bug de personaje atorado sobre el mesh). Confirmado sin relación al Bug 3. Attachado como hijo de `Moon`, no de `DirectionalLight` — ver hallazgo de jerarquía. |
| `dark_moon` | Texture | |
| `gradient_sphere` / `GradientSphere` | Texture | ⚠️ Posible duplicado — verificar cuál está realmente asignado en materiales |
| `Sun` | Texture | |
| `T_Stars` | Texture | |

## Material Functions confirmadas

| Asset | Tipo | Notas |
|---|---|---|
| `MF_SunEclipse` | Material Function | ⚠️ No existía en el proyecto migrado — causaba el Bug 2. **Recreada manualmente** (el Migrate nativo de Unreal falló por incompatibilidad de versión 5.5.4 → 5.4.4). Reconectada en `M_SkySphere` en el pin `Sun Eclipse Phase In (S)`. |

---

## Jerarquía de componentes confirmada en BP_DayNight

Confirmada esta sesión vía inspección directa del panel Components/Details.
Más completa que lo documentado en sesiones previas:

```
BP_DayNight (Self)
  DefaultSceneRoot
    ├── Billboard
    ├── VolumetricCloud
    ├── ExponentialHeightFog
    ├── SkyAtmosphere
    └── DirectionalLight            ← "Sol"
          ├── SkyLight
          └── Moon                  ← ⚠️ ícono idéntico a Directional Light — probable segundo componente de luz direccional para la luna, NO confirmado como Scene Component vacío
                └── SM_SkySphere    ← el mesh de la esfera de cielo está attachado aquí, no al Sol
```

**Implicación no resuelta:** la lógica documentada en el EventGraph
(`Sun Yaw → Make Rotator → Set World Rotation (Target: Directional Light)`)
rota únicamente al componente `DirectionalLight` (Sol). El mesh visual
`SM_SkySphere` depende del transform de `Moon`, un componente distinto.
La relación exacta entre ambos —si hay lógica adicional que sincronice su
rotación, o que alterne cuál está activo según la hora— no está documentada
todavía. Ver Bug 3, Hipótesis 3.

---

## Variables confirmadas en BP_DayNight

| Variable | Tipo | Notas |
|---|---|---|
| `DayNightCycle_Timeline` | Timeline (auto-generada vía My Blueprint → Timelines) | ✅ Recreado correctamente tras Bug 1. Ver sección Bug 1. |
| `Total Day Night Time` | Float | Usado para `Set Play Rate` = (60 * 24) / Total Day Night Time, y como base del `Length` del Timeline |
| `Time` | Float | Recibe salida del Float Track `Time` |
| `Day` | Integer/Float (tipo exacto no confirmado — inferido de Add node) | Incrementado en `Update Data`. Ver nota de diseño abajo |
| `Sunrise Time` / `Sunset Time` | Float | Valores simples de instancia. Confirmado: sin relación a Data Table por ahora |

---

## Timeline — `DayNightCycle_Timeline` — Tracks confirmados

**Length:** `24.00` — **Tick Group:** `TG_PrePhysics` — **Looping:** activado (ícono de loop resaltado en azul)

| Track | Tipo confirmado | Comportamiento |
|---|---|---|
| `Time` | **Float Track** | Curva lineal de `0.00` a `24.00` a lo largo de todo el Length. Un keyframe en (0,0) y otro en (24,24). |
| `DayEnded` | **Float Track** | Curva plana en `0.00`, con un keyframe visible en `(24.00, 0.00)`. ⚠️ Pendiente confirmar si hay keyframes ocultos no visibles en la captura (ej. un salto a `1.0` cerca de t=24). |
| `Sun Yaw` | **Float Track** | ⚠️ **Aún sin capturar — pendiente para mañana.** Crítico para el Bug 3: si la curva posiciona el sol bajo el horizonte al iniciar el Timeline, podría contribuir al apagón. |

### Corrección de documentación (sesión previa)
`Day Ended` fue documentado erróneamente como Event Track en una sesión
anterior. Corregido: es un **Float Track**. Evidencia: el panel de
`DayEnded` en el Curve Editor muestra dropdown `External Curve`, checkbox
`Synchronize View`, opción `Reorder`, y eje Y numérico de 0 a 24 — idéntico
formato a `Time` y `Sun Yaw`. Los Event Tracks reales no tienen eje Y
numérico ni curva editable.

---

## EventGraph — flujo confirmado (asset original, coincide con BP_DayNight migrado)

```
Event BeginPlay
  → Get All Actors with Interface (BPI_IsLight) → SET All Lights in Level
  → Set Play Rate (Target: DayNightCycle_Timeline, New Rate: (60*24)/Total Day Night Time)
  → Sequence
      Then 0 → DayNightCycle_Timeline.Play
      Then 1 → Set Timer by Event (Event: CheckLight, Time: Total Day Night Time, Looping: true)

DayNightCycle_Timeline (pines confirmados, en orden: Time, Day Ended, Sun Yaw)
  Update    → Update Data (Target: self)   ⚠️ ver nota de diseño abajo
  Time      → SET Time → alimenta los In Range(Float) de CheckLight
  Sun Yaw   → Make Rotator (Y=Pitch) → Set World Rotation (Target: Directional Light)
  Day Ended → SIN CONEXIÓN, confirmado en original y en migración. No es un bug — placeholder sin uso actual.

CheckLight (Custom Event — su "contenido" es el exec chain directo, no una función separada)
  → In Range (Float) [Time, Sunrise Time, Sunset Time] → NOT → Select Float (Pick A: 1.0 / B: 0.0) → Set Scalar Parameter Value (MPC_IsLight, Parameter: LightLevel)
  → In Range (Float) [Time, Sunrise Time, Sunset Time] → NOT → For Each Loop (All Lights in Level) → Turn on/Off Light (Target: BPI_IsLight, TurnLightOn?)
```

### Update Data (interior confirmado)
```
Entry (Update Data) → Get Day → Add (Day, 1) → SET Day
```
Replicado correctamente, coincide con el original.

**Nota de diseño (no es bug de migración, es observación de lógica):**
`Update Data` está conectado al pin `Update` del Timeline, que dispara **cada
frame** mientras el Timeline reproduce — no una vez por ciclo de 24h. Si
`Day` representa un contador de días transcurridos, esto lo incrementaría
muchas veces por segundo en vez de una vez al día. No modificar esto todavía
— está fiel al original. Queda registrado como punto a revisar cuando se
diseñe el sistema de ciclos lunares/eclipses, ya que probablemente `Day`
deba incrementarse vía el pin `Day Ended` (actualmente sin usar) en vez de
`Update`.

**Pendiente de esta sesión:** buscar con `Ctrl+F` en el EventGraph nodos
relacionados a `Moon`, `Set Visibility`, `Set Intensity`, `Set Actor Hidden`
— posible lógica de encendido/apagado Sol/Luna aún no documentada, relevante
para el Bug 3.

---

## Bug 1 — Timeline con ERROR tras migración 5.5.4 → 5.4.4

**Estado:** ✅ Resuelto.

**Síntoma:** Nodo Timeline mostraba banner "ERROR!" al compilar. Faltaban
pins (`Day Ended` no aparecía) comparado con el asset original.

**Causa confirmada:** La variable `DayNightCycle_Timeline` fue creada
manualmente desde el panel Variables (`+`) en vez de usarse el flujo nativo
My Blueprint → Timelines → `+`. Esto no genera un Timeline Template real,
dejando el nodo sin tracks definidas.

**Fix aplicado:**
1. Eliminar variable manual `DayNightCycle_Timeline`.
2. Crear Timeline vía My Blueprint → Timelines → `+`.
3. Recrear tracks: `Time` (Float Track), `DayEnded` (Float Track), `Sun Yaw` (Float Track).
4. Reconectar nodo en EventGraph según flujo confirmado.

---

## Bug 2 — Material M_SkySphere incompleto tras migración

**Estado:** 🟡 Parcialmente resuelto — nodo faltante recreado y reconectado.
Persiste diferencia menor de instrucciones (455 vs 483) y persiste el
síntoma general de iluminación (ver Bug 3).

### Diagnóstico descartado
Rotación/Transform de `SM_SkySphere` y del Directional Light — confirmado
idéntico entre proyectos.

### Causa raíz confirmada
Comparando resultados de búsqueda "Eclipse" en el grafo de `M_SkySphere`
entre el proyecto original y el migrado, nodo por nodo:

| Resultado de búsqueda | Original | Migrado (antes del fix) |
|---|---|---|
| `SunEclipse` — Named Reroute Declaration | ✅ | ✅ |
| `'Eclipse Phase'` Param(1) — Scalar Parameter | ✅ | ✅ |
| `MF_SunEclipse` — Material Function Call | ✅ | ❌ Faltaba |
| `Sun Eclipse Phase In (S)` (pin de entrada) | ✅ | ❌ Faltaba |
| `SunEclipse` — Named Reroute Usage | ✅ | ✅ |
| `Eclipse` — Comment | ✅ | ✅ |

### Fix aplicado
El Migrate nativo de Unreal falló por incompatibilidad de versión (5.5.4 →
5.4.4) al intentar traer `MF_SunEclipse` con sus dependencias. Se recreó la
Material Function manualmente y se reconectó en `M_SkySphere`, pin
`Sun Eclipse Phase In (S)`. Resultado: los parámetros `Eclipse Phase`,
`SunScale`, `MoonPhase`, `MoonScale` ahora aparecen en `MI_SkySpherePhases`,
con valores reaplicados:

| Parámetro | Valor reaplicado |
|---|---|
| Eclipse Phase | `0.5871` |
| SunBrightness | `6.496162` |
| SunScale | `0.052` |
| MoonPhase | `0.744966` |

**Pendiente:** el conteo de instrucciones de shader tras el fix es **455**,
todavía por debajo de las **483** del original (diferencia de 28
instrucciones). Podría haber otro nodo/función menor faltante. Investigar
si esta diferencia es relevante para el Bug 3 o es un tema aparte, de menor
prioridad.

---

## Bug 3 — Sin cielo ni luz visible en runtime (PIE) — ACTIVO

**Estado:** 🔴 Activo — hallazgo de jerarquía nuevo esta sesión, causa raíz
aún no confirmada.

### Síntoma
El preview estático del editor (antes de dar Play) se ve correcto — cielo
estrellado y suelo visibles. Al dar Play, `Event BeginPlay` corre, el
Timeline arranca, y la escena se oscurece por completo.

### Confirmado esta sesión
- El problema ocurre específicamente en runtime (PIE), no en el editor.
- El actor `BP_DayNight` sí está colocado en el nivel de prueba.
- Intensity del `DirectionalLight` confirmado **idéntico** entre proyecto
  original y migrado: **2.75 lux** ambos.
- Subir manualmente el Intensity durante Play **no resuelve** el apagón —
  **descartado como causa directa por valor numérico**.
- Agregar un **segundo** Directional Light de prueba en la escena **sí
  ilumina correctamente** — esto indica que el `DirectionalLight` original
  de `BP_DayNight` no está aportando luz por alguna bandera/configuración
  o conflicto, no por su valor de Intensity.
- Ambos proyectos comparten la misma configuración de Auto Exposure en
  Render Settings — descartado como causa.
- `ExponentialHeightFog`: Fog Density `0.02`, Fog Height Falloff `0.2`, Fog
  Max Opacity `1.0` — valores capturados, dentro de rango razonable, no se
  identifica como causa evidente de apagón total (una densidad de 0.02 no
  debería producir oscuridad completa).
- `VolumetricCloud`: Planet Radius `6360.0`, Cloud Material
  `m_SimpleVolumetricCloud_Ins` — capturado, pendiente comparar contra el
  original si el problema persiste tras descartar las hipótesis de luz.
- No hay conflicto de doble sistema de cielo — solo `BP_DayNight` está en
  la escena de prueba.
- Colisión de `SM_SkySphere` cambiada a `NoCollision` — cambio intencional
  del usuario, confirmado sin relación al Bug 3.

### Hallazgo nuevo — jerarquía de componentes más completa de lo documentado
Ver sección "Jerarquía de componentes confirmada" arriba. Existe un
componente `Moon` con ícono de luz (idéntico al de `DirectionalLight`),
anidado como hijo de `DirectionalLight`, y `SM_SkySphere` está attachado
como hijo de `Moon`, no del Sol. Esta relación no estaba documentada en
sesiones previas y es ahora la hipótesis principal.

### Hipótesis activas para la próxima sesión (en orden de prioridad)

**Hipótesis 1 — Unidades de Intensity distintas pese a mismo número**
El campo Intensity de un Directional Light en Unreal tiene un dropdown de
unidad (Lux / Candela / Unitless) junto al valor numérico. Un mismo número
`2.75` puede representar magnitudes de luz completamente distintas según la
unidad seleccionada. No confirmado si el dropdown coincide entre proyectos.

**Hipótesis 2 — Conflicto de Atmosphere Sun Light Index entre Sol y Moon**
Unreal permite un componente con `Atmosphere Sun Light` activado en índice 0
(sol) y opcionalmente otro en índice 1 (luna). Si `DirectionalLight` y
`Moon` comparten el mismo índice, pueden anularse entre sí o causar
comportamiento indefinido en el cálculo de iluminación atmosférica.

**Hipótesis 3 — Lógica de encendido/apagado Sol/Luna no documentada**
Posible nodo en el EventGraph (no detectado aún en las exportaciones
previas) que alterna visibilidad o intensidad entre `DirectionalLight` y
`Moon` según la hora, y que podría estar dejando ambos apagados
simultáneamente. Pendiente búsqueda con Ctrl+F.

**Hipótesis 4 — Problema de configuración base del proyecto, no de BP_DayNight**
Si las hipótesis 1–3 no revelan la causa, aislar con un Blueprint mínimo de
prueba (solo los 4 componentes base, sin Moon, sin SkyLight, sin
SM_SkySphere, sin Material custom, sin lógica en EventGraph) para
determinar si el problema viene de algo dentro de BP_DayNight o de una
configuración más general del proyecto (World Settings / Post Process
Volume con override de Exposure).

---

## Plan de acción para la próxima sesión (en orden)

1. 🔴 **Confirmar unidad de Intensity** del `DirectionalLight` (dropdown
   junto al campo numérico: Lux / Candela / Unitless) y compararla contra
   el proyecto original.
2. 🔴 **Inspeccionar el componente `Moon`** — captura completa de su panel
   Details (Transform, sección Light si la tiene, sección Atmosphere and
   Cloud). Confirmar si tiene `Atmosphere Sun Light` activado y su
   `Atmosphere Sun Light Index`.
3. 🔴 **Inspeccionar el mismo dato en `DirectionalLight`** (Sol) — su
   propio `Atmosphere Sun Light Index` — y confirmar que sea distinto al
   de `Moon`.
4. 🟡 **Buscar en el EventGraph** (`Ctrl+F`) nodos relacionados a `Moon`,
   `Set Visibility`, `Set Intensity`, `Set Actor Hidden` — posible lógica
   de alternancia Sol/Luna no documentada aún.
5. 🟡 **Captura completa del track `Sun Yaw`** del Timeline — pendiente
   desde hace varias sesiones, crítico por si la curva posiciona el sol
   bajo el horizonte en `t=0`.
6. ⚪ Si los puntos 1–5 no revelan la causa: ejecutar la **prueba de
   aislamiento** — Blueprint nuevo mínimo (`BP_DayNight_TEST`) con solo los
   4 componentes base, sin Material custom, sin Moon, sin lógica en
   EventGraph. Colocar en el nivel (con el `BP_DayNight` original
   desactivado) y dar Play.
   - Si se ilumina correctamente → el problema está en algo agregado
     después (Moon, SkyLight, SM_SkySphere, Material, o EventGraph) —
     reintroducir uno por uno para aislar cuál rompe la iluminación.
   - Si sigue negro → revisar configuración base del proyecto: World
     Settings (Lightmass → Force No Precomputed Lighting) y Post Process
     Volume (override de Exposure Min/Max EV100) en el nivel de prueba.
7. ⚪ Revisar diferencia de 28 instrucciones de shader en `M_SkySphere`
   (455 vs 483) — buscar si falta algún nodo menor adicional a
   `MF_SunEclipse`.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Bug 3 — sin cielo/luz en runtime | Ver sección Bug 3 e Hipótesis 1–4 | 🔴 Activo — prioridad actual |
| Track `Sun Yaw` sin capturar completo | Pendiente desde varias sesiones, crítico para Bug 3 | 🔴 Pendiente próxima sesión |
| Relación exacta entre `Moon` y `DirectionalLight` sin documentar | ¿Alternancia Sol/Luna? ¿Sincronización de rotación? No confirmado | 🔴 Pendiente próxima sesión |
| Keyframes ocultos posibles en `DayEnded` | Verificar si hay un salto de valor cerca de t=24 | ⏳ Pendiente |
| `Update Data` disparado por `Update` (cada frame) en vez de una vez por ciclo | No es bug de migración — diseño del original. Revisar al construir sistema de días/lunas | ⏳ Pendiente decisión de diseño |
| Duplicado `gradient_sphere` / `GradientSphere` | Verificar cuál está realmente en uso en los materiales | ⏳ Pendiente |
| Tipo exacto de variable `Day` (Integer vs Float) sin confirmar | Inferido del nodo Add, no confirmado desde panel Variables | ⏳ Pendiente |
| Diferencia de 28 instrucciones de shader en M_SkySphere (455 vs 483) | Posible nodo/función menor faltante adicional a MF_SunEclipse | ⏳ Pendiente, prioridad menor |
| `MF_SunEclipse` recreada manualmente en vez de migrada | El Migrate nativo falla por incompatibilidad 5.5.4→5.4.4. Verificar que la reconstrucción manual sea funcionalmente idéntica al original | ⏳ Pendiente verificación profunda |

---

*Archivo actualizado — sesión Light Paradox (Bug 3: hallazgo de jerarquía
completa de componentes — componente `Moon` identificado como probable
segunda luz direccional con `SM_SkySphere` attachado a él; Intensity
descartado como causa directa; hipótesis de conflicto de Atmosphere Sun
Light Index y lógica Sol/Luna no documentada abiertas; plan de acción
detallado para próxima sesión.)*
*Project: Light Paradox · Base: EasySurvivalRPGv5 (sistema Day/Night migrado de proyecto externo UE 5.5.4)*
