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
        → [NUEVO — sesión actual] Guarda Effect Handle aplicado en output
          Applied Effect Handles
```

---

## STR_StateData — Row Struct

Asset: `STR_StateData`

| Campo | Tipo | Descripción |
|---|---|---|
| `Effects` | Array de DataTableRowHandle | Filas de `DT_StatusEffects` que componen este Estado. Un Estado puede contener N efectos. |
| `IsHitEffect` | Boolean | Propiedad de diseño. True = Estado ofensivo, se aplica al golpear. False = Estado pasivo, se aplica al portador o por evento. No es un filtro de código — es una convención de diseño. |
| `Rate` | Float (0–100) | Probabilidad porcentual de que el Estado se ejecute al activarse. 100 = siempre se aplica. |
| `Duration` | Float | Duración en segundos. Usado cuando EffectsDurationOverride = True. 0 = infinito. |
| `EffectsDurationOverride` | Boolean | True = usar Duration de este Estado para todos los efectos, ignorando sus duraciones individuales en DT_StatusEffects. False = cada efecto usa su propia Duration. |
| `IsPermanent` | Boolean | True = el Estado no expira. Time Remaining se pasa como 0.0. Persiste hasta que la fuente lo cancele explícitamente. |
| `AffectedFactions` | Array de E_Faction | Facciones que pueden recibir este Estado. Si el array está vacío = afecta a cualquier actor. |

### Semántica de IsHitEffect confirmada

| Contexto | IsHitEffect = False | IsHitEffect = True |
|---|---|---|
| **Runa equipada** | Efecto se aplica al jugador portador | Efecto se aplica a lo que el jugador golpea |
| **Enemigo** | Efecto se aplica al propio enemigo en Init Character | Efecto se aplica al actor golpeado en Trace Deal Damage |
| **Trigger de zona** | Efecto se aplica al actor que entra | N/A — triggers usan False por diseño |

---

## DT_States — Data Table

Asset: `DT_States` — Row Struct: `STR_StateData`

### Filas confirmadas

| Row Name | Effects | IsHitEffect | Rate | Duration | EffectsDurationOverride | IsPermanent | AffectedFactions |
|---|---|---|---|---|---|---|---|
| `State_Poison` | `[Effect_Poison]` | False | 100 | 10.0 | True | False | [] |
| `State_Freeze` | `[Effect_Slow]` | False | 100 | 10.0 | True | False | [] |
| `State_PoisonHit` | `[Effect_Poison]` | True | 25 | 10.0 | True | False | [] |

---

## BP_StateApplier — Blueprint Function Library

**Tipo:** Blueprint Function Library
**Estado:** ✅ Funcional (con fix de guard aplicado en sesión actual)

> **Nota crítica:** BP_StateApplier NO debe ser un Actor. Intentar usarlo como
> Actor via Spawn Actor congela el editor de Unreal. Debe ser siempre
> Blueprint Function Library.

### Función: ApplyState

**Inputs:**

| Parámetro | Tipo | Descripción |
|---|---|---|
| `StateRowHandle` | DataTableRowHandle | Fila del Estado en DT_States |
| `Target` | Actor Object Reference | Actor que recibirá los efectos |
| `Instigator` | Actor Object Reference | Actor que origina la aplicación (puede ser null) |

**Outputs — agregado en sesión actual:**

| Parámetro | Tipo | Descripción |
|---|---|---|
| `AppliedEffectHandles` | Array de DataTableRowHandle | Effect Handles (filas de `DT_StatusEffects`) que efectivamente se aplicaron en esta llamada. Necesario porque el caller (`EquipmentChanged`) debe guardar el Effect Handle real para poder removerlo después — el State Handle no sirve para eso (ver Bug 3 más abajo). |

### Flujo interno confirmado (actualizado con guard y output)

```
Entry (StateRowHandle, Target, Instigator)
  → Break DataTableRowHandle (StateRowHandle)
  → Get Data Table Row (DT_States)
      Row Not Found → Return Node (Applied Effect Handles: vacío)
      Row Found →
          Break STR_StateData
          Random Integer in Range (0, 100) <= Float to Int (Rate)
          → Branch
              False → Return Node (Applied Effect Handles: vacío)
              True  →
                Local Variable: AppliedEffectHandles (Array de DataTableRowHandle) — NUEVO
                For Each Loop (Effects)
                  Loop Body →
                    Break DataTableRowHandle (Array Element)
                    Get Data Table Row (DT_StatusEffects)
                      Row Found →
                        Cast To BP_Character_Base (Target)
                          Then →
                            Get Faction
                            IS EMPTY (AffectedFactions)
                            → Branch
                                True  → [guard + aplicar efecto]
                                False →
                                    CONTAINS (AffectedFactions, Get Faction)
                                    → Branch
                                        True  → [guard + aplicar efecto]
                                        False → [sin acción]

                [guard + aplicar efecto] — NUEVO en sesión actual
                  Check Status Effect By Handle (Target: Ability System Component,
                                                   Status Effect Handle: Array Element)
                  → Branch (Result)
                      True  → [ya está activo — no reaplicar, loop continúa solo]
                      False →
                          Select 1 (Float) — EffectsDurationOverride
                          Select 2 (Float) — IsPermanent → 0.0
                          Get Ability System Component
                          Make STR_SaveData_StatusEffect
                          Load Status Effect
                            Success → Append (AppliedEffectHandles, Array Element)
                          [exec sin conexión — loop avanza solo]

                Completed → Return Node (Applied Effect Handles: GET AppliedEffectHandles)
