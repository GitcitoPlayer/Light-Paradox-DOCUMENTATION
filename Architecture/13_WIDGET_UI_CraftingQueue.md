# 13 — Widget: UI_CraftingQueue
### Widget: UI_CraftingQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 rediseño + Lógica 4)
### Actualizado: sesión Light Paradox — Fix candados fantasma en Action/Crafting Queue

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

> **⚠️ Contradicción detectada — sesión Fix candados fantasma, pendiente verificar en editor:**
> los console errors de runtime nombran el nodo como `Set IsBlocked`:
> ```
> Blueprint Runtime Error: "Accessed None trying to read property LockedSlot".
>   Node: Set IsBlocked  Graph: AddRuneToQueue  Function: Add Rune to Queue  Blueprint: UI_CraftingQueue
> ```
> pero la documentación de este flujo (arriba) describe ese mismo paso como
> `GET LockedSlot → SET bIsLocked = True`. El log de runtime es la fuente de mayor prioridad
> (es el comportamiento real compilado, no una inferencia de sesión anterior) — indica que el
> nodo real en el grafo podría estar seteando `IsBlocked` en vez de `bIsLocked` sobre
> `LockedSlot`. **No confirmado todavía.**
>
> **Por qué importa:** `IsBlocked` e `bIsLocked` son gates distintos y no intercambiables (ver
> `04_BLUEPRINT_UI_ItemSlot.md`, Notas de arquitectura): `IsBlocked` es para operaciones de
> repair/use, `bIsLocked` es específicamente el gate del sistema de runas que controla
> `LockIcon`. Si el nodo real está seteando `IsBlocked` en vez de `bIsLocked`, el candado visual
> (`LockIcon`) podría no estar sincronizado correctamente con el bloqueo real de drops en el
> slot, aunque el síntoma no sea evidente en todos los casos.
>
> **Acción pendiente:** abrir `UI_CraftingQueue → AddRuneToQueue` en el editor y confirmar el
> nombre exacto de la variable en el primer nodo `SET` después de `GET LockedSlot`. Si dice
> `IsBlocked`, corregirlo a `bIsLocked` (Compile + Save). Si dice `bIsLocked`, actualizar esta
> nota confirmando que el nombre del nodo en el log fue simplemente el nombre interno mostrado
> por Unreal para esa variable y no hay bug real aquí.

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

> **✅ Actualizado — sesión Fix candados fantasma:** desde esta sesión, `OnDrop` solo llega a
> llamar `AddRuneToQueue` si además de `Does Container Match Tag Query` se cumple
> `IsValidRuneDestination` (el slot destino es realmente un slot de runa dentro del
> `EquipmentContainer`) y el `Container` del slot pasa `Cast To BP_EquipmentComponent`. Ver
> `04_BLUEPRINT_UI_ItemSlot.md`, sección OnDrop, para el detalle completo del fix.

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
widget `UI_RuneAssignQueue` igual se creaba y se agregaba a `RuneQueueBox` (esos nodos no
dependen de `LockedSlot`), quedando huérfano — sin candado real que desbloquear al completar
el timer.

Un caso relacionado, descubierto durante el diagnóstico: el problema también ocurría con
`SlotNumber` dentro del `EquipmentContainer` pero en un slot no-runa (Head=0, Body=1, etc.),
y con inventarios ampliados por mochila, donde el índice de un slot de inventario podía
coincidir numéricamente con el rango 14–22 reservado a runas duplicadas de Head — confirmando
que `SlotNumber` es relativo al `Container` que lo contiene, no un ID global (ver nota en
`04_BLUEPRINT_UI_ItemSlot.md`).

### Fix aplicado
En `UI_ItemSlot.OnDrop`, antes de evaluar `Does Container Match Tag Query`, se agregó:
1. `Cast To BP_EquipmentComponent` sobre `GET Container` — descarta cualquier `Container` que
   no sea el `EquipmentContainer` (nivel Container).
2. `IsValidRuneDestination` — combinación `OR` de `SlotNumber == 7` y
   `(SlotNumber >= 14 AND SlotNumber <= 22)` — descarta slots del `EquipmentContainer` que no
   son de runa (nivel slot).
