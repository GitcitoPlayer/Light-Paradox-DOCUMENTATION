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
| `IsHitEffect` | Boolean | Propiedad de diseño. True = Estado ofensivo, se aplica al golpear. False = Estado pasivo, se aplica al portador o por evento. No es un filtro de código — es una convención de diseño que condiciona qué flujo del enemigo lo ejecuta. |
| `Rate` | Float (0–100) | Probabilidad porcentual de que el Estado se ejecute al activarse. 100 = siempre se aplica. |
| `Duration` | Float | Duración en segundos. Usado cuando EffectsDurationOverride = True. 0 = infinito. |
| `EffectsDurationOverride` | Boolean | True = usar Duration de este Estado para todos los efectos, ignorando sus duraciones individuales en DT_StatusEffects. False = cada efecto usa su propia Duration. |
| `IsPermanent` | Boolean | True = el Estado no expira. Time Remaining se pasa como 0.0. Persiste hasta que la fuente lo cancele explícitamente. Ignora Duration y EffectsDurationOverride. |
| `AffectedFactions` | Array de E_Faction | Facciones que pueden recibir este Estado. Si el array está vacío = afecta a cualquier actor. |

### Semántica de IsHitEffect confirmada

| Contexto | IsHitEffect = False | IsHitEffect = True |
|---|---|---|
| **Runa equipada** | Efecto se aplica al jugador portador | Efecto se aplica a lo que el jugador golpea |
| **Enemigo** | Efecto se aplica al propio enemigo en Init Character | Efecto se aplica al actor golpeado en Trace Deal Damage |
| **Trigger de zona** | Efecto se aplica al actor que entra | N/A — triggers usan False por diseño |

> **Nota crítica:** `IsHitEffect` no es evaluado dentro de `BP_StateApplier`.
> Es una propiedad de diseño que el Blueprint del enemigo (u otra fuente) lee
> para decidir en qué momento llamar a `ApplyState` y con qué Target.

### Notas de diseño

- Un Estado con `Duration = 0` y `EffectsDurationOverride = True` produce efectos infinitos.
- `IsPermanent = True` tiene prioridad sobre `EffectsDurationOverride` y `Duration`.
- `AffectedFactions` vacío = sin restricción — afecta a cualquier actor.
- Múltiples efectos en un Estado se aplican simultáneamente al mismo target.

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
                        Break STR_StatusEffectInstance → Duration
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
                  Select 1 (Float)
                    False → Duration de STR_StatusEffectInstance
                    True  → Duration de STR_StateData
                    Index → EffectsDurationOverride
                  Select 2 (Float)
                    False → Return Value de Select 1
                    True  → 0.0
                    Index → IsPermanent
                  Get Ability System Component (de As BP Character Base)
                  Make STR_SaveData_StatusEffect
                    Effect Handle:       Array Element
                    Instigator Is Owner: False
                    Stack:               1
                    Time Remaining:      Return Value de Select 2
                  Load Status Effect
                    Target:    Ability System Component
                    Save Data: STR Save Data Status Effect
                  [exec de Load Status Effect sin conexión — loop avanza solo]

                Completed → Return Node
```

### Notas de implementación

- `AffectedFactions` es tipo `Array de E_Faction` — `CONTAINS` compara valores de enum directamente.
- El exec de salida de `Load Status Effect` va sin conexión — el loop avanza automáticamente.
- Conectar el exec de vuelta al loop causa ciclo infinito y congela el editor.
- `Enum to Name` devuelve `NewEnumerator0` — NO usar. Usar `Enum to String`.

---

## E_Faction — confirmado

Variable en `BP_Character_Base` — categoría Settings/AI — tipo `E_Faction`.

| Display Name |
|---|
| `Player` |
| `Undead` |
| `SmallAnimals` |
| `Human` |
| `Chickens` |
| `Pigs` |

---

## Integración con enemigos — BP_Character_Undead2

### Variable BaseState

| Propiedad | Valor |
|---|---|
| Nombre | `BaseState` |
| Tipo | `DataTableRowHandle` |
| Instance Editable | ✅ |
| Category | `Customization` |

### Init Character

```
[después de Create Mark]
  → Get Data Table Row (BaseState)
      Row Not Found → Return Node
      Row Found →
          Break STR_StateData → IsHitEffect
          → Branch
              True  → Return Node
              False →
                Item Handle Is Valid (BaseState)
                → Branch
                    False → Return Node
                    True  → Apply State (Target: Self) → Return Node
