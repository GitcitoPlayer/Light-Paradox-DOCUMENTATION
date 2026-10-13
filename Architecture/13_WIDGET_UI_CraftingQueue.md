# 13 — Widget: UI_CraftingQueue
### Widget: UI_CraftingQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 rediseño + Lógica 4)
### Actualizado: sesión Light Paradox — Orden de remoción de runas (contradicción IsBlocked/bIsLocked RESUELTA + bug nuevo)

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
  → GET LockedSlot → SET IsBlocked = True          ← ✅ CORREGIDO (antes decía bIsLocked)
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

> **✅ Contradicción RESUELTA (sesión Orden de remoción de runas):** el log de runtime
> nombraba el nodo como `Set IsBlocked` mientras este .md decía `SET bIsLocked`. El usuario
> verificó en el editor los nodos reales de `AddRuneToQueue` **y** de `OnRuneAssignComplete`:
> **ambos usan `IsBlocked`**. La documentación estaba equivocada. `bIsLocked` es una variable
> huérfana en `UI_ItemSlot` (sin Get ni Set en el proyecto).
>
> **Consecuencia de diseño (deuda registrada):** el cooldown de runas depende de `IsBlocked`,
> variable compartida con repair/use del asset base. Es frágil. No se corrige ahora porque el
> sistema funciona; ver Deuda técnica.

---

## Event Graph — Custom Events

### OnRuneAssignComplete ✅
**Firma:** sin parámetros — requerido por Set Timer by Event.
**Estado:** Implementado — rediseñado en sesión Lógica 1 rediseño.

**Flujo:**
```
OnRuneAssignComplete →
  GET PendingLockedSlot → SET IsBlocked = False     ← ✅ CORREGIDO (antes decía bIsLocked)
  → GET PendingLockedSlot → GET LockIcon → Set Visibility (Collapsed)
  → GET RuneQueueWidgets → Get [0] → Remove From Parent
  → RuneQueueWidgets → Remove Index (0)
```

> **Nota:** Index 0 es temporal. Cuando haya múltiples runas en queue
> simultáneamente necesitará lógica para identificar cuál widget
> corresponde al timer que disparó.

> **Interacción con `bRemovalLocked` (sesión Orden de remoción de runas):** este evento
> escribe `IsBlocked`, nunca `bRemovalLocked`. Son gates independientes, así que el fin de un
> cooldown no puede desbloquear un slot que debe seguir bloqueado por orden de remoción.
> El velo (`RemovalLockOverlay`) y el candado (`LockIcon`) son widgets distintos.

---

## Diseño del sistema de runas — decisiones confirmadas

| Decisión | Detalle |
|---|---|
| El cooldown es para desbloquear el siguiente slot | No para asignar la runa — la runa se asigna inmediatamente en éxito |
| El widget en Action Queue muestra un candado | No el ícono de la runa |
| BtnCancel deshabilitado | El cliente confirmó que no se necesita cancelación |
| Éxito → bloquea slot SIGUIENTE | El slot que se desbloqueará al terminar el cooldown |
| Fallo → bloquea slot ACTUAL | Penalización — el jugador debe esperar el cooldown para reintentar |
| Remoción de runas en orden descendente | Implementado aparte con `bRemovalLocked` (ver `04` y `08`), independiente del cooldown |

---

## Relación con UI_HUD

`UI_CraftingQueue` es accedido desde `UI_HUD` vía:
- `UpdateCraftingQueue_BPI` → delega a `UpdateCraftingQueue`
- `UpdateCraftingQueueBlueprint_BPI` → delega a `UpdateCraftingQueueBlueprint`

`AddRuneToQueue` es llamada desde `UI_ItemSlot` → `OnDrop` cuando el ítem
dropeado es detectado como runa via `Does Container Match Tag Query`.

> **✅ Actualizado — sesión Fix candados fantasma:** `OnDrop` solo llega a llamar
> `AddRuneToQueue` si además de `Does Container Match Tag Query` se cumple
> `IsValidRuneDestination` y el `Container` del slot pasa `Cast To BP_EquipmentComponent`.
> Ver `04_BLUEPRINT_UI_ItemSlot.md`, sección OnDrop.
>
> **⚠️ Limitación conocida:** esa validación **no comprueba que el slot destino esté vacío**
> — ver bug abierto abajo.