```

---

## E_Faction — confirmado

Variable en `BP_Character_Base` — categoría Settings/AI.

| Display Name |
|---|
| `Player` / `Undead` / `SmallAnimals` / `Human` / `Chickens` / `Pigs` |

---

## Integración con enemigos — BP_Character_Undead2

### Variable BaseState
- Tipo: `DataTableRowHandle` — Instance Editable — Category: `Customization`

### Init Character — aplica Estado cuando IsHitEffect = False
### Trace Deal Damage — aplica Estado cuando IsHitEffect = True

> Cast Failed en Trace Deal Damage debe ir a Play Sound at Location —
> conectarlo a Apply State causa error de garbage collection.

---

## Triggers de prueba

### StateTrigger
```
On Component Begin Overlap → Cast To BP_Character_Base → Apply State → Destroy Actor
```

### BP_SlowTrigger — Row Name: `State_Freeze`

---

## Arquitectura de ítems de ESRPGv5

### STR_ItemInstance vs STR_ItemData

| Campo | STR_ItemInstance | STR_ItemData |
|---|---|---|
| Todos los campos base | ✅ | ✅ |
| `Amount`, `Charges`, `Durability`, `Decay` | ❌ | ✅ |

> **⚠️ Inferencia:** STR_ItemInstance = definición estática en DT_Items.
> STR_ItemData = representación runtime con valores dinámicos.

### ✅ CONFIRMADO en sesión actual — construcción de STR_ItemData

`STR_ItemData` se construye en `BP_ItemsLibrary` → función **`Make Item (pure)`**,
vía el patrón:

```
Break STR_ItemInstance (desde DT_Items)
  → [todos los campos] → Make STR_ItemData