```

### Trace Deal Damage

```
[después de Apply Advanced Point Damage]
  → Cast To BP_Character_Base (Damaged Actor)
      Cast Failed → Play Sound at Location
      Then →
        Get Data Table Row (BaseState)
          Row Found →
              Break STR_StateData → IsHitEffect
              → Branch
                  False → [sin acción]
                  True  → Apply State (Target: As BP Character Base)
```

> **Nota crítica:** `Cast Failed` debe ir a `Play Sound at Location` — conectarlo
> a Apply State causa error de garbage collection en BP_AbilitySystemComponent.

---

## Triggers de prueba

### StateTrigger

```
On Component Begin Overlap (Box)
  → Cast To BP_Character_Base (Other Actor)
      Cast Failed → [termina]
      Then → Apply State → Destroy Actor (self)
```

### BP_SlowTrigger
Mismo patrón. Row Name: `State_Freeze`.

---

## Arquitectura de ítems de ESRPGv5 — documentado en Fase 3

### STR_ItemInstance vs STR_ItemData

`DT_Items` usa `STR_ItemInstance` como Row Struct.
`STR_ItemData` es la representación del ítem en runtime.

| Campo | STR_ItemInstance | STR_ItemData |
|---|---|---|
| Todos los campos base | ✅ | ✅ |
| `Amount` | ❌ | ✅ |
| `Charges` | ❌ | ✅ |
| `Durability` | ❌ | ✅ |
| `Decay` | ❌ | ✅ |

> **⚠️ Inferencia:** `STR_ItemInstance` es la definición estática en DT_Items.
> `STR_ItemData` es la representación runtime con valores dinámicos.
> Requiere inspección futura para confirmar dónde se construye `STR_ItemData`
> desde `STR_ItemInstance`.

El campo `ItemStates` fue agregado a **ambos structs**.

### Dos rutas de funcionamiento para ítems usables

#### Ruta A — Handles → DT_Abilities → DT_StatusEffects
**Ejemplo:** `Component_Corn`

```
DT_Items (Component_Corn)
  Handles → "Ability" → DT_Abilities / Ability_Corn
    AbilityClass: BP_Ability_Base
    Handles → "StatusEffect" → DT_StatusEffects / Effect_Corn
      → Aplica efecto por duración (Health + Hunger por segundo)
```

- Usa `BP_Ability_Base` como clase de habilidad
- El efecto se aplica a lo largo del tiempo via Status Effect
- Configurable completamente desde DataTables sin tocar Blueprints

#### Ruta B — UseAbilityClass directo
**Ejemplo:** `Usable_RedDrink`

```
DT_Items (Usable_RedDrink)
  UseAbilityClass: BP_Ability_ChangeState_RedDrink
    Change State Instantly:
      Health: 0.25 (25% del MaxHealth)
      Health Percent: True
```

- Usa una clase de habilidad específica con lógica hardcodeada
- El efecto es instantáneo — modifica atributos directamente sin Status Effect
- Requiere un Blueprint por cada variante de comportamiento

> **Nota:** Ninguna de las dos rutas incluye mecanismo de cancelación de efectos.
> ESRPGv5 no tiene ítems que remuevan Status Effects activos.

---

## Sistema de atributos de equipamiento — ESRPGv5

### Cómo funciona EquipmentAttributes en ítems cosméticos

Cuando el jugador equipa un ítem (incluyendo runas), `BP_Character_Player`
ejecuta la función `EquipmentChanged` que sigue este flujo:

```
EquipmentChanged (Equipment Slot, Slot, Item)
  → SET Local Item
  → SET Local Equipment Slot
  → Switch on E_EquipmentSlot
      → Actualiza mesh correspondiente (Head, Body, Pants, Hands, Feet,
        Backpack, Tool, HeadRuneWord, BodyRuneWord, etc.)
  → Update Footstep Settings
  → Get Equipment Items (Equipment Container)
      → Equiped Items
      → Exclude Broken Items
          → Result Items
          → Get Items Equipment Attributes
              → Equipment Attributes (array combinado de TODOS los ítems equipados)
              → Update Equipment Attributes (Target: Attributes Component)
```

**Patrón clave:** ESRPGv5 recalcula **todos** los atributos de **todos** los ítems
equipados en cada cambio — no solo el ítem que cambió. Esto evita
desincronización acumulativa.

### Punto de entrada para Estados de runas (Fase 4)

La lógica de Estados de runas debe agregarse **después de `Update Equipment Attributes`**
en `EquipmentChanged`, siguiendo el mismo patrón de "recalcular todo":

```
[después de Update Equipment Attributes]
  → Clear States de runas (remover todos los efectos activos de runas)
  → Get Equipment Items → Exclude Broken Items
  → For Each Loop (Result Items)
      → Get Data Table Row (Item.ItemHandle → DT_Items)
          → Break STR_ItemInstance → ItemStates
          → For Each Loop (ItemStates)
              → Apply State (Array Element, Target: Self)