---

## Bug — Candados fantasma en Action/Crafting Queue ✅ RESUELTO (en el origen)

### Síntoma
Al soltar una Rune Word en cualquier `UI_ItemSlot` que no fuera un slot de runa válido
(slots de inventario, hotbar, mochila, o incluso otros slots de equipamiento como Head/Body/
Pants dentro del propio `EquipmentContainer`), se acumulaban íconos de candado en el
Action/Crafting Queue que nunca se quitaban, aunque su contador llegara a 0.

### Console errors observados
```
Blueprint Runtime Error: "Accessed None trying to read property LockedSlot".
  Node: Set IsBlocked        Graph: AddRuneToQueue   Function: Add Rune to Queue   Blueprint: UI_CraftingQueue
Blueprint Runtime Error: "Accessed None trying to read property LockedSlot".
  Node: Set Visibility       Graph: AddRuneToQueue   Function: Add Rune to Queue   Blueprint: UI_CraftingQueue
Blueprint Runtime Error: "Accessed None".
  Node: Set Visibility       Graph: AddRuneToQueue   Function: Add Rune to Queue   Blueprint: UI_CraftingQueue
Blueprint Runtime Error: "Accessed None trying to read property PendingLockedSlot".
  Node: Set IsBlocked        Graph: EventGraph       Function: Execute Ubergraph UI Crafting Queue   Blueprint: UI_CraftingQueue
Blueprint Runtime Error: "Accessed None trying to read property PendingLockedSlot".
  Node: Set Visibility       Graph: EventGraph       Function: Execute Ubergraph UI Crafting Queue   Blueprint: UI_CraftingQueue
Blueprint Runtime Error: "Accessed None".
  Node: Set Visibility       Graph: EventGraph       Function: Execute Ubergraph UI Crafting Queue   Blueprint: UI_CraftingQueue
```

### Causa raíz confirmada
El check "es runa" en `UI_ItemSlot.OnDrop` solo evaluaba el tag del ítem arrastrado, nunca el
slot destino. Con `SuccessChance` default `100` en cualquier slot, el flujo ÉXITO/FALLO
siempre entraba en ÉXITO y llamaba a `GetNextRuneSlot(CurrentSlotNumber: GET SlotNumber)`
(`08_BLUEPRINT_UI_Character.md`), cuyo `Select` solo tiene casos para `7, 14–22`. Para
cualquier otro `SlotNumber`, el `Select` devuelve `None`, que se pasa como `LockedSlot` a
`AddRuneToQueue` — de ahí los `Accessed None` sobre `LockedSlot`/`PendingLockedSlot`. El
widget `UI_RuneAssignQueue` igual se creaba y se agregaba a `RuneQueueBox`, quedando huérfano.

Un caso relacionado: el problema también ocurría con `SlotNumber` dentro del
`EquipmentContainer` pero en un slot no-runa (Head=0, Body=1, etc.), y con inventarios
ampliados por mochila, donde el índice de un slot de inventario podía coincidir
numéricamente con el rango 14–22 — confirmando que `SlotNumber` es relativo al `Container`
que lo contiene, no un ID global.

### Fix aplicado
En `UI_ItemSlot.OnDrop`, antes de evaluar `Does Container Match Tag Query`, se agregó:
1. `Cast To BP_EquipmentComponent` sobre `GET Container`.
2. `IsValidRuneDestination` — `OR` de `SlotNumber == 7` y `(SlotNumber >= 14 AND SlotNumber <= 22)`.
3. Ambos combinados con `AND` junto a `Does Container Match Tag Query`.

**Estado: ✅ Resuelto en el origen.** Documentación completa: `04_BLUEPRINT_UI_ItemSlot.md`.

---

## 🐞 Bug abierto — Runa soltada sobre slot de runa YA OCUPADO dispara cooldown y candado

**Detectado:** sesión Orden de remoción de runas. **Estado: 🔴 Abierto — sin fix aplicado.**

**Reproducción (confirmada por el usuario):** slot 1 con runa y velo, slot 2 con runa asignada,
slot 3 disponible. Arrastrar una runa del inventario y soltarla sobre el slot 2 → se dispara el
cooldown y aparece el candado (y su widget en `RuneQueueBox`), aunque la acción no tiene sentido.

