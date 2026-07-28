# 20 — Sistema: States (Estados)
### Sistema: States — Sistema central de Estados de Light Paradox
### Base Asset: EasySurvivalRPGv5
### Fuente: Diseño del cliente + sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto

El sistema de Estados es la capa de abstracción central sobre el sistema de
Status Effects de ESRPGv5. Un Estado es un contenedor de datos que agrupa
uno o más efectos y define el contexto y condiciones de su aplicación.

**Motivación:** Reemplazar la construcción manual de nodos en triggers, enemigos,
trampas, consumibles y runas por una sola llamada a `BP_StateApplier`.
El sistema es el punto de entrada único para aplicar cualquier conjunto de
efectos en cualquier contexto del juego.

---

## Arquitectura del sistema

```
[Cualquier fuente: trigger, enemigo, trampa, consumible, runa]
        │
        ▼
  BP_StateApplier (Blueprint Function Library)
    → Lee DT_States (fila del Estado)
    → Evalúa Rate (probabilidad)
    → Si pasa:
        → Itera Effects array
        → Verifica AffectedFactions
        → Lee Duration de DT_StatusEffects
        → Aplica IsPermanent / EffectsDurationOverride
        → Llama BP_AbilitySystem.Load Status Effect (Target)
        → Guarda Effect Handle aplicado en output Applied Effect Handles
```

---

## STR_StateData — Row Struct

Asset: `STR_StateData`

| Campo | Tipo | Descripción |
|---|---|---|
| `Effects` | Array de DataTableRowHandle | Filas de `DT_StatusEffects` que componen este Estado. |
| `IsHitEffect` | Boolean | True = Estado ofensivo, se aplica al golpear. False = Estado pasivo. |
| `Rate` | Float (0–100) | Probabilidad porcentual de que el Estado se ejecute al activarse. |
| `Duration` | Float | Duración en segundos. Usado cuando EffectsDurationOverride = True. |
| `EffectsDurationOverride` | Boolean | True = usar Duration de este Estado para todos los efectos. |
| `IsPermanent` | Boolean | True = el Estado no expira. Time Remaining se pasa como 0.0. |
| `AffectedFactions` | Array de E_Faction | Facciones que pueden recibir este Estado. Vacío = cualquier actor. |

---

## DT_States — Data Table

Asset: `DT_States` — Row Struct: `STR_StateData`

| Row Name | Effects | IsHitEffect | Rate | Duration | EffectsDurationOverride | IsPermanent | AffectedFactions |
|---|---|---|---|---|---|---|---|
| `State_Poison` | `[Effect_Poison]` | False | 100 | 10.0 | True | False | [] |
| `State_Freeze` | `[Effect_Slow]` | False | 100 | 10.0 | True | False | [] |
| `State_PoisonHit` | `[Effect_Poison]` | True | 25 | 10.0 | True | False | [] |

---

## BP_StateApplier — Blueprint Function Library

**Tipo:** Blueprint Function Library
**Estado:** ✅ Funcional para aplicación. El output `AppliedEffectHandles` se
confirmó funcional esta sesión (ver Bug 3 abajo).

> **Nota crítica:** BP_StateApplier NO debe ser un Actor.

### Función: ApplyState

**Inputs:** `StateRowHandle`, `Target`, `Instigator`
**Outputs:** `AppliedEffectHandles` (Array de DataTableRowHandle)

### Flujo interno confirmado (con guard anti-duplicado)

```
Entry (StateRowHandle, Target, Instigator)
  → Get Data Table Row (DT_States)
      Row Found →
          Random Integer <= Rate → Branch
              True →
                Local Variable: AppliedEffectHandles
                For Each Loop (Effects)
                  → Faction check → [guard + aplicar efecto]

                [guard + aplicar efecto]
                  Check Status Effect By Handle (ASC, Handle)
                  → Branch (Result)
                      True  → ya activo, NO reaplica, NO hace Append
                      False → Load Status Effect → Append (AppliedEffectHandles)

                Completed → Return Node (Applied Effect Handles)
```

> **✅ CONFIRMADO esta sesión:** en la primera aplicación de un Estado
> (guard = False, efecto no estaba activo), `AppliedEffectHandles` SÍ llega
> con longitud 1 (`PUNTO 1c - AppliedEffectHandles Length (raw): 1`,
> confirmado por Print String directo sobre el output de `Apply State`,
> antes de cualquier Append externo). **`ApplyState` no es la fuente del
> Bug 3.**

---

## Estados en Rune Words — Fase 4

### Structs y variables