3. Ambos combinados con `AND` junto a `Does Container Match Tag Query` antes de entrar al
   flujo ÉXITO/FALLO.

Documentación completa del fix: `04_BLUEPRINT_UI_ItemSlot.md`, sección OnDrop.

**Estado: ✅ Resuelto en el origen** (el candado fantasma ya no se genera). Ver pendientes de
hardening abajo para una capa de seguridad adicional dentro de `AddRuneToQueue` mismo.

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
- **✅ Nuevo (Fix candados fantasma):** `AddRuneToQueue` en sí **no valida** que `LockedSlot`
  sea un valor válido antes de operar sobre él — la validación completa vive ahora únicamente
  en el llamador (`UI_ItemSlot.OnDrop`). Esto es consistente con la prioridad de "modificación
  mínima" del proyecto (el fix se aplicó en el punto exacto donde nace el problema, sin tocar
  `AddRuneToQueue`), pero deja a esta función sin defensa propia contra futuros puntos de
  entrada que pudieran pasar un `LockedSlot` inválido — ver pendiente de Hardening abajo.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Index 0 hardcodeado en OnRuneAssignComplete | Necesita lógica dinámica para queue múltiple | Pendiente — post Lógica 1 |
| AssignDuration hardcodeado a 5.0 | Debe hacerse configurable | ⏳ Pendiente |
| Pin RuneIcon sin uso en AddRuneToQueue | Pendiente limpiar tras rediseño | ⏳ Pendiente |
| Variables Pending* posiblemente sin uso | Evaluar si PendingTargetContainer/Slot/Source* siguen siendo necesarios | ⏳ Pendiente |
| **Candados fantasma al soltar runa en slot no-runa** | Causa raíz: `OnDrop` no validaba el slot destino, solo el tag del ítem. Fix aplicado en `UI_ItemSlot.OnDrop` (Cast To BP_EquipmentComponent + IsValidRuneDestination). Ver sección dedicada arriba y `04_BLUEPRINT_UI_ItemSlot.md`. | ✅ **Resuelto en el origen** esta sesión |
| **⏳ PENDIENTE 1 — Hardening opcional: `IsValid(LockedSlot)` en `AddRuneToQueue`** | No obligatorio para el bug reportado (el fix en `OnDrop` ya corta el problema de raíz). Se recomienda como capa de seguridad adicional: agregar `IsValid(LockedSlot)` al inicio de `AddRuneToQueue` y, si `Is Not Valid`, hacer `Return` antes de `SET PendingLockedSlot` / `Create Widget`. Esto evitaría candados huérfanos ante **cualquier** futuro punto de entrada a `AddRuneToQueue` que pase un `LockedSlot` inválido, sin depender de que el llamador siempre valide correctamente. | 🟡 Pendiente — no bloqueante, prioridad baja |
| **⏳ PENDIENTE 2 — Verificar contradicción `Set IsBlocked` vs `Set bIsLocked` en `AddRuneToQueue`** | Ver nota dedicada en la sección "AddRuneToQueue — flujo implementado" arriba. El log de runtime nombra el nodo `Set IsBlocked`; la documentación describe `SET bIsLocked = True`. Requiere abrir el grafo real y confirmar cuál variable se está seteando — el log tiene prioridad sobre el .md hasta esa verificación. Si el nodo real es `IsBlocked`, corregir a `bIsLocked` (Compile + Save) ya que son gates distintos y no intercambiables (ver `04_BLUEPRINT_UI_ItemSlot.md`). | 🔴 Pendiente verificación en editor — prioridad alta, puede afectar sincronización visual del candado |

---

*Archivo actualizado — sesión Light Paradox (Fix candados fantasma en Action/Crafting Queue)*
*Cambios: documentado el síntoma completo, causa raíz y fix del bug de candados fantasma
(resuelto en el origen, en UI_ItemSlot.OnDrop); registrados dos pendientes nuevos: (1) Hardening
opcional — IsValid(LockedSlot) en AddRuneToQueue como capa de seguridad adicional, no bloqueante;
(2) verificación de contradicción entre el log de runtime (Set IsBlocked) y la documentación
existente (Set bIsLocked) — pendiente de confirmar en el editor cuál es el nodo real.*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