**Diagnóstico (inferencia a partir de lo documentado):** lógica de `OnDrop` en `UI_ItemSlot`.
`IsValidRuneDestination` solo mira el número de slot, no si el slot ya tiene ítem. El slot 2
no está bloqueado por `bRemovalLocked` (su siguiente, el 3, está vacío), así que entra al
flujo ÉXITO/FALLO y llama `AddRuneToQueue` con `LockedSlot` válido (slot 3) — por eso no
genera errores de `Accessed None`: el candado es "legítimo" pero la acción no debería haber
ocurrido. `AddRuneToQueue` no se defiende (no valida nada, ver notas).

**Fix propuesto (no implementado):** en `OnDrop`, rama `True` de "es runa Y slot destino
válido", agregar un `Branch` con `NOT Item Is Valid(Get Item(Container, SlotNumber))`;
`False` → `Return Node (True)` sin acción. Detalle paso a paso y decisión de diseño pendiente
(rechazar vs reemplazar) en `04_BLUEPRINT_UI_ItemSlot.md`.

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
- `AddRuneToQueue` en sí **no valida** que `LockedSlot` sea válido ni que la acción tenga
  sentido — toda la validación vive en el llamador (`UI_ItemSlot.OnDrop`). Consistente con la
  prioridad de "modificación mínima", pero deja a esta función sin defensa propia ante futuros
  puntos de entrada — ver pendiente de Hardening.
- **Gates de bloqueo:** el cooldown usa `IsBlocked` (compartido con repair/use). El orden de
  remoción usa `bRemovalLocked` (propio, ver `04`). `bIsLocked` es huérfana.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Index 0 hardcodeado en OnRuneAssignComplete | Necesita lógica dinámica para queue múltiple | Pendiente — post Lógica 1 |
| AssignDuration hardcodeado a 5.0 | Debe hacerse configurable | ⏳ Pendiente |
| Pin RuneIcon sin uso en AddRuneToQueue | Pendiente limpiar tras rediseño | ⏳ Pendiente |
| Variables Pending* posiblemente sin uso | Evaluar si PendingTargetContainer/Slot/Source* siguen siendo necesarios | ⏳ Pendiente |
| Candados fantasma al soltar runa en slot no-runa | Fix en `UI_ItemSlot.OnDrop` | ✅ Resuelto en el origen |
| ⏳ Hardening opcional: `IsValid(LockedSlot)` en `AddRuneToQueue` | No obligatorio. Capa de seguridad: `IsValid(LockedSlot)` al inicio y `Return` si Is Not Valid | 🟡 Pendiente — baja prioridad |
| ✅ Verificar `Set IsBlocked` vs `Set bIsLocked` en `AddRuneToQueue` | **Resuelto:** ambos nodos reales (`AddRuneToQueue` y `OnRuneAssignComplete`) usan `IsBlocked`. Documentación corregida | ✅ Cerrado esta sesión |
| **Cooldown de runas depende de `IsBlocked` (frágil)** | `IsBlocked` también pertenece a repair/use del asset base. Evaluar migrar el cooldown a variable propia (posible reutilizar la huérfana `bIsLocked` o crear una nueva) y resolver junto con la variable huérfana | ⏳ **Nuevo** — pendiente, prioridad media |
| **`bIsLocked` huérfana en `UI_ItemSlot`** | Sin Get ni Set. Borrar o reutilizar | ⏳ **Nuevo** |
| **🐞 Runa sobre slot ocupado dispara cooldown/candado** | Ver sección dedicada. Fix propuesto en `OnDrop` | 🔴 **Nuevo — abierto** |

---

*Archivo actualizado — sesión Light Paradox (Orden de remoción de runas)*
*Cambios: contradicción `IsBlocked`/`bIsLocked` resuelta (ambos nodos reales usan `IsBlocked`), flujos de `AddRuneToQueue` y `OnRuneAssignComplete` corregidos; interacción con `bRemovalLocked` documentada; deuda del cooldown frágil y variable huérfana registradas; bug nuevo de runa sobre slot ocupado registrado.*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