**STR_RuneStateHandles** — `Handles`: Array de DataTableRowHandle
**ActiveRuneStates** en BP_Character_Player — Map (Integer → STR_RuneStateHandles), Category: `State`

### Flujo en EquipmentChanged

```
[después de Update Equipment Attributes]

BLOQUE A — Remover Estados de runa anterior:
  FIND (ActiveRuneStates, Key: Slot)
    → Break STR_RuneStateHandles → Handles
    → For Each Loop
        Loop Body → Remove Status Effect By Handle (Array Element)
        Completed → Branch (Item Is Valid)
            False → REMOVE (Map: ActiveRuneStates, Key: Slot)

BLOQUE B — Aplicar Estados de runa nueva:
  Branch (Item Is Valid)
    True →
        Break STR_ItemData → Item States → For Each Loop
            Loop Body →
              Apply State (State Row Handle: Array Element, Target: Self)
                → Applied Effect Handles
              → Append (Handles del FIND, Applied Effect Handles)
              → Add (Map: ActiveRuneStates, Key: Slot, Value: Make STR_RuneStateHandles)
```

---

## Bug 1 — Estados de runa no se aplican al equipar

**Estado: ✅ RESUELTO.** Causa: pin `Item States` desconectado en `Make Item (pure)`
(`BP_ItemsLibrary`). Resuelto conectando el pin.

---

## Bug 2 — Segundo slot de runa no aplica atributos de equipamiento

**Estado:** ⏳ Pendiente — prioridad menor, sin tocar.

---

## Bug 3 — El efecto no se remueve al desequipar la runa 🔴 ACTIVO — DIAGNÓSTICO AVANZADO

**Estado al cierre de esta sesión: 🔴 SIN RESOLVER, pero con causa raíz
localizada con alta precisión mediante Print String + Output Log.**

### Síntomas confirmados (sesiones previas, siguen vigentes)

| Configuración | Comportamiento observado |
|---|---|
| `IsPermanent = True` | Al quitar la runa, el efecto sigue activo indefinidamente. |
| `IsPermanent = False` | El efecto no se detiene al desequipar — sigue corriendo hasta expirar por su propio Duration natural. Mientras la runa sigue puesta, el countdown se reinicia en loop (10→0→10) por el re-disparo automático de `EquipmentChanged`. |

### Diagnóstico ejecutado esta sesión — Print String en cadena, confirmado vía Output Log

Se instrumentaron los siguientes puntos de log (todos con `Print to Screen` +
`Print to Log`, confirmados vía Output Log filtrado, no solo HUD en pantalla
— el HUD reordena visualmente mensajes del mismo frame y no es confiable
para diagnósticos de orden de ejecución):

| Punto | Ubicación | Qué mide |
|---|---|---|
| PUNTO 1c | Justo después del exec-out de `Apply State`, antes de cualquier Append externo | Length de `AppliedEffectHandles` tal como sale crudo de `ApplyState` |
| ADD REAL | Justo después del exec-out del nodo `ADD (Map)` real en Bloque B | Slot (Key) + Length de Handles que se está guardando |
| FIND REMOVE | Justo antes del `For Each Loop` de Bloque A | Slot (Key) + Length de Handles que `FIND` devuelve |
| REMOVE - Success | Dead-end después de `Remove Status Effect By Handle`, dentro de `Loop Body` | Si esta línea nunca aparece, el `For Each Loop` está iterando sobre array vacío |
| EQCHANGED START | Primer nodo de `EquipmentChanged` | Slot + timestamp de cada disparo de la función |
| MAP REMOVE EJECUTADO | Antes del nodo `REMOVE (Map)` en la rama False de Bloque A | Confirma si el Map se autoborra en cada ciclo automático |

### Resultados confirmados (vía Output Log, filtro exacto por texto)

1. **`Apply State` funciona correctamente en la primera aplicación:**
   ```
   PUNTO 1c - AppliedEffectHandles Length (raw): 1
   ```
   Confirmado en el primer equipamiento de la sesión de PIE. `ApplyState`
   **no es la fuente del bug**.

2. **`EquipmentChanged` se dispara 2 veces por frame**, una por cada slot
   afectado en el recálculo (ej. `Slot: 0` para el casco, `Slot: 7` para la
   runa), y además se re-dispara automáticamente cada ~1s mientras el efecto
   de daño por tick está activo (ya documentado en sesión anterior, causa
   raíz aún no identificada — mitigado con guard en `ApplyState`, no
   eliminado). Al filtrar por `Slot: 7` específicamente, se confirma un ciclo
   repetido cada ~1s:
   ```
   ADD REAL     - Slot: 7 - Handles guardados: 1
   FIND REMOVE  - Slot: 7 - Handles encontrados: 0
   ```
   Esto se repite consistentemente en cada ciclo, con la runa **todavía
   puesta** — es decir, en el mismo `Slot`, en la misma clave del mismo Map
   `ActiveRuneStates`, Bloque B reporta haber guardado `1` handle, y Bloque A
   (evaluado en el ciclo siguiente) reporta encontrar `0`.

