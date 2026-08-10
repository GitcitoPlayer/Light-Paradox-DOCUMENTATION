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
| `M_SkySphere` | Material | ⚠️ Estaba incompleto tras migración — ver Bug 2. Reparado parcialmente esta sesión (nodo `MF_SunEclipse` recreado). |
| `MI_SkySpherePhases` | Material Instance | Parámetros `Eclipse Phase`, `SunScale`, `MoonPhase`, `MoonScale` ahora visibles tras reparar `M_SkySphere`. Valores reaplicados. |
| `SM_SkySphere` | Static Mesh | Transform (Rotation/Location) confirmado idéntico al original — copiado directamente. Descartado como causa del Bug 2. |
| `dark_moon` | Texture | |
| `gradient_sphere` / `GradientSphere` | Texture | ⚠️ Posible duplicado — verificar cuál está realmente asignado en materiales |
| `Sun` | Texture | |
| `T_Stars` | Texture | |

## Material Functions confirmadas

| Asset | Tipo | Notas |
|---|---|---|
| `MF_SunEclipse` | Material Function | ⚠️ No existía en el proyecto migrado — causaba el Bug 2. **Recreada manualmente esta sesión** (el Migrate nativo de Unreal falló por incompatibilidad de versión 5.5.4 → 5.4.4). Reconectada en `M_SkySphere` en el pin `Sun Eclipse Phase In (S)`. |

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

### Corrección de documentación (sesión anterior)
En una sesión previa se documentó `Day Ended` como **Event Track**. Corregido:
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

**Pendiente:** confirmar keyframes exactos del track `Sun Yaw` (no capturado
completo) y confirmar si `DayEnded` tiene keyframes ocultos además del
visible en `(24.00, 0.00)`.

---

## Bug 2 — Material M_SkySphere incompleto tras migración

**Estado:** 🟡 Parcialmente resuelto esta sesión — nodo faltante recreado y
reconectado, pero el síntoma original (sin cielo, sin luz) **persiste**.
Ver Bug 3 para el problema actualmente activo.

### Diagnóstico descartado
Rotación/Transform de `SM_SkySphere` y del Directional Light — confirmado
idéntico entre proyectos (copiado directamente por el usuario, valores
verificados iguales).

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

El parámetro `Eclipse Phase` y el Named Reroute existían ("la tubería"
estaba), pero no había nada del otro lado consumiendo el dato — faltaba la
llamada a la Material Function `MF_SunEclipse`, que hace el cálculo real
del efecto de eclipse sobre el sol. Esto explicaba en cascada:
- `Eclipse Phase`, `SunScale`, `MoonPhase`, `MoonScale` no aparecían como
  parámetros activos en `MI_SkySpherePhases`.
- Diferencia de instrucciones de shader (331 vs 483 del original) — faltaba
  toda la lógica interna de `MF_SunEclipse`.

### Fix aplicado esta sesión
El **Migrate nativo de Unreal falló por incompatibilidad de versión**
(5.5.4 → 5.4.4) al intentar traer `MF_SunEclipse` junto con sus
dependencias. Se optó por **recrear la Material Function manualmente**:
1. Se recreó `MF_SunEclipse` como asset nuevo en el proyecto migrado.
2. Se conectó como nodo `Material Function Call` dentro de `M_SkySphere`,
   en el pin `Sun Eclipse Phase In (S)`.
3. Resultado confirmado: los parámetros `Eclipse Phase`, `SunScale`,
   `MoonPhase`, `MoonScale` ahora aparecen correctamente en
   `MI_SkySpherePhases`, y los valores de override fueron reaplicados:

| Parámetro | Valor reaplicado |
|---|---|
| Eclipse Phase | `0.5871` |
| SunBrightness | `6.496162` |
| SunScale | `0.052` |
| MoonPhase | `0.744966` |

**Pendiente de confirmar:** si hubo alguna otra Material Function además de
`MF_SunEclipse` que también faltara (por ejemplo una equivalente para la
luna — no confirmada aún, ver Bug 3). También pendiente confirmar si el
conteo de instrucciones de shader del `M_SkySphere` reparado ya se acerca a
las 483 originales, o si sigue por debajo (indicaría más nodos faltantes).

