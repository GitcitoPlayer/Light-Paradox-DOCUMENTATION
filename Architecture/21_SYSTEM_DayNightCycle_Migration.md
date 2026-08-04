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
| `M_SkySphere` | Material | Pendiente verificar uso real en escena |
| `MI_SkySpherePhases` | Material Instance | Probable relación con fases lunares — pendiente inspección |
| `SM_SkySphere` | Static Mesh | Pendiente verificar uso real en escena |
| `dark_moon` | Texture | |
| `gradient_sphere` / `GradientSphere` | Texture | ⚠️ Posible duplicado — verificar cuál está realmente asignado en materiales |
| `Sun` | Texture | |
| `T_Stars` | Texture | |

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
| `DayEnded` | **Float Track** (corregido — NO es Event Track, ver Corrección abajo) | Curva plana en `0.00`, con un keyframe visible en `(24.00, 0.00)`. ⚠️ Pendiente confirmar si hay keyframes ocultos no visibles en la captura (ej. un salto a `1.0` cerca de t=24). |
| `Sun Yaw` | **Float Track** | ⚠️ Contenido de la curva no capturado aún — pendiente para próxima sesión |

### Corrección de documentación (esta sesión)
En la sesión anterior se documentó `Day Ended` como **Event Track**. Corregido:
es un **Float Track**. Evidencia: el panel de `DayEnded` en el Curve Editor
muestra dropdown `External Curve`, checkbox `Synchronize View`, opción
`Reorder`, y eje Y numérico de 0 a 24 — idéntico formato a `Time` y `Sun Yaw`.
Los Event Tracks reales no tienen eje Y numérico ni curva editable.

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

---

## Bug 1 — Timeline con ERROR tras migración 5.5.4 → 5.4.4

**Estado:** ✅ Resuelto (fix aplicado sesión anterior — pendiente confirmación
final de que compila sin ERROR tras recrear las 3 tracks).

**Síntoma:** Nodo Timeline mostraba banner "ERROR!" al compilar. Faltaban
pins (`Day Ended` no aparecía) comparado con el asset original.

**Causa confirmada:** La variable `DayNightCycle_Timeline` fue creada
manualmente desde el panel Variables (`+`) en vez de usarse el flujo nativo
My Blueprint → Timelines → `+`. Esto no genera un Timeline Template real,
dejando el nodo sin tracks definidas.

**Fix aplicado:**
1. Eliminar variable manual `DayNightCycle_Timeline`.
2. Crear Timeline vía My Blueprint → Timelines → `+`.
3. Recrear tracks: `Time` (Float Track), `DayEnded` (Float Track — corregido,
   ver nota de corrección arriba), `Sun Yaw` (Float Track).
4. Reconectar nodo en EventGraph según flujo confirmado.

**Pendiente de esta sesión:** confirmar keyframes exactos del track `Sun Yaw`
(no capturado completo) y confirmar si `DayEnded` tiene keyframes ocultos
además del visible en `(24.00, 0.00)`.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Track `Sun Yaw` sin capturar completo | Necesario para terminar de validar el Timeline | ⏳ Pendiente próxima sesión |
| Keyframes ocultos posibles en `DayEnded` | Verificar si hay un salto de valor cerca de t=24 no visible en captura actual | ⏳ Pendiente próxima sesión |
| `Update Data` disparado por `Update` (cada frame) en vez de una vez por ciclo | No es bug de migración — es diseño del original. Revisar al construir sistema de días/lunas | ⏳ Pendiente decisión de diseño |
| Duplicado `gradient_sphere` / `GradientSphere` | Verificar cuál está realmente en uso en los materiales | ⏳ Pendiente |
| Tipo exacto de variable `Day` (Integer vs Float) sin confirmar | Inferido del nodo Add, no confirmado desde panel Variables | ⏳ Pendiente |
| **Verificar en escena de prueba nueva** que todos los Componentes y Materiales (`M_Procedural_Sky_MASTER02`, `M_SkySphere`, `MI_SkySpherePhases`, `SM_SkySphere`, texturas) estén correctamente asignados/apuntando entre sí tras la migración manual | Migración copia assets pero las referencias entre Material ↔ Material Instance ↔ Static Mesh ↔ Blueprint pueden romperse silenciosamente al copiar entre proyectos de distinta versión | 🔴 **Prioridad próxima sesión** |
| Confirmar que `DayNightCycle_Timeline` compila sin ERROR tras el fix del Bug 1 | Fix aplicado pero sin confirmación visual final en captura | 🔴 **Prioridad próxima sesión** |

---

## Checklist para la próxima sesión (en orden de prioridad)

1. 🔴 Probar `BP_DayNight` en una escena nueva de prueba — verificar que el
   Directional Light, `SM_SkySphere`, y todos los materiales/texturas estén
   correctamente enlazados y que el ciclo se vea funcionando visualmente
   (sol/luna rotando, luces encendiéndose/apagándose con `CheckLight`).
2. 🔴 Confirmar que el Timeline compila sin banner de `ERROR!` tras el fix
   del Bug 1.
3. ⏳ Captura completa del track `Sun Yaw` (keyframes, interpolación).
4. ⏳ Zoom al track `DayEnded` para descartar keyframes ocultos cerca de t=24.
5. ⏳ Confirmar tipo exacto de la variable `Day` (Integer vs Float) desde el
   panel Variables del original.
6. ⏳ Resolver duplicado `gradient_sphere` / `GradientSphere` — cuál está en
   uso real en los materiales.

---

*Archivo actualizado — sesión Light Paradox (migración Day/Night Cycle, corrección DayEnded a Float Track, confirmación CheckLight/BPI/MPC sin diferencias)*
*Project: Light Paradox · Base: EasySurvivalRPGv5 (sistema Day/Night migrado de proyecto externo UE 5.5.4)*