3. **`Remove Status Effect By Handle` nunca se ejecuta** (confirmado — el
   Print `REMOVE - Success: {0}` nunca imprime, ni al desequipar 1 vez ni 2
   veces consecutivas). Esto es consistente con el punto 2: el `For Each
   Loop` de Bloque A siempre itera sobre un array de longitud 0, por lo que
   `Loop Body` nunca corre ni una sola vez.

4. **`MAP REMOVE EJECUTADO` (el `REMOVE (Map)` en la rama `False` de
   `Item Is Valid`) NO se ejecuta en loop** — solo se dispara cuando el
   jugador de verdad desequipa la runa (1 vez si se desequipa 1 vez, 2 veces
   si se desequipa 2 veces). Esto **descarta la hipótesis de que el Map se
   autoborra en cada ciclo automático** mientras la runa sigue puesta. El
   `REMOVE (Map)` real solo borra una entrada que, según el diagnóstico
   punto 2, **ya llegaba vacía de antes** — es decir, es un no-op cosmético
   en la práctica, no la causa del bug.

### Conclusión de esta sesión

El bug real vive en algún punto entre el momento en que Bloque B ejecuta
`ADD (Map)` (que reporta éxito, `Handles guardados: 1`) y el momento en que
Bloque A ejecuta `FIND (Map)` en el ciclo siguiente (que reporta
`Handles encontrados: 0`) — sobre la **misma Key** (`Slot: 7`) del **mismo
Map nominal** (`ActiveRuneStates`). Las hipótesis anteriores (Effect Handle
incorrecto, comparación de Handles rota, autoborrado del Map en cada ciclo)
quedan **descartadas o no confirmadas** por este diagnóstico. La hipótesis
activa ahora es más fundamental: puede haber una desincronización a nivel de
**qué variable `ActiveRuneStates` está siendo leída/escrita realmente**, o el
struct guardado en el Map llega vacío por algún motivo no confirmado aún en
la construcción de `Make STR_RuneStateHandles` dentro de Bloque B.

### Nota sobre `Remove Status Effect from owner by handle` (función interna en BP_AbilitySystemComponent)

Se inspeccionó esta función (la que ejecutaría `Remove Status Effect By
Handle` si llegara a dispararse) y su lógica visual parece razonable: itera
`Active Status Effects`, compara `Handle` vía `Handles Are Equals`, y si
coincide hace `Destroy Actor` + `Return Node (Success: true)`. **Esta función
es irrelevante para el Bug 3 en su estado actual**, porque el diagnóstico
confirma que la llamada a `Remove Status Effect By Handle` desde
`BP_Character_Player` nunca ocurre (array vacío en el `For Each Loop` que la
contiene) — el problema está antes de llegar a esta función, no dentro de
ella. No se requiere seguir revisando esta función hasta resolver la causa
raíz en Bloque A/B.

### Plan de diagnóstico pendiente para la próxima sesión

**Paso inmediato — diferenciar "la Key no existe" de "la Key existe pero el
array interno está vacío":**

1. En Bloque A, junto al `FIND (ActiveRuneStates, Key: Slot)`, agregar un
   nodo `Contains Key` sobre la misma variable `Active Rune States` con el
   mismo `Slot` como Key.
2. Convertir el resultado (bool) a String, imprimir:
   `"CONTAINS KEY - Slot: {0} - Existe: {1}"`.
3. Insertar en paralelo al print `FIND REMOVE` ya existente, antes del `For
   Each Loop`.
4. Probar: equipar runa, esperar 2-3s, desequipar. Leer resultado vía Output
   Log (filtro `CONTAINS`).

**Interpretación esperada:**

| Resultado | Conclusión / siguiente paso |
|---|---|
| `Existe: false` siempre, incluso justo después de que `ADD REAL` reportó éxito | El `Add (Map)` de Bloque B y el `FIND` de Bloque A probablemente **no apuntan a la misma variable de instancia** `ActiveRuneStates`. Verificar (click en cada nodo Get de la variable en ambos bloques y confirmar en el panel My Blueprint que ambos resaltan la misma declaración — no una variable local sombreando el nombre, no una variable duplicada). |
| `Existe: true`, pero el array interno (`Handles`) tiene Length 0 | El problema está en la construcción de `Make STR_RuneStateHandles` en Bloque B — revisar si el pin `Handles` de ese `Make` realmente recibe el array ya modificado por el `Append`, o si por error se está pasando un array distinto/vacío al `Make`. |