```

**Es el único punto en `BP_ItemsLibrary`** donde se llama `Make STR_ItemData`
(confirmado revisando el buscador "Find in Blueprints" sobre todas las funciones
de la librería — ninguna otra función arma este struct desde cero).

El campo `ItemStates`. fue agregado a **ambos structs** (`STR_ItemInstance` y
`STR_ItemData`), pero al hacerlo, el pin `Item States` de `Break STR_ItemInstance`
**no quedó conectado** al pin `Item States` de `Make STR_ItemData` — el cable
nunca se trazó al agregar el campo nuevo. Por eso `ItemStates` llegaba siempre
con `Length = 0` a runtime, sin importar lo que estuviera configurado en `DT_Items`.

**FIX APLICADO Y CONFIRMADO:** se conectó manualmente el pin `Item States` de
`Break STR_ItemInstance` al pin `Item States` de `Make STR_ItemData` dentro de
`Make Item (pure)`. Con esto, `ItemStates` ya se propaga correctamente a runtime.

El campo `ItemStates` fue agregado a **ambos structs**.

### Dos rutas de ítems usables

**Ruta A — Handles → DT_Abilities → DT_StatusEffects:** Corn (efecto por duración)
**Ruta B — UseAbilityClass directo:** RedDrink (cambio instantáneo de atributos)

---

## Sistema de atributos de equipamiento — ESRPGv5

### EquipmentChanged en BP_Character_Player

```
EquipmentChanged (Equipment Slot, Slot, Item)
  → SET Local Item / Local Equipment Slot
  → Switch on E_EquipmentSlot → actualiza meshes
  → Update Footstep Settings
  → Get Equipment Items → Exclude Broken Items
      → Get Items Equipment Attributes
          → Update Equipment Attributes (Attributes Component)
  → [Estados de runas — Bloque A / Bloque B, ver abajo]
```

**Patrón clave:** recalcula todos los atributos de todos los ítems equipados
en cada cambio — no solo el ítem que cambió.

### ✅ CONFIRMADO en sesión actual — EquipmentChanged se re-dispara cada tick de daño

Se instrumentó `EquipmentChanged` con un Print String al inicio de la función
(`Get Game Time in Seconds` + texto) y se confirmó mediante reproducción con
la runa de veneno equipada:

```
Time: 41.05 → 41.05 → 40.05 → 40.05 → 39.05 → 39.05 ...
```

`EquipmentChanged` se dispara **dos veces por cada tick de daño** del efecto
activo (TickInterval del poison = 1.0s, confirmado en `EffectAttributesMapped
→ DamagePerSecond = 5.0`). Con un ítem que no aplica Estados (ej. el sombrero
cosmético `Equipment_Head`), el Print solo aparece **una vez** al equipar —
confirma que el disparo repetido está ligado a la baja de vida/atributos
del personaje mientras el Estado está activo, no al acto de equipar en sí.

**Causa exacta del disparo repetido: NO CONFIRMADA.** Se infiere que algún
sistema de ESRPGv5 recalcula `EquipmentAttributes` cuando cambian los atributos
base del personaje (Health u otros), y ese recálculo dispara `EquipmentChanged`
como side-effect. No se ha inspeccionado el sistema de atributos para confirmar
el punto exacto del disparo. Pendiente para sesión futura si se decide atacar
la causa raíz en vez de solo mitigar el síntoma.

**Mitigación aplicada (no ataca la causa raíz):** guard `CheckStatusEffectByHandle`
insertado dentro de `ApplyState` (ver sección BP_StateApplier arriba). Evita que
la re-ejecución de `EquipmentChanged` vuelva a spawnear instancias duplicadas del
efecto mientras ya está activo. **Confirmado funcional** — el error de consola
`"Attempted to access ... pending kill or garbage"` en `Set Stack`/`Set Duration`
dejó de aparecer tras aplicar el guard.

---

## Estados en Rune Words — Fase 4

### Structs y variables nuevas

**STR_RuneStateHandles** (struct nuevo)
- `Handles`: Array de DataTableRowHandle

**ActiveRuneStates** en BP_Character_Player
- Tipo: Map (Integer → STR_RuneStateHandles)
- Category: `State`
- Propósito: guardar los handles de efectos activos por slot de runa

### Flujo implementado en EquipmentChanged (actualizado sesión actual)

```
[después de Update Equipment Attributes]

BLOQUE A — Remover Estados de runa anterior:
  FIND (ActiveRuneStates, Key: Slot)
    → Break STR_RuneStateHandles → Handles
    → For Each Loop
        Loop Body → Remove Status Effect By Handle (Array Element)
                    [FIX APLICADO: exec de entrada de este nodo estaba
                     desconectado — se conectó desde Loop Body. Confirmado
                     que ahora se ejecuta.]
        Completed → Branch (Item Is Valid)