```

> **⚠️ Pendiente:** Confirmar si `ClearStatusEffects` en `BP_AbilitySystemComponent`
> borra absolutamente todos los efectos o permite filtrar por fuente. Si borra todo,
> necesitamos un mecanismo alternativo de tracking de efectos de runas.

### Dispatchers y eventos relevantes en BP_EquipmentComponent

| Dispatcher / Evento | Descripción |
|---|---|
| `OnEquipmentChanged` | Se dispara cuando cambia el equipamiento |
| `OnSlotItemChanged_Event` | Custom Event — recibe `Slot` (Integer) e `Item` nuevo |

`OnSlotItemChanged_Event` solo pasa el ítem nuevo — no el anterior.
Por esto el patrón de "recalcular todo" es preferible al de "remover anterior / aplicar nuevo".

---

## Mecanismo de remoción de Status Effects — confirmado

Asset: `BP_AbilitySystemComponent`

| Función | Descripción | Uso para runas |
|---|---|---|
| `ClearStatusEffects` | Elimina todos los efectos activos del owner | ⚠️ Pendiente confirmar si filtra por fuente |
| `RemoveStatusEffectByHandle` | Elimina un efecto específico por su Handle | ✅ Correcto para remoción individual |
| `RemoveStatusEffectByID` | Elimina un efecto específico por su ID | 🟡 Alternativa posible |

### Flujo interno de RemoveStatusEffectByHandle

```
RemoveStatusEffectByHandle (Effect Handle)
  → SET Local Status Effects = Active Status Effects
  → For Each Loop (Local Status Effects)
      → Break STR_StatusEffectInstance → Handle
      → Handles Are Equals (Handle1: Array Element, Handle2: Effect Handle)
          → Branch
              True  → Destroy Actor → Return Node (Success: true)
              False → [continúa loop]
  Completed → Return Node (Success: false)
```

**Input:** `Effect Handle` — DataTableRowHandle apuntando a fila de `DT_StatusEffects`

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

**Decisión arquitectural confirmada:**
- Punto de entrada: función `EquipmentChanged` en `BP_Character_Player`
- Patrón: "recalcular todo" — igual que `Update Equipment Attributes`
- Pendiente: confirmar comportamiento de `ClearStatusEffects` antes de construir

**Flujo objetivo:**
```
EquipmentChanged
  → [cadena existente]
  → Update Equipment Attributes  ← ya existe
  → Clear States de runas        ← nuevo
  → For Each Loop (ítems equipados)
      → For Each Loop (ItemStates del ítem)
          → Apply State (Target: Self)  ← nuevo
```

---

## Pendientes fuera de alcance actual

| Caso de uso | Estado |
|---|---|
| BaseState en BP_Character_Base | Evaluar migración para todos los personajes | ⏳ |
| Estados en consumibles | Pospuesto — uso ambiguo | ⏳ |
| Cancelación de efectos desde contextos arbitrarios | Demasiado abierto | ⏳ |
| Confirmar diferencia exacta STR_ItemInstance vs STR_ItemData | Requiere inspección del flujo runtime | ⏳ |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Error de stack duplicado en Load Status Effect | "BP_StatusEffect_TickDamage_C_1 is not valid" — ocurre cuando el efecto ya está activo y se aplica de nuevo. No destructivo. Origen en BP_AbilitySystemComponent. | ⚠️ Pendiente |
| BaseState solo en BP_Character_Undead2 | Evaluar migración a BP_Character_Base | ⏳ |
| ClearStatusEffects sin inspeccionar | Crítico para Fase 4 — confirmar si filtra por fuente o borra todo | ⏳ Próxima sesión |
| Diferencia STR_ItemInstance vs STR_ItemData sin confirmar | Inferencia documentada — requiere inspección del flujo runtime | ⏳ |
| ItemStates agregado a ambos structs | Confirmar cuál es el correcto una vez resuelta la ambigüedad | ⏳ |

---

*Archivo actualizado — sesión Light Paradox (Fase 4 en progreso)*
*Cambios: Sistema EquipmentChanged documentado, patrón de recalcular todo confirmado, punto de entrada para Estados de runas identificado, ClearStatusEffects pendiente de inspección*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
