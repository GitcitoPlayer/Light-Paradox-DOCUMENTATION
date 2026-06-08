# 13 — Widget: UI_CraftingQueue
### Widget: UI_CraftingQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 — Cooldown RuneAssign)

---

## Contexto

`UI_CraftingQueue` es el widget que contiene y gestiona los slots visuales de la
cola de crafteo y de asignación de runas. Vive dentro de `UI_HUD` como widget hijo.
Contiene `CraftingQueueBox` para crafteo y `RuneQueueBox` para runas — ambos
paneles son independientes.

El label del widget fue cambiado de "Crafting Queue" a "Action Queue" para
reflejar que contiene múltiples tipos de acciones con tiempo.

---

## Hierarchy confirmado

```
[UI_CraftingQueue]
  └── CraftingQueueSizeBox
        └── [Border]
              └── [Vertical Box]
                    ├── [Text] "Action Queue"
                    ├── CraftingQueueBox   ← crafteo
                    └── RuneQueueBox       ← runas
```

---

## Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `CraftingQueueBox` | Panel Widget | Contenedor de UI_QueueBlueprint. Gestionado por UpdateCraftingQueue |
| `RuneQueueBox` | Panel Widget | Contenedor de UI_RuneAssignQueue. Gestionado por AddRuneToQueue |
| `QueueBlueprintWidgets` | Array de UI_QueueBlueprint | Referencias a widgets de crafteo instanciados |
| `RuneQueueWidgets` | Array de UI_RuneAssignQueue | Referencias a widgets de runa instanciados |
| `PendingTargetContainer` | BP_ContainerComponent (Object Reference) | Contenedor destino pendiente de asignación |
| `PendingTargetSlot` | Integer | Slot destino pendiente |
| `PendingSourceContainer` | BP_ContainerComponent (Object Reference) | Contenedor origen pendiente |
| `PendingSourceSlot` | Integer | Slot origen pendiente |

---

## Funciones confirmadas

### UpdateCraftingQueue
**Tooltip:** "Update crafting queue."
Rebuild completo de `CraftingQueueBox`. No toca `RuneQueueBox`.

**Inputs:** `Queue Blueprints` (Array de STR_QueueBlueprint)

**Flujo:**
```
Entry (Queue Blueprints) →
  Clear Children (Target: CraftingQueueBox) →
  CLEAR QueueBlueprintWidgets →
  For Each Loop (Array: Queue Blueprints)
    Loop Body →
      Create UI Queue Blueprint Widget (100x100) →
      Add Child (Target: CraftingQueueBox) →
      ADD → QueueBlueprintWidgets
    Completed → [termina]
```

### UpdateCraftingQueueBlueprint
**Tooltip:** "Update crafting queue blueprint by index."
Actualiza slot individual de crafteo por índice. No toca `RuneQueueBox`.

**Inputs:** `Index` (Integer), `Queue Blueprint` (STR_QueueBlueprint)

### AddRuneToQueue — flujo implementado

**Inputs:**

| Nombre | Tipo |
|---|---|
| `RuneIcon` | Texture2D (Object Reference) |
| `AssignDuration` | Float |
| `TargetContainer` | BP_ContainerComponent (Object Reference) |
| `TargetSlot` | Integer |
| `SourceContainer` | BP_ContainerComponent (Object Reference) |
| `SourceSlot` | Integer |

**Flujo:**

```
Entry →
  SET PendingTargetContainer ← TargetContainer
  SET PendingTargetSlot ← TargetSlot
  SET PendingSourceContainer ← SourceContainer
  SET PendingSourceSlot ← SourceSlot
  → Create Widget (Class: UI_RuneAssignQueue)
  → Init Rune Assign Queue (todos los inputs)
  → Add Child (Target: RuneQueueBox, Content: Return Value)
  → ADD Return Value → RuneQueueWidgets
  → Set Timer by Event
      Time: AssignDuration
      Looping: False
      Event: Create Event → OnRuneAssignComplete
```

---

## Event Graph — Custom Events

### OnRuneAssignComplete
**Firma:** sin parámetros — requerido por Set Timer by Event.
**Estado:** ⏳ Pendiente de implementar.

**Flujo planificado:**
```
OnRuneAssignComplete →
  Try Move Item To Container Slot BPI
    Target: Get Owning Player
    From Container: GET PendingSourceContainer
    From Slot: GET PendingSourceSlot
    To Container: GET PendingTargetContainer
    To Slot: GET PendingTargetSlot
    Amount: -1
  → GET RuneQueueWidgets[0]
  → Remove From Parent
  → RuneQueueWidgets → Remove Index (0)
```

> **Nota:** Index 0 es temporal. Cuando haya múltiples runas en queue
> simultáneamente necesitará lógica para identificar cuál widget
> corresponde al timer que disparó.

---

## Relación con UI_HUD

`UI_CraftingQueue` es accedido desde `UI_HUD` vía:
- `UpdateCraftingQueue_BPI` → delega a `UpdateCraftingQueue`
- `UpdateCraftingQueueBlueprint_BPI` → delega a `UpdateCraftingQueueBlueprint`

`AddRuneToQueue` será llamada desde `UI_ItemSlot` → `OnDrop` cuando el ítem
dropeado sea una runa. Pendiente de implementar la intercepción.

---

## Notas de arquitectura

- `CraftingQueueBox` y `RuneQueueBox` son paneles independientes.
  El rebuild de crafteo nunca afecta los widgets de runa.
- El timer vive en `UI_CraftingQueue` vía `Set Timer by Event`.
  `UI_RuneAssignQueue` no tiene timer propio.
- `Set Timer by Event` no puede pasar parámetros al evento.
  Los datos pendientes se guardan en variables de instancia
  (`PendingTargetContainer`, etc.) antes de arrancar el timer.
- `StartRuneTimer` fue evaluado y descartado — el timer vive directamente
  en `AddRuneToQueue`, más limpio.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| OnRuneAssignComplete sin implementar | Flujo planificado documentado arriba | ⏳ Pendiente |
| Index 0 hardcodeado en OnRuneAssignComplete | Necesita lógica dinámica para queue múltiple | Pendiente — post Lógica 1 |
| Interceptar OnDrop en UI_ItemSlot | Punto de entrada del cooldown desde el drag | ⏳ Pendiente |
| Bind_Time_Text en UI_RuneAssignQueue | Countdown visual pendiente | ⏳ Pendiente |
| BtnCancel en UI_RuneAssignQueue | Cancelar timer y devolver runa | ⏳ Pendiente |

---

*Archivo actualizado — sesión Light Paradox (Lógica 1 — AddRuneToQueue implementado, OnRuneAssignComplete pendiente)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