---

## Bug 3 — Sin cielo ni luz visible pese a Material reparado (ACTIVO)

**Estado:** 🔴 Activo — en investigación, sesión actual.

**Síntoma:** Tras reparar `M_SkySphere` (Bug 2) y confirmar que los
parámetros de `MI_SkySpherePhases` ya están completos y con sus valores
reaplicados, el cielo y la luz **siguen sin verse** en la escena de prueba.

**Estado del diagnóstico:** Recién abierto. Aún no se ha determinado si el
problema persiste en:
- El preview estático del editor únicamente, o también en Play In Editor (PIE)
- El propio Material (aunque más completo, puede seguir faltando algo)
- La configuración de la escena de prueba (Directional Light, Skylight,
  Post Process, Exposure)
- La colocación real del actor `BP_DayNight` en el nivel

**Pendiente para la próxima respuesta de diagnóstico:** ver sección
"Checklist para continuar" al final de este documento.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Bug 3 — sin cielo/luz pese a Material reparado | Ver sección Bug 3 | 🔴 Activo — prioridad actual |
| Track `Sun Yaw` sin capturar completo | Necesario para terminar de validar el Timeline | ⏳ Pendiente |
| Keyframes ocultos posibles en `DayEnded` | Verificar si hay un salto de valor cerca de t=24 | ⏳ Pendiente |
| `Update Data` disparado por `Update` (cada frame) en vez de una vez por ciclo | No es bug de migración — diseño del original. Revisar al construir sistema de días/lunas | ⏳ Pendiente decisión de diseño |
| Duplicado `gradient_sphere` / `GradientSphere` | Verificar cuál está realmente en uso en los materiales | ⏳ Pendiente |
| Tipo exacto de variable `Day` (Integer vs Float) sin confirmar | Inferido del nodo Add, no confirmado desde panel Variables | ⏳ Pendiente |
| Confirmar si faltan más Material Functions en `M_SkySphere` aparte de `MF_SunEclipse` | Conteo de instrucciones de shader aún no reverificado tras el fix | ⏳ Pendiente próxima sesión |
| `MF_SunEclipse` recreada manualmente en vez de migrada | El Migrate nativo falla por incompatibilidad 5.5.4→5.4.4. Verificar que la reconstrucción manual sea funcionalmente idéntica al original (mismos cálculos internos) | ⏳ Pendiente verificación profunda |

---

## Checklist para continuar (Bug 3 — próxima respuesta)

Para acotar el Bug 3 necesito que confirmes lo siguiente:

1. **¿Ya probaste en Play In Editor (PIE)?** El viewport de edición del
   Blueprint es un preview estático — no ejecuta `Event BeginPlay` ni corre
   el Timeline. Si aún no le diste Play a la escena de prueba, es el primer
   paso antes de seguir diagnosticando el Material.
2. **¿El actor `BP_DayNight` está colocado dentro del nivel de prueba?**
   (Arrastrado al viewport del nivel, no solo abierto en su propio editor
   de Blueprint.)
3. **Conteo de instrucciones de shader actual** de `M_SkySphere` tras el
   fix — ¿ya se acerca a 483, o sigue bajo?
4. Captura del **Directional Light** en la escena de prueba: Intensity,
   Affects World, y si el checkbox **Atmosphere/Fog** o similar está activado.
5. Si el nivel de prueba tiene un actor **Sky Atmosphere** o **Sky Light**
   además del `SM_SkySphere` — confirmar si coexisten o si hay conflicto
   entre dos sistemas de cielo distintos en la misma escena.

---

*Archivo actualizado — sesión Light Paradox (Bug 2 reparado parcialmente:
MF_SunEclipse recreada manualmente tras fallo de Migrate por incompatibilidad
de versión; parámetros de MI_SkySpherePhases confirmados completos y
revalorados. Bug 3 abierto: sin cielo ni luz pese a Material reparado.)*
*Project: Light Paradox · Base: EasySurvivalRPGv5 (sistema Day/Night migrado de proyecto externo UE 5.5.4)*
