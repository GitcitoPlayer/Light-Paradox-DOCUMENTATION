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
**Estado:** ✅ Funcional

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

### Flujo interno confirmado

```
Entry (StateRowHandle, Target, Instigator)
  → Break DataTableRowHandle (StateRowHandle)
  → Get Data Table Row (DT_States)
      Row Not Found → Return Node
      Row Found →
          Break STR_StateData
          Random Integer in Range (0, 100) <= Float to Int (Rate)
          → Branch
              False → Return Node
              True  →
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
                                True  → [aplicar efecto]
                                False →
                                    CONTAINS (AffectedFactions, Get Faction)
                                    → Branch
                                        True  → [aplicar efecto]
                                        False → [sin acción]

                [aplicar efecto]
                  Select 1 (Float) — EffectsDurationOverride
                  Select 2 (Float) — IsPermanent → 0.0
                  Get Ability System Component
                  Make STR_SaveData_StatusEffect
                  Load Status Effect
                  [exec sin conexión — loop avanza solo]

                Completed → Return Node
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
> Requiere inspección futura para confirmar dónde se construye STR_ItemData
> desde STR_ItemInstance.

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
  → [NUEVO — Estados de runas]
```

**Patrón clave:** recalcula todos los atributos de todos los ítems equipados
en cada cambio — no solo el ítem que cambió.

---

## Estados en Rune Words — Fase 4

### Structs y variables nuevas

**STR_RuneStateHandles** (struct nuevo)
- `Handles`: Array de DataTableRowHandle

**ActiveRuneStates** en BP_Character_Player
- Tipo: Map (Integer → STR_RuneStateHandles)
- Category: `State`
- Propósito: guardar los handles de efectos activos por slot de runa

### Flujo implementado en EquipmentChanged

```
[después de Update Equipment Attributes]

BLOQUE A — Remover Estados de runa anterior:
  FIND (ActiveRuneStates, Key: Slot)
    → Break STR_RuneStateHandles → Handles
    → For Each Loop
        Loop Body → Remove Status Effect By Handle (Array Element)
        Completed → Branch (Item Is Valid)

BLOQUE B — Aplicar Estados de runa nueva:
  Branch
    False → REMOVE (ActiveRuneStates, Key: Slot) → [fin]
    True  →
        Break STR_ItemData (Item del input)
          → Item States → For Each Loop
              Loop Body →
                Apply State (State Row Handle: Array Element, Target: Self)
                → Add (Array: Handles del FIND)
                → Add (Map: ActiveRuneStates, Key: Slot, Value: Make STR_RuneStateHandles)
              Completed → [fin]
```

### Estado de implementación

| Componente | Estado |
|---|---|
| `STR_RuneStateHandles` | ✅ Creado |
| `ActiveRuneStates` en BP_Character_Player | ✅ Creado |
| Bloque A — Remove en EquipmentChanged | ✅ Implementado |
| Bloque B — Apply en EquipmentChanged | ✅ Implementado |
| Prueba funcional | ❌ Sin efecto visible — bug pendiente |

---

## Bug 1 — Estados de runa no se aplican al equipar

**Síntoma:** Al equipar una runa con `ItemStates` configurado (`State_Poison`),
no se aplica ningún efecto. No hay errores en el log.

**Diagnóstico realizado:**

| Paso | Print String | Resultado |
|---|---|---|
| Después de Update Equipment Attributes | "EquipmentChanged ejecutado" | ✅ Aparece — función sí se ejecuta |
| Dentro del segundo For Each Loop | "ItemState encontrado" | ❌ No aparece — loop no se ejecuta |
| En el True del Branch (Item Is Valid) | "Item es valido" | ✅ Aparece — Branch llega al True |
| Length de ItemStates array | Número | ❌ Siempre devuelve 0 |

**Causa confirmada:** `ItemStates` llega con Length 0 al segundo `For Each Loop`.
El array está vacío aunque el ítem tenga `ItemStates` configurado en `DT_Items`.

**Hipótesis de causa raíz:**
El pin `Item` de `EquipmentChanged` es de tipo `STR_ItemData`. El campo `ItemStates`
fue agregado a `STR_ItemInstance` (el Row Struct de `DT_Items`) pero puede que
`STR_ItemData` en runtime no incluya ese campo — o que el struct se construya
desde `STR_ItemInstance` sin propagar el campo nuevo.

**Próximo paso de diagnóstico:**
Inspeccionar cómo se construye el `STR_ItemData` que llega al pin `Item` de
`EquipmentChanged` — específicamente si se copia desde `STR_ItemInstance` y
si el campo `ItemStates` está siendo propagado correctamente.

---

## Bug 2 — Segundo slot de runa no aplica atributos de equipamiento

**Síntoma:** Al desbloquear el segundo slot de runa (HeadRuneSlot_1) y colocar
la misma runa, los `EquipmentAttributes` no se aplican. Solo aparece el
debug log "No es runa" en pantalla.

**Contexto:** El primer slot funciona correctamente — suma y resta atributos.
El segundo slot falla en aplicar atributos y muestra el mensaje de debug
"No es runa" que es un Print String temporal en `UI_ItemSlot.OnDrop`.

**Hipótesis:** El sistema de detección de runas en `OnDrop` no está
reconociendo el slot como válido para runas, o el índice del segundo slot
no está siendo manejado correctamente en la cadena de validación.

**Estado:** ⏳ Pendiente — prioridad menor al Bug 1.
Requiere inspección de `UI_ItemSlot.OnDrop` y la lógica de detección
de runas por Gameplay Tag.

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
- Bug 1 pendiente de resolución 🔴
- Bug 2 pendiente documentado ⏳

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Confirmar diferencia STR_ItemInstance vs STR_ItemData en runtime | ⏳ Relacionado con Bug 1 |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | No destructivo. Origen en BP_AbilitySystemComponent. | ⚠️ Pendiente |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| Bug 1 — ItemStates siempre llega vacío en EquipmentChanged | Length = 0 confirmado. Probable desincronización entre STR_ItemInstance y STR_ItemData en runtime. Próximo paso: inspeccionar cómo se construye STR_ItemData. | 🔴 Alta prioridad |
| Bug 2 — Segundo slot de runa no aplica atributos | Debug log "No es runa" aparece. Probable problema en detección de Gameplay Tag en OnDrop de UI_ItemSlot. | ⏳ Prioridad menor |
| Print String "No es runa" debug temporal en UI_ItemSlot.OnDrop | Pendiente eliminar cuando se resuelva Bug 2 | ⏳ |

---

*Archivo actualizado — sesión Light Paradox (Bug 1 y Bug 2 documentados)*
*Cambios: Diagnóstico de Bug 1 completado hasta Length=0, Bug 2 registrado, próximos pasos definidos*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