BLOQUE B — Aplicar Estados de runa nueva:
  Branch
    False → REMOVE (ActiveRuneStates, Key: Slot) → [fin]
    True  →
        Break STR_ItemData (Item del input)
          → Item States → For Each Loop
              Loop Body →
                Apply State (State Row Handle: Array Element, Target: Self)
                  → Applied Effect Handles (output nuevo)
                → Append (Target Array: Handles del FIND,
                          Source Array: Applied Effect Handles)
                  [FIX APLICADO: antes se hacía Add del State Row Handle
                   directamente — handle equivocado, ver Bug 3]
                → Add (Map: ActiveRuneStates, Key: Slot,
                       Value: Make STR_RuneStateHandles)
              Completed → [fin]
```

### Estado de implementación

| Componente | Estado |
|---|---|
| `STR_RuneStateHandles` | ✅ Creado |
| `ActiveRuneStates` en BP_Character_Player | ✅ Creado |
| Bloque A — Remove en EquipmentChanged | ✅ Implementado — exec conectado |
| Bloque B — Apply en EquipmentChanged | ✅ Implementado — usa Effect Handles reales |
| Fix pin ItemStates en Make Item | ✅ Aplicado y confirmado |
| Guard anti-duplicado en ApplyState | ✅ Aplicado y confirmado |
| Prueba funcional — aplicar efecto | ✅ Funciona correctamente |
| Prueba funcional — remover efecto al desequipar | ❌ **Sigue sin funcionar — Bug 3 activo, ver abajo** |

---

## Bug 1 — Estados de runa no se aplican al equipar

**Estado: ✅ RESUELTO en sesión actual.**

**Causa raíz confirmada:** el pin `Item States` en el nodo `Break STR_ItemInstance`
dentro de `Make Item (pure)` (`BP_ItemsLibrary`) nunca estaba conectado al pin
`Item States` de `Make STR_ItemData` en el mismo grafo. El array llegaba
siempre vacío (`Length = 0`) a cualquier `STR_ItemData` en runtime,
independientemente de la configuración en `DT_Items`.

**Fix:** se conectó manualmente ese pin. Confirmado — el segundo `For Each Loop`
en `EquipmentChanged` ahora se ejecuta y `ItemStates` refleja lo configurado
en `DT_Items`.

---

## Bug 2 — Segundo slot de runa no aplica atributos de equipamiento

**Estado:** ⏳ Pendiente — prioridad menor, no atacado en esta sesión.

**Síntoma:** Al desbloquear el segundo slot de runa (HeadRuneSlot_1) y colocar
la misma runa, los `EquipmentAttributes` no se aplican. Solo aparece el
debug log "No es runa" en pantalla (Print String temporal en
`UI_ItemSlot.OnDrop`).

**Estado:** Sin tocar. Requiere inspección de `UI_ItemSlot.OnDrop` y la lógica
de detección de runas por Gameplay Tag. No se ha vuelto a probar tras los
fixes de esta sesión — posible que algo haya cambiado indirectamente, pero
no confirmado.

---

## Bug 3 — El efecto no se remueve al desequipar la runa ⚠️ ACTIVO — PENDIENTE

**Estado: 🔴 SIN RESOLVER al cierre de esta sesión.**

### Síntomas confirmados

| Configuración | Comportamiento observado |
|---|---|
| `IsPermanent = True` | Al quitar la runa, el efecto sigue activo indefinidamente. Nunca se remueve. |
| `IsPermanent = False` | Mientras la runa está equipada, el contador llega a 0 y se reinicia a 10 solo (por el re-disparo de `EquipmentChanged`, ver arriba — comportamiento esperado dado lo que sabemos). Al desequipar, el efecto **no se detiene de inmediato** — sigue corriendo hasta que expira por su propio Duration natural. |

### Diagnóstico realizado en esta sesión

1. **Hipótesis inicial (descartada como causa única):** se sospechó que `Remove
   Status Effect By Handle` recibía el **State Handle** (hacia `DT_States`, ej.
   `State_Poison`) en vez del **Effect Handle** real (hacia `DT_StatusEffects`,
   ej. `Effect_Poison`) — porque `Remove Status Effect by Handle` internamente
   compara contra `Break STR_StatusEffectInstance → Handle`, que es un Effect
   Handle.

2. **Fix aplicado:** se agregó el output `AppliedEffectHandles` a `ApplyState`
   (ver sección BP_StateApplier arriba) para que el caller reciba los Effect
   Handles reales, y se reemplazó el `Add` (State Handle) por un `Append`
   (Applied Effect Handles) en el Bloque B de `EquipmentChanged`.

3. **Resultado de la prueba tras el fix: SIN CAMBIO.** El efecto sigue sin
   detenerse al desequipar, tanto en `IsPermanent = True` como `= False`.
   No aparecen errores nuevos en consola tras el fix.

### Conclusión de esta sesión

El fix del Effect Handle era necesario y probablemente correcto a nivel de
tipo de dato, pero **no fue suficiente** — hay algo más en la cadena que
impide que la remoción tenga efecto real. La hipótesis del handle equivocado
queda parcialmente descartada como única causa; puede seguir siendo parte
del problema, pero hay al menos un factor adicional sin identificar.

### Hipótesis abiertas para la próxima sesión (ninguna confirmada)

- **A.** El `Effect Handle` que llega ahora a `ActiveRuneStates.Handles` sigue
  sin coincidir con el `Handle` real en `Active Status Effects` — posible que
  `Get Data Table Row` dentro de `ApplyState` esté generando una instancia de
  `STR_StatusEffectInstance` cuyo campo interno `Handle` no sea idéntico al
  `DataTableRowHandle` original usado para buscarlo (por ejemplo, si el campo
  `Handle` de la fila en `DT_StatusEffects` no está configurado correctamente —
  recordar la nota de `15_SYSTEM_StatusEffects.md`: el campo `Handle` no se
  autocompleta en filas nuevas y debe configurarse manualmente).
- **B.** El `FIND (ActiveRuneStates, Key: Slot)` en Bloque A no está encontrando
  la entrada correcta — posible desincronización entre el `Slot` usado al
  aplicar (Bloque B) y el `Slot` usado al remover (Bloque A), especialmente
  si el orden de ejecución de `EquipmentChanged` cambia el valor de `Slot`
  entre ambos bloques.
- **C.** El re-disparo constante de `EquipmentChanged` (ver arriba) podría estar
  interfiriendo con el propio Bloque A — si se desequipa la runa justo cuando
  `EquipmentChanged` se está re-ejecutando por el tick de daño, podría haber
  una condición de carrera entre Bloque A (remover) y Bloque B (re-aplicar)
  que anule la remoción.
- **D.** `Append` podría no estar escribiendo de vuelta correctamente en el
  array `Handles` obtenido del `FIND` — verificar si `FIND` sobre un `Map`
  devuelve una copia o una referencia; si es copia, el `Append` modifica una
  copia local y nunca se refleja en `ActiveRuneStates` hasta que se hace el
  `Add` (Map) — habría que confirmar que ese `Add` (Map) se está ejecutando
  después del `Append` con el array ya modificado, no antes.

### Plan de diagnóstico para la próxima sesión — usar Print String

**Paso 1 — Confirmar qué Effect Handle se está guardando realmente**

En Bloque B de `EquipmentChanged`, justo después del `Append`, agregar un
Print String que imprima el `Row Name` del último elemento agregado al array
`Handles` (usar `Get [Last Index]` sobre el array, luego `Break
DataTableRowHandle` → `Row Name` → Print String).

**Paso 2 — Confirmar qué Effect Handle se está intentando remover**

En Bloque A de `EquipmentChanged`, dentro del `For Each Loop` sobre `Handles`,
justo antes de `Remove Status Effect By Handle`, agregar un Print String con
el `Row Name` del `Array Element` (el handle que se va a intentar remover).

**Paso 3 — Confirmar qué Handle tienen los efectos realmente activos**

Dentro de `Remove Status Effect by Handle` (en `BP_AbilitySystemComponent`),
justo después de `Break STR_StatusEffectInstance`, agregar un Print String
temporal con el `Row Name` del `Handle` de cada elemento de `Active Status
Effects` que se está comparando en el loop.

**Comparar los tres resultados:**
- Si el Row Name del Paso 1 y el Paso 2 coinciden, pero ninguno coincide con
  ningún resultado del Paso 3 → el problema es que el efecto nunca se registró
  con ese Handle en `Active Status Effects` (revisar Hipótesis A, campo
  `Handle` mal configurado en `DT_StatusEffects` para `Effect_Poison`).
- Si el Paso 1 y el Paso 3 coinciden pero el Paso 2 no coincide con ninguno
  de los dos → el problema está en el `FIND` de Bloque A (revisar Hipótesis B).
- Si los tres coinciden pero el efecto de todas formas no se destruye →
  revisar si `Handles Are Equals` está comparando correctamente structs de
  `DataTableRowHandle` (posible bug de comparación por valor vs por referencia,
  poco común pero no descartable) — revisar Branch inmediatamente después de
  `Handles Are Equals` dentro de `Remove Status Effect by Handle`.

**Paso 4 (solo si el Paso 3 revela timing extraño)**

Si los Print Strings del Paso 2 y Paso 3 aparecen intercalados de forma
sospechosa con los prints de `EquipmentChanged` re-disparándose cada segundo
(Hallazgo confirmado arriba), investigar Hipótesis C — puede que sea necesario
agregar un pequeño lock/flag temporal que impida que Bloque B se ejecute
mientras Bloque A está en medio de su propio loop de remoción, para eliminar
la condición de carrera antes de seguir diagnosticando.

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
- Bug 1 (ItemStates vacío) — ✅ **RESUELTO en sesión actual**
- Bug 2 (segundo slot no aplica atributos) — ⏳ pendiente, prioridad menor
- Bug 3 (remoción al desequipar no funciona) — 🔴 **ACTIVO, prioridad alta, plan de diagnóstico listo arriba**

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Causa raíz del re-disparo de EquipmentChanged en cada tick de daño | ⏳ Mitigado con guard, causa raíz sin inspeccionar |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | Causa raíz: EquipmentChanged se re-disparaba en cada tick de daño, generando spawns duplicados. Mitigado con guard CheckStatusEffectByHandle en ApplyState. | ✅ Mitigado |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| Pin ItemStates desconectado en Make Item (BP_ItemsLibrary) | Causaba Bug 1 completo | ✅ Resuelto |
| Exec desconectado en Remove Status Effect By Handle (BP_Character_Player) | Contribuye a Bug 3 | ✅ Conectado — no resolvió el bug por sí solo |
| Bug 3 — remoción de Estado al desequipar no funciona | Ver sección Bug 3 arriba, plan de diagnóstico con Print String listo para próxima sesión | 🔴 Alta prioridad — activo |
| Bug 2 — Segundo slot de runa no aplica atributos | Sin retocar en esta sesión | ⏳ Prioridad menor |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar cuando se resuelva Bug 2 | ⏳ |
| Causa raíz de re-disparo de EquipmentChanged por tick de daño | No inspeccionada — solo mitigada con guard | ⏳ Media prioridad, revisar si el guard no cubre otros casos futuros |

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

*Archivo actualizado — sesión Light Paradox (debugging Bug 1 resuelto, Bug 3 diagnóstico en progreso)*
*Cambios: Bug 1 marcado como resuelto con causa raíz documentada (pin ItemStates en Make Item),
guard anti-duplicado documentado en ApplyState, output AppliedEffectHandles agregado a ApplyState,
Bug 3 documentado con síntomas, fix parcial aplicado sin éxito, e hipótesis abiertas con plan de
diagnóstico detallado para la próxima sesión, hallazgo de re-disparo de EquipmentChanged por tick
de daño documentado y mitigado*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
