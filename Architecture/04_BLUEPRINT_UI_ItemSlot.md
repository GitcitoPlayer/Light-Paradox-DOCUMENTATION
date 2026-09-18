# 04 — Blueprint: UI_ItemSlot
### Widget: UI_ItemSlot
### Base Asset: EasySurvivalRPGv5
### Fuente: Exports .txt raw — sesión Light Paradox
### Actualizado: sesión Light Paradox — Fix candados fantasma en Action/Crafting Queue

---

## Variables relevantes (inferidas de los exports)

| Variable | Tipo | Notas |
|---|---|---|
| `ItemData` | STR_ItemData (struct) | Dato del ítem actual del slot |
| `Container` | BP_ContainerComponent | Referencia al contenedor del slot |
| `SlotNumber` | Integer | Índice del slot |
| `ItemSlotType` | E_ItemSlotType (enum) | Tipo de slot |
| `IsBlocked` | Boolean | Si el slot está bloqueado para drag |
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
| `bIsLocked` | Boolean | True cuando el slot tiene candado activo. Rechaza drops mientras es True. |

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
                          ├── Icon         ← ícono del ítem
                          ├── [Amount]     ← cantidad del stack
                          └── LockIcon     ← candado (Collapsed por defecto, Visible cuando bIsLocked = True)
                    └── [Size Box]
                          └── DurabilityBar
                    └── [NeedRepairBox]
                          └── NeedRepairIcon
              └── Frame
```

> **Nota:** `LockIcon` es un widget Image agregado en sesión Light Paradox (Lógica 1 rediseño).
> Tiene `Is Variable = True`. Visibility default = `Collapsed`.
> Textura: placeholder asignado en Designer — pendiente arte final.

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

```
Entry (PointerEvent) →
  Get ItemData, Get bUseItem, Get IsBlocked
  Condition: (ItemIsValid(ItemData) AND NOT IsBlocked)
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

### Flujo completo con intercepción de runas y sistema de probabilidad (Lógica 1 rediseño + Fix candados fantasma)

> **✅ Actualizado — sesión Light Paradox (Fix candados fantasma en Action/Crafting Queue).**
> El check original de "es runa" solo evaluaba el tag del ítem arrastrado, sin validar si el
> **slot destino** (`self`) era realmente un slot de runa dentro del `EquipmentContainer`. Esto
> causaba que soltar una Rune Word en cualquier `UI_ItemSlot` (inventario, hotbar, u otros
> slots de equipamiento no-runa como Head/Body/Pants) disparara el flujo ÉXITO/FALLO,
> llamara a `GetNextRuneSlot` con un `SlotNumber` fuera de su `Select`, recibiera `None`, y
> generara un candado fantasma en `UI_CraftingQueue` (`Accessed None trying to read property
> LockedSlot` / `PendingLockedSlot`). Ver `13_WIDGET_UI_CraftingQueue.md` para el detalle
> completo del síntoma y los pendientes de hardening asociados.

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
        Branch (NOT IsBlocked AND NOT bIsLocked)
          → False → [termina]
          → True  →
              bRepairAction = ItemIsRepairable(self.ItemData) AND ItemIsRepairKit(BP_DraggedItem.ItemData)
              bUseItemFlag = (bUseItem == True)

              Branch (bRepairAction AND bUseItemFlag)
                → True  → TryToRepairItem_BPI → PlaySound2D → Return True

                → False →
                    ✅ NUEVO — Cast To BP_EquipmentComponent (Object: GET Container)
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
                        → ✅ NUEVO — AND con IsValidRuneDestination:
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

## Notas de arquitectura

- `UI_ItemSlot` **llama directamente** a `UpdateItemData` desde varios puntos.
  Relevante para el gate `bSlotUpdateEnabled` definido en `03_LIGHTPARADOX_PROJECT_RULES.md`.
- El widget **se auto-actualiza** vía `Construct` y `PreConstruct` — viola Rule 3.3.
  Pendiente de resolver en clase hija `UI_ItemSlot_LP`.
- El sistema de decay usa un timer propio con `SetTimerDelegate → UpdateDecayTick`.
  Este timer vive en el slot, no en `UI_HUD` — viola Rule 3.4.
- `bUseItem` actúa como gate interno del slot para controlar si se muestra el ícono real.
- `bIsLocked` es el gate del sistema de runas — independiente de `IsBlocked`.
  `IsBlocked` es para operaciones de repair/use. `bIsLocked` es para el candado de runas.
- **✅ Nuevo (Fix candados fantasma):** el check "es runa" en `OnDrop` ahora valida dos capas
  independientes antes de entrar al flujo ÉXITO/FALLO: (1) `Cast To BP_EquipmentComponent`
  sobre `Container` — descarta cualquier `UI_ItemSlot` que no pertenezca al
  `EquipmentContainer` (inventario, hotbar, mochila, etc.), y (2) `IsValidRuneDestination` —
  descarta slots del `EquipmentContainer` que no son de runa (Head, Body, Pants, Hands, Feet,
  Backpack, Tool, y los slots base de rune word de otras familias aún no implementados). Ambas
  capas son necesarias — ninguna reemplaza a la otra. Ver sección OnDrop arriba.

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
| **Candados fantasma al soltar runa en slot no-runa** | — | ✅ **RESUELTO esta sesión** — ver sección OnDrop arriba. Fix: `Cast To BP_EquipmentComponent` + `IsValidRuneDestination` combinados con `AND` antes del flujo ÉXITO/FALLO. |
| **Hardening opcional pendiente — `IsValid(LockedSlot)` en `AddRuneToQueue`** | Ver `13_WIDGET_UI_CraftingQueue.md` | ⏳ Pendiente — capa de seguridad adicional, no bloqueante. El fix de OnDrop ya corta el problema de raíz; esto es defensa extra contra futuros puntos de entrada a `AddRuneToQueue` que pudieran pasar un `LockedSlot` inválido. |
| **Contradicción a verificar — `Set IsBlocked` vs `Set bIsLocked` en `AddRuneToQueue`** | Ver `13_WIDGET_UI_CraftingQueue.md` | ⏳ Pendiente verificación en editor — el log de runtime nombra el nodo `Set IsBlocked`, pero la documentación de `AddRuneToQueue` describe `SET bIsLocked = True`. El log (comportamiento real compilado) tiene prioridad sobre el .md hasta confirmar en el grafo real cuál variable se está seteando. |

---

*Archivo actualizado — sesión Light Paradox (Fix candados fantasma en Action/Crafting Queue)*
*Cambios: documentado el fix de OnDrop con Cast To BP_EquipmentComponent + IsValidRuneDestination
combinados con AND antes del flujo ÉXITO/FALLO; nota crítica agregada sobre SlotNumber como índice
local al Container (no global); registrados dos pendientes: Hardening opcional (IsValid check en
AddRuneToQueue) y verificación de contradicción Set IsBlocked/bIsLocked — ver 13_WIDGET_UI_CraftingQueue.md*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
