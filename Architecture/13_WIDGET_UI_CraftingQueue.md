# 13 — Widget: UI_CraftingQueue
### Widget: UI_CraftingQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 rediseño + Lógica 4)

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
                    └── RuneQueueBox       ← runas/candados
```

---

## Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `CraftingQueueBox` | Panel Widget | Contenedor de UI_QueueBlueprint. Gestionado por UpdateCraftingQueue |
| `RuneQueueBox` | Panel Widget | Contenedor de UI_RuneAssignQueue. Gestionado por AddRuneToQueue |
| `QueueBlueprintWidgets` | Array de UI_QueueBlueprint | Referencias a widgets de crafteo instanciados |
| `RuneQueueWidgets` | Array de UI_RuneAssignQueue | Referencias a widgets de candado instanciados |
| `PendingLockedSlot` | UI_ItemSlot (Object Reference) | Slot que tiene el candado activo durante el cooldown |
| `PendingTargetContainer` | BP_ContainerComponent (Object Reference) | ⚠️ Pendiente evaluar si sigue siendo necesario tras rediseño |
| `PendingTargetSlot` | Integer | ⚠️ Pendiente evaluar si sigue siendo necesario |
| `PendingSourceContainer` | BP_ContainerComponent (Object Reference) | ⚠️ Pendiente evaluar si sigue siendo necesario |
| `PendingSourceSlot` | Integer | ⚠️ Pendiente evaluar si sigue siendo necesario |

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

### AddRuneToQueue — flujo implementado ✅

**Inputs:**

| Nombre | Tipo | Notas |
|---|---|---|
| `RuneIcon` | Texture2D (Object Reference) | ⚠️ Sin uso tras rediseño — pendiente limpiar |
| `AssignDuration` | Float | Duración del cooldown |
| `TargetContainer` | BP_ContainerComponent (Object Reference) | Contenedor destino |
| `TargetSlot` | Integer | Slot destino |
| `SourceContainer` | BP_ContainerComponent (Object Reference) | Contenedor origen |
| `SourceSlot` | Integer | Slot origen |
| `LockedSlot` | UI_ItemSlot (Object Reference) | Slot a bloquear con candado durante el cooldown |

**Flujo:**

```
Entry →
  SET PendingLockedSlot ← LockedSlot
  SET PendingTargetContainer ← TargetContainer
  SET PendingTargetSlot ← TargetSlot
  SET PendingSourceContainer ← SourceContainer
  SET PendingSourceSlot ← SourceSlot
  → GET LockedSlot → SET bIsLocked = True
  → GET LockedSlot → GET LockIcon → Set Visibility (Visible)
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

### OnRuneAssignComplete ✅
**Firma:** sin parámetros — requerido por Set Timer by Event.
**Estado:** Implementado — rediseñado en sesión Lógica 1 rediseño.

**Flujo:**
```
OnRuneAssignComplete →
  GET PendingLockedSlot → SET bIsLocked = False
  → GET PendingLockedSlot → GET LockIcon → Set Visibility (Collapsed)
  → GET RuneQueueWidgets → Get [0] → Remove From Parent
  → RuneQueueWidgets → Remove Index (0)
```

> **Nota:** Index 0 es temporal. Cuando haya múltiples runas en queue
> simultáneamente necesitará lógica para identificar cuál widget
> corresponde al timer que disparó.

---

## Diseño del sistema de runas — decisiones confirmadas

| Decisión | Detalle |
|---|---|
| El cooldown es para desbloquear el siguiente slot | No para asignar la runa — la runa se asigna inmediatamente en éxito |
| El widget en Action Queue muestra un candado | No el ícono de la runa |
| BtnCancel deshabilitado | El cliente confirmó que no se necesita cancelación |
| Éxito → bloquea slot SIGUIENTE | El slot que se desbloqueará al terminar el cooldown |
| Fallo → bloquea slot ACTUAL | Penalización — el jugador debe esperar el cooldown para reintentar |

---

## Relación con UI_HUD

`UI_CraftingQueue` es accedido desde `UI_HUD` vía:
- `UpdateCraftingQueue_BPI` → delega a `UpdateCraftingQueue`
- `UpdateCraftingQueueBlueprint_BPI` → delega a `UpdateCraftingQueueBlueprint`

`AddRuneToQueue` es llamada desde `UI_ItemSlot` → `OnDrop` cuando el ítem
dropeado es detectado como runa via `Does Container Match Tag Query`.

---

## Notas de arquitectura

- `CraftingQueueBox` y `RuneQueueBox` son paneles independientes.
  El rebuild de crafteo nunca afecta los widgets de runa.
- El timer vive en `UI_CraftingQueue` vía `Set Timer by Event`.
  `UI_RuneAssignQueue` no tiene timer propio.
- `Set Timer by Event` no puede pasar parámetros al evento.
  Los datos pendientes se guardan en variables de instancia antes de arrancar el timer.
- El move de la runa ocurre en `OnDrop` (caso éxito), no en `OnRuneAssignComplete`.
  `OnRuneAssignComplete` solo desbloquea el slot y limpia el widget.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Index 0 hardcodeado en OnRuneAssignComplete | Necesita lógica dinámica para queue múltiple | Pendiente — post Lógica 1 |
| AssignDuration hardcodeado a 5.0 | Debe hacerse configurable | ⏳ Pendiente |
| Pin RuneIcon sin uso en AddRuneToQueue | Pendiente limpiar tras rediseño | ⏳ Pendiente |
| Variables Pending* posiblemente sin uso | Evaluar si PendingTargetContainer/Slot/Source* siguen siendo necesarios | ⏳ Pendiente |

---

*Archivo actualizado — sesión Light Paradox (Lógica 1 rediseño + Lógica 4 probabilidad)*
*Cambios: OnRuneAssignComplete rediseñado para desbloquear slot, AddRuneToQueue actualizado con LockedSlot, PendingLockedSlot agregado, diseño del sistema documentado*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