**Recordatorio para la próxima sesión:** no proponer fixes nuevos sin
ejecutar este paso primero. El patrón de esta sesión (hipótesis razonable →
sin confirmar con datos → fix aplicado → sin efecto) ya se repitió una vez
con el Effect Handle; evitar repetirlo.

---

## Cobertura de Estados de gameplay

### ✅ Cubiertos completamente
Envenenar, Sangrado, Quemar, Congelar, Parálisis, Def aumentada, Atk aumentado, Def elemental

### 🟡 Cubiertos parcialmente
Petrificar, Fractura, Contusión, Herida grave, Desgarre, Invisibilidad

### 🔴 Requieren sistemas nuevos
Terror, Confundir, Cegar

---

## Plan de fases

### Fase 1 — DT_States + BP_StateApplier base ✅
### Fase 2 — IsHitEffect como filtro de diseño ✅
### Fase 3 — Inspección STR_ItemData + mecanismo de remoción ✅
### Fase 4 — Estados en Rune Words ⏳
- Implementación completa ✅
- Bug 1 (ItemStates vacío) — ✅ RESUELTO
- Bug 2 (segundo slot no aplica atributos) — ⏳ pendiente, prioridad menor
- Bug 3 (remoción al desequipar no funciona) — 🔴 ACTIVO. Causa raíz acotada
  a la desincronización Map entre Bloque A y Bloque B. Plan de diagnóstico
  con `Contains Key` listo para próxima sesión.

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Causa raíz del re-disparo de EquipmentChanged en cada tick de daño | ⏳ Mitigado con guard, causa raíz sin inspeccionar. Confirmado nuevamente esta sesión que persiste. |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | Mitigado con guard CheckStatusEffectByHandle en ApplyState | ✅ Mitigado |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| Pin ItemStates desconectado en Make Item (BP_ItemsLibrary) | Causaba Bug 1 completo | ✅ Resuelto |
| Bug 3 — remoción de Estado al desequipar no funciona | Causa raíz acotada a desincronización de Map entre Bloque A/B. Ver sección Bug 3 arriba. Plan con `Contains Key` listo. | 🔴 Alta prioridad — activo |
| Bug 2 — Segundo slot de runa no aplica atributos | Sin retocar en esta sesión | ⏳ Prioridad menor |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar cuando se resuelva Bug 2 | ⏳ |
| Causa raíz de re-disparo de EquipmentChanged por tick de daño | No inspeccionada — solo mitigada con guard. Confirmado que sigue ocurriendo, 2 veces por frame (Slot 0 + Slot 7) más el re-disparo cada ~1s | ⏳ Media prioridad |
| Múltiples Print String de diagnóstico temporal agregados esta sesión (PUNTO 1, 1b, 1c, ADD REAL, FIND REMOVE, MAP REMOVE EJECUTADO, REMOVE - Success, EQCHANGED START) | Pendientes de limpiar una vez resuelto Bug 3 — ver lista completa en contexto de sesión | ⏳ Limpieza pendiente post-fix |

---

## Nota para Logica 5 -- Estados y persistencia de runas

> **NOTA DE ARQUITECTURA PARA LOGICA 5**
>
> El sistema de Estados usa `STR_SaveData_StatusEffect` para serializar el estado
> activo de un efecto: `Effect Handle + Stack + Time Remaining`.
> Este patrón de "guardar estado de un sistema activo en un struct serializable"
> es directamente relevante para la pregunta de Logica 5:
> **"¿Puede cada herramienta o cosmetico guardar su propia configuracion de runas?"**
>
> Si el sistema de runas sigue un patrón similar — un struct con los handles de
> las runas asignadas + su estado — podría persistirse por item de la misma forma
> que los efectos activos se guardan en Save Data.
>
> Esto no es una decisión tomada. Es arquitectura observable que debe considerarse
> al diseñar la persistencia de runas en Logica 5.

---

*Archivo actualizado — sesión Light Paradox (Bug 3 — diagnóstico profundo con Print String + Output Log)*
*Cambios: Apply State confirmado funcional (descartado como causa), ciclo ADD REAL / FIND REMOVE
documentado con evidencia de Output Log, hipótesis de autoborrado del Map descartada
(MAP REMOVE EJECUTADO no corre en loop), Remove Status Effect from owner by handle inspeccionada
y descartada como causa (nunca se llega a ejecutar), plan de diagnóstico con Contains Key
preparado para la siguiente sesión*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
