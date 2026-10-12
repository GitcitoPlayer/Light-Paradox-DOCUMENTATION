# 04 — Blueprint: UI_ItemSlot
### Widget: UI_ItemSlot
### Base Asset: EasySurvivalRPGv5
### Fuente: Exports .txt raw + inspección directa — sesión Light Paradox
### Actualizado: sesión Light Paradox — Orden de remoción de runas (bRemovalLocked) + correcciones de documentación

---

## ⚠️ Convención de nombres — cómo Unreal muestra las variables `bNombre`

Las variables Boolean nombradas con prefijo `b` (formato `bNombreNombre`) se muestran en
los nodos **Get / Set** y en las etiquetas de nodos sin la `b` y con espacios entre
palabras. Es solo presentación: el nombre real de la variable no cambia.

| Nombre real de la variable | Cómo aparece en el nodo Get/Set |
|---|---|
| `bRemovalLocked` | `Removal Locked` |
| `bUseItem` | `Use Item` |
| `bIsLocked` | `Is Locked` |

Usar siempre el nombre real (`bRemovalLocked`) al documentar y el nombre mostrado
(`Removal Locked`) al buscar en el editor. Confirmado por captura de `OnMouseButtonDown`.

---

## Variables relevantes (inferidas de los exports)

| Variable | Tipo | Notas |
|---|---|---|
| `ItemData` | STR_ItemData (struct) | Dato del ítem actual del slot |
| `Container` | BP_ContainerComponent | Referencia al contenedor del slot |
| `SlotNumber` | Integer | Índice del slot |
| `ItemSlotType` | E_ItemSlotType (enum) | Tipo de slot |
| `IsBlocked` | Boolean | Gate de drag/drop. Del asset base (repair/use). **También es la variable que realmente usa el cooldown de runas** (ver `13_WIDGET_UI_CraftingQueue.md`). Uso compartido = frágil. |
| `IsRightMouse` | Boolean | Si el último click fue botón derecho |
| `bUseItem` | Boolean | Flag de uso de ítem (Alexito's Lock) |
| `bUseItemChance` | Float (double) | Probabilidad de uso (Alexito's Chance) |
| `Counter` | Integer | Contador para delay |
| `MoveSound` | SoundBase | Sonido al mover ítem |
| `SplitSound` | SoundBase | Sonido al dividir stack |
| `BackgroundTexture` | Texture2D | Textura de fondo del slot |
| `BlackboardBrushTint` | SlateColor | Tint cuando slot está vacío |
| `CommonBrushTint` | SlateColor | Tint cuando slot tiene ítem |
| `DecayTimerHandle` | TimerHandle | Handle del timer de decay |
| `DecayTimerTick` | Float (double) | Intervalo del timer de decay |
| `Icon` | Image (UMG widget) | Widget imagen del ícono |
| `SuccessChance` | Integer | Probabilidad de éxito al asignar runa. Default 100. Asignado por slot en Designer. |
| `bRemovalLocked` | Boolean | ✅ **NUEVO.** True cuando el slot de runa está bloqueado por orden de remoción (el slot siguiente también tiene runa). Default False. Independiente de `IsBlocked`. Se escribe solo vía `SetRemovalLocked`. |
| `bIsLocked` | Boolean | 🗑️ **HUÉRFANA — sin ningún Get ni Set en el proyecto.** Confirmado por búsqueda en `UI_ItemSlot` y verificación de los nodos reales de `UI_CraftingQueue` (que usan `IsBlocked`). Ignorar; pendiente borrar o resolver junto con la deuda del cooldown frágil. |

> **⚠️ Nota crítica confirmada en sesión de fix de candados fantasma:** `SlotNumber` es un
> índice **local al Container que lo contiene** — no es un ID único global en todo el juego.
> `InventoryContainer` y `EquipmentContainer` indexan sus propios slots de forma independiente
> desde 0. Es esperado por diseño que, por ejemplo, el slot 8 del inventario y el slot 8 del
> `EquipmentContainer` (`BodyRuneWord`) compartan el mismo número. Cualquier lógica que
> compare `SlotNumber` sin primero confirmar de qué `Container` proviene corre el riesgo de
> tratar un slot de inventario como si fuera un slot de equipamiento (o viceversa). Ver
> sección "OnDrop" más abajo para el fix aplicado sobre este problema.

---

## Hierarchy de UI_ItemSlot (confirmado)

```
[UI_ItemSlot]
  └── [Size Box]
        └── ButtonItem
              └── [Overlay]
                    ├── [Background]
                    └── ItemOverlay
                          ├── Icon                 ← ícono del ítem
                          ├── RemovalLockOverlay   ← ✅ NUEVO: velo oscuro (Image) de bloqueo por orden
                          ├── [Amount]             ← cantidad del stack
                          └── LockIcon             ← candado de cooldown (Collapsed por defecto)
                    └── [Size Box]
                          └── DurabilityBar
                    └── [NeedRepairBox]
                          └── NeedRepairIcon
              └── Frame
```

> **Nota — `LockIcon`:** widget Image agregado en sesión Light Paradox (Lógica 1 rediseño).
> `Is Variable = True`. Visibility default = `Collapsed`. Su visibilidad la controlan
> `AddRuneToQueue` (Visible) y `OnRuneAssignComplete` (Collapsed) en `UI_CraftingQueue`.
> Textura: placeholder asignado en Designer — pendiente arte final.

> **Nota — `RemovalLockOverlay`:** widget Image, `Is Variable = True`, ubicado entre `Icon`
> y `[Amount]` (el orden del Hierarchy es el orden de capas). Slot Overlay: Horizontal y
> Vertical Alignment `Fill`. Color and Opacity: R0 G0 B0 **A 0.55**. Visibility default =
> `Collapsed`. Es un velo oscuro **sin ícono**, para distinguirlo del candado de cooldown
> (`LockIcon`). Visibilidad controlada únicamente por `SetRemovalLocked`.
> Si un Image sin textura no se dibujara, asignar textura blanca plana en Brush → Image
> (inferencia, no verificada — el velo se ve correctamente en juego según el usuario).

---

## Valores de SuccessChance por slot (Head)

| Slot | Widget | SuccessChance |
|---|---|---|
| 1 | `Equipment_HeadRuneSlot` | 100 |
| 2 | `Equipment_HeadRuneSlot_1` | 90 |
| 3 | `Equipment_HeadRuneSlot_2` | 80 |
| 4 | `Equipment_HeadRuneSlot_3` | 70 |
| 5 | `Equipment_HeadRuneSlot_4` | 60 |
| 6 | `Equipment_HeadRuneSlot_5` | 50 |
| 7 | `Equipment_HeadRuneSlot_6` | 40 |
| 8 | `Equipment_HeadRuneSlot_7` | 30 |
| 9 | `Equipment_HeadRuneSlot_8` | 20 |
| 10 | `Equipment_HeadRuneSlot_9` | 10 |

> **Nota:** Slot 1 siempre tiene 100% — nunca falla. Confirmado por el cliente.

---

## EventGraph

### PreConstruct
```
PreConstruct (IsDesignTime) →
  Branch (IsDesignTime == True)
    → True  → UpdateItemData (with current ItemData)
    → False → [nada]
```

### Construct
```
Construct →
  CallFunction: DelayAssignment →
  IsValid (Container) →
    Is Valid     → SetContainerReference (Container) →
    Is Not Valid → UpdateItemData (with current ItemData)
```

### OnMouseLeave
```
OnMouseLeave →
  Set IsRightMouse = False
```

### OnMouseButtonDown — ✅ documentado esta sesión (captura directa) + modificado
```
On Mouse Button Down (My Geometry, Mouse Event) →
  Branch (Condition ← NOT Removal Locked)                      ← ✅ NUEVO
    True  →
      SET Is Right Mouse ← Is Mouse Button Down
                             (Input: Mouse Event, Mouse Button: Right Mouse Button)
      → Branch (Condition ← Is Right Mouse)
          True  → Return Node (Return Value: Handled)
          False → Detect Drag if Pressed (Pointer Event: Mouse Event,
                                          Drag Key: Left Mouse Button)
                  → Return Node (Return Value: resultado de Detect Drag if Pressed)
    False → [sin conexión — termina sin hacer nada, devuelve valor por defecto (Unhandled)]
```

> **Efecto del cambio:** con el slot bloqueado por orden, `Is Right Mouse` nunca pasa a `True`
> y no se inicia `Detect Drag if Pressed`. La rama `False` sin conectar es intencional e
> inofensiva.
>
> **⏳ Pendiente de verificar:** el evento/función que **ejecuta** la acción de click derecho
> (mover/usar ítem) no está exportado. Si lee `Is Right Mouse`, el cambio de arriba ya lo
> cubre; si tiene otro camino, requiere su propio `Branch (NOT Removal Locked)`. Buscar con
> Ctrl+F `Is Right Mouse` y exportar ese grafo (probable `UI_ItemSlot OnMouseButtonUp.txt`).

### Custom Event: DelayAssignment
```
DelayAssignment →
  Delay (Duration = Counter convertido a float) →
  PrintString "X S Have Passed" [DevelopmentOnly] →
  Set bUseItem = True →
  UpdateItemData (with current ItemData) →
  PrintString "Item Updated" [DevelopmentOnly]
```

---

## SetRemovalLocked (Función) — ✅ NUEVA esta sesión

**Propósito:** activar/desactivar el bloqueo por orden de remoción y su velo visual.

**Input:** `bLocked` (Boolean)

```
Function Entry (bLocked) →
  SET bRemovalLocked ← bLocked →
  Set Visibility (Target: GET RemovalLockOverlay,
                  In Visibility ← Select (Index: bLocked)
                                    Option 0 (False) = Collapsed
                                    Option 1 (True)  = HitTestInvisible)
```

> `HitTestInvisible` garantiza que el velo nunca intercepte el mouse. El overlay es solo
> visual: el bloqueo real vive en las condiciones de `OnDragDetected`, `OnDrop` y
> `OnMouseButtonDown`.
> Esta función es llamada **solo** desde `UI_Character.UpdateRuneRemovalLocks`
> (ver `08_BLUEPRINT_UI_Character.md`).

---

## UpdateItemData
**Tooltip:** Update item data information and update icon and amount.

```
Entry (ItemData) →
  Set self.ItemData = ItemData →
  UpdateItemIcon →
  UpdateVisibility →
  UpdateItemAmount →
  UpdateItemDurability →
  Branch (ItemCanDecay(ItemData))
    → True  → SetTimerDelegate (Delegate: UpdateDecayTick, Time: DecayTimerTick, Looping: true) →
               Set DecayTimerHandle = ReturnValue
    → False → K2_ClearAndInvalidateTimerHandle (Handle: DecayTimerHandle)
```

---

## UpdateItemIcon
**Tooltip:** Update item icon.

```
Entry →
  Get ItemData, Get bUseItem
  Condition: (ItemIsValid(ItemData) AND bUseItem == True)
  Branch (Condition)
    → True  → Set LocalIcon = GetItemIcon(ItemData) →
               SetBrushFromTexture (Icon widget, LocalIcon, bMatchSize: true) →
               SetBrushTintColor (Icon widget, CommonBrushTint) →
               SetVisibility (Icon widget, Select(IsValid(LocalIcon): Visible / Hidden))

    → False → Select (IsValid(BackgroundTexture): Option True = BackgroundTexture / Option False = null)
               Set LocalIcon = SelectedTexture →
               SetBrushFromTexture (Icon widget, LocalIcon, bMatchSize: true) →
               SetBrushTintColor (Icon widget, BlackboardBrushTint) →
               SetVisibility (Icon widget, Select(IsValid(LocalIcon): Visible / Hidden))
```

---

## SetContainerReference
**Tooltip:** Set container reference.

```
Entry (Container) →
  Set self.Container = Container →
  IsValid (Container)
    → Is Not Valid → [termina]
    → Is Valid     → CheckEquipmentSlot →
                     FindContainerSlotSettings (Container, SlotIndex: SlotNumber) →
                     Branch (Success)
                       → False → [ambas ramas convergen en →]
                       → True  → IsValid (SlotSettings.Background)
                                    → Is Not Valid → [converge]
                                    → Is Valid     → Set BackgroundTexture = SlotSettings.Background
                     [converge] →
                     Set ItemData = Container.GetItem(SlotNumber) →
                     UpdateItemData (ItemData)
```

---

## OnDragDetected
**Tooltip:** Start drag operation if item is valid.

> ✅ **Modificado esta sesión:** la condición del `Branch` ahora incluye `NOT bRemovalLocked`
> (nodo `AND` ampliado a 3 pines con *Add pin*).

```
Entry (PointerEvent) →
  Get ItemData, Get bUseItem, Get IsBlocked, Get bRemovalLocked
  Condition: (ItemIsValid(ItemData) AND NOT IsBlocked AND NOT bRemovalLocked)   ← ✅ 3 pines
  Branch (Condition)
    → False → Return (Operation: null)
    → True  →
        GetItemAmount(ItemData) → Amount
        IsShiftDown(PointerEvent) → bShift
        IsControlDown(PointerEvent) → bControl
        bShift OR bControl → bSplitMode
        Amount > 1 → bHasMultiple

        LocalAmount = Select (bControl) / Select (bShift) / Select (bHasMultiple)

        Set LocalAmount = resultado →
        PlaySound2D (Select(bSplitMode: SplitSound / MoveSound)) →
        CreateWidget (UI_DraggedItem, ItemIcon: GetItemIcon(ItemData), ItemAmount: LocalAmount) →
        CreateDragDropOperation (BP_DraggedItem, DefaultDragVisual: UI_DraggedItem,
          ItemData, FromContainer: Container, FromSlot: SlotNumber,
          FromSlotType: ItemSlotType, Amount: LocalAmount) →
        Return (Operation: BP_DraggedItem)
```

---

## OnDrop
**Tooltip:** Try to move item to slot.

### Flujo completo con intercepción de runas y sistema de probabilidad

> **✅ Corrección de documentación (esta sesión):** versiones anteriores de este archivo
> decían que el primer `Branch` tras el Cast Success era `NOT IsBlocked AND NOT bIsLocked`.
> **Era incorrecto.** El grafo real solo tenía `NOT IsBlocked` (confirmado por inspección
> directa del usuario). `bIsLocked` nunca se leyó en este grafo — es una variable huérfana.
> El candado de cooldown bloquea drops a través de `IsBlocked`.
> Hoy el `Branch` es: `NOT IsBlocked AND NOT bRemovalLocked`.

> **✅ Fix candados fantasma (sesión anterior):** el check "es runa" validaba solo el tag del
> ítem arrastrado, nunca el slot destino. Se agregaron `Cast To BP_EquipmentComponent` e
> `IsValidRuneDestination`. Ver `13_WIDGET_UI_CraftingQueue.md`.

```
Entry (Operation, PointerEvent) →
  DynamicCast (Operation → BP_DraggedItem)
    → Cast Failed →
        DynamicCast (Operation → BP_AbstractItem)
          → Cast Failed → Return False
          → Cast Success →
              GetOwningPlayer →
              TryToAddAbstractItemToSlot_BPI → PlaySound2D → Return True

    → Cast Success →
        Branch (NOT IsBlocked AND NOT bRemovalLocked)          ← ✅ corregido / ampliado
          → False → [termina]
          → True  →
              bRepairAction = ItemIsRepairable(self.ItemData) AND ItemIsRepairKit(BP_DraggedItem.ItemData)
              bUseItemFlag = (bUseItem == True)

              Branch (bRepairAction AND bUseItemFlag)
                → True  → TryToRepairItem_BPI → PlaySound2D → Return True

                → False →
                    Cast To BP_EquipmentComponent (Object: GET Container)
                      Cast Failed →
                        TryMoveItemToContainerSlot_BPI →
                        Set bUseItem = (RandomInteger(100) <= bUseItemChance) →
                        DelayAssignment → PlaySound2D → Return True

                      Cast Success →
                        Does Container Match Tag Query
                          Tag Container: Get Item Tags (BP_DraggedItem.ItemData)
                          Tag Query: Any Tags Match →
                            EasyRPG.Items.Equipment.HeadRuneWord
                            EasyRPG.Items.Equipment.BodyRuneWord
                            EasyRPG.Items.Equipment.PantsRuneWord
                            EasyRPG.Items.Equipment.HandsRuneWord
                            EasyRPG.Items.Equipment.FeetRuneWord
                            EasyRPG.Items.Equipment.BackpackRuneWord
                            EasyRPG.Items.Equipment.ToolRuneWord
                        → AND con IsValidRuneDestination:
                              Equal (Integer): GET SlotNumber == 7          → IsBaseRuneSlot
                              >= (Integer): GET SlotNumber >= 14            → B2
                              <= (Integer): GET SlotNumber <= 22            → B3
                              AND (B2, B3)                                  → InDuplicateRuneRange
                              OR (IsBaseRuneSlot, InDuplicateRuneRange)     → IsValidRuneDestination
                        → AND (Does Container Match Tag Query, IsValidRuneDestination)
                    → Branch (es runa Y slot destino válido)
                        → False → TryMoveItemToContainerSlot_BPI →
                                  Set bUseItem = (RandomInteger(100) <= bUseItemChance) →
                                  DelayAssignment → PlaySound2D → Return True

                        → True →
                            Random Integer in Range (0..100) <= GET SuccessChance
                            → Branch (Éxito/Fallo)

                              ÉXITO (True) →
                                TryMoveItemToContainerSlot_BPI
                                  From Container: BP_DraggedItem.FromContainer
                                  From Slot: BP_DraggedItem.FromSlot
                                  To Container: GET Container
                                  To Slot: GET SlotNumber
                                  Amount: -1
                                → Get Owning Player → Get HUD → Cast To BP_HUD_Game
                                → GET HUD → Cast To UI_HUD → GET CharacterInformation
                                → GetNextRuneSlot (CurrentSlotNumber: GET SlotNumber)
                                → Get Owning Player → Get HUD → Cast To BP_HUD_Game
                                → GET HUD → Cast To UI_HUD → GET CraftingQueue
                                → AddRuneToQueue (
                                    AssignDuration: 5.0,
                                    TargetContainer: GET Container,
                                    TargetSlot: GET SlotNumber,
                                    SourceContainer: BP_DraggedItem.FromContainer,
                                    SourceSlot: BP_DraggedItem.FromSlot,
                                    LockedSlot: NextSlot)
                                → Return True

                              FALLO (False) →
                                → Get Owning Player → Get HUD → Cast To BP_HUD_Game
                                → GET HUD → Cast To UI_HUD → GET CraftingQueue
                                → AddRuneToQueue (
                                    AssignDuration: 5.0,
                                    LockedSlot: Get Reference To Self)
                                → Return True
```

> **Nota:** `AssignDuration` hardcodeado a `5.0` como valor de prueba.
> Debe hacerse configurable en sesión futura.

> **Nota:** `Print String "No es runa"` existe como debug temporal en el pin False
> del Branch de detección de runas. Pendiente eliminar.

> **Nota sobre escalabilidad del fix:** cuando se repliquen los slots de runa múltiple para
> Body/Pants/Hands/Feet/Backpack/Tool (ver `16_SYSTEM_RuneBinding_WeaponCosmetic.md`, Fase 1),
> el bloque `IsValidRuneDestination` necesitará un `OR` adicional por cada familia — mismo
> patrón usado en `CheckContainerSlotForItem` (`06_BLUEPRINT_BP_EquipmentComponent.md`).

---

## 🐞 Bug abierto — Runa soltada sobre un slot de runa YA OCUPADO dispara cooldown y candado

**Detectado:** sesión Light Paradox (Orden de remoción de runas). **Estado: 🔴 Abierto — sin fix aplicado.**

### Reproducción (confirmada por el usuario)
1. Slot 1 con runa (con velo `RemovalLockOverlay`), slot 2 con runa asignada, slot 3 disponible (vacío).
2. Arrastrar una runa del inventario y soltarla encima del **slot 2** (ocupado), no del slot 3.
3. Resultado: se dispara el flujo ÉXITO/FALLO → aparece el candado (`LockIcon`) y el widget de
   cooldown en Action Queue, aunque la acción no tiene sentido de diseño.

### Diagnóstico (inferencia razonada a partir del grafo documentado — no se inspeccionó el grafo en vivo)
- **Tipo de bug:** lógica de `OnDrop` — validación incompleta del slot destino.
- `IsValidRuneDestination` solo evalúa el **número** de slot (`7` o `14–22`). No evalúa si el
  slot destino **ya contiene un ítem**. El slot 2 es un slot de runa válido por número, y no
  está bloqueado por `bRemovalLocked` (su slot siguiente, el 3, está vacío), por lo que pasa
  `NOT IsBlocked AND NOT bRemovalLocked` y entra al flujo ÉXITO/FALLO.
- Con `SuccessChance` del slot 2 = 90, casi siempre entra en ÉXITO, llama `GetNextRuneSlot`
  (slot 3) y `AddRuneToQueue` → candado + cooldown.
- Este bug es **independiente** de `bRemovalLocked`: ocurre en cualquier slot de runa ocupado
  que no esté bloqueado por orden (el último runa asignado de la cadena).

### Fix propuesto (NO implementado — pendiente de aprobar y probar)
Dentro de `OnDrop`, rama `True` del `Branch (es runa Y slot destino válido)`, **antes** de
`Random Integer in Range`:

1. Crear `Get Item` (Target: `GET Container`, Slot: `GET SlotNumber`) → `Item Is Valid`.
2. Nodo `NOT` → nodo `Branch` (`Slot vacío`).
3. `True` (slot vacío) → continúa el flujo ÉXITO/FALLO existente sin cambios.
4. `False` (slot ocupado) → `Return Node (Return Value: True)` sin hacer nada (rechazar).

Se prefiere leer el ítem desde el `Container` (fuente de verdad) y no desde `ItemData` del
widget, por consistencia con `UpdateRuneRemovalLocks`.

### ⚠️ Decisión de diseño pendiente (cliente)
Soltar una runa sobre un slot de runa ocupado: ¿**rechazar** (fix propuesto) o permitir
**reemplazo/swap**? Reemplazar rompería el orden/cadena de runas y el sistema de Estados por
slot, y `TryMoveItemToContainerSlot` en el caso "destino ocupado no apilable" tiene
comportamiento sin validar (ver `05_BLUEPRINT_BP_ContainerComponent.md`). Se recomienda
rechazar mientras el cliente no indique lo contrario.

---

## Notas de arquitectura

- `UI_ItemSlot` **llama directamente** a `UpdateItemData` desde varios puntos.
  Relevante para el gate `bSlotUpdateEnabled` definido en `03_LIGHTPARADOX_PROJECT_RULES.md`.
- El widget **se auto-actualiza** vía `Construct` y `PreConstruct` — viola Rule 3.3.
  Pendiente de resolver en clase hija `UI_ItemSlot_LP`.
- El sistema de decay usa un timer propio con `SetTimerDelegate → UpdateDecayTick`.
  Este timer vive en el slot, no en `UI_HUD` — viola Rule 3.4.
- `bUseItem` actúa como gate interno del slot para controlar si se muestra el ícono real.
- **Gates de bloqueo en `UI_ItemSlot` (estado real):**
  - `IsBlocked` — gate base de drag/drop (repair/use del asset base). **Es también el que usa
    el cooldown de runas** (confirmado en los nodos reales de `UI_CraftingQueue`).
  - `bRemovalLocked` — ✅ nuevo, gate del **orden de remoción** de runas. Independiente de `IsBlocked`:
    el fin de un cooldown (`IsBlocked = False`) no puede desbloquear un slot que debe seguir
    bloqueado por orden.
  - `bIsLocked` — 🗑️ huérfana, sin uso.
- **Puntos de bloqueo de `bRemovalLocked`:** `OnDragDetected`, `OnDrop` y `OnMouseButtonDown`
  (ver secciones arriba). El overlay es puramente visual.
- El check "es runa" en `OnDrop` valida dos capas antes de ÉXITO/FALLO: `Cast To
  BP_EquipmentComponent` (descarta contenedores que no son de equipamiento) e
  `IsValidRuneDestination` (descarta slots que no son de runa). **No valida que el slot destino
  esté vacío** — ver bug abierto arriba.

---

## Deuda técnica registrada

| Problema | Regla violada | Estado |
|---|---|---|
| Auto-update en Construct y PreConstruct | Rule 3.3 | Pendiente — resolver en UI_ItemSlot_LP |
| Timer de decay vive en el slot, no en UI_HUD | Rule 3.4 | Pendiente — requiere refactor de decay |
| Ediciones en asset base, no en clase hija | Rule 4.1 | Pendiente — migrar a UI_ItemSlot_LP |
| AssignDuration hardcodeado a 5.0 en OnDrop | — | Pendiente — hacer configurable |
| Print String "No es runa" debug temporal | — | Pendiente — eliminar |
| Pin RuneIcon sin uso en AddRuneToQueue | — | Pendiente — limpiar |
| Candados fantasma al soltar runa en slot no-runa | — | ✅ Resuelto (sesión anterior) — Cast To BP_EquipmentComponent + IsValidRuneDestination |
| Hardening opcional — `IsValid(LockedSlot)` en `AddRuneToQueue` | Ver `13` | ⏳ Pendiente, no bloqueante |
| **`bIsLocked` variable huérfana** | — | ⏳ **Nuevo** — sin Get ni Set. Borrar o resolver junto con el siguiente ítem |
| **Cooldown de runas usa `IsBlocked` (gate compartido con repair/use del asset base)** | — | ⏳ **Nuevo** — frágil: si repair/use modifican `IsBlocked` en un slot de runa en cooldown podrían interferir. Evaluar migrar el cooldown a una variable propia (posible uso de la variable huérfana `bIsLocked` o una nueva) |
| `OnDrop` no tenía `NOT bIsLocked` pese a lo documentado | — | ✅ Documentación corregida esta sesión (el candado funciona vía `IsBlocked`) |
| **Acción de click derecho (mover/usar) sin exportar** | — | ⏳ **Nuevo** — verificar dónde se lee `Is Right Mouse` y si requiere su propio `NOT Removal Locked` |
| **🐞 Runa sobre slot de runa ocupado dispara cooldown/candado** | — | 🔴 **Nuevo — abierto.** Ver sección dedicada. Pendiente decisión de diseño (rechazar vs reemplazar) y aplicar fix |

---

*Archivo actualizado — sesión Light Paradox (Orden de remoción de runas)*
*Cambios: variable `bRemovalLocked`, widget `RemovalLockOverlay`, función `SetRemovalLocked`; condiciones ampliadas en `OnDragDetected`, `OnDrop` y `OnMouseButtonDown` (flujo real documentado desde captura); corrección: el `Branch` real de `OnDrop` no incluía `bIsLocked`; `bIsLocked` marcada huérfana; cooldown documentado como dependiente de `IsBlocked`; convención de nombres `bNombre` → `Nombre`; bug nuevo de runa sobre slot ocupado registrado con fix propuesto.*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
