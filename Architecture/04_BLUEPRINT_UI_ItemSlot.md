# 04 — Blueprint: UI_ItemSlot
### Widget: UI_ItemSlot
### Base Asset: EasySurvivalRPGv5
### Fuente: Exports .txt raw — sesión Light Paradox

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

### Flujo completo con intercepción de runas (modificado — sesión Lógica 1)

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
        Branch (NOT IsBlocked)
          → False → [termina]
          → True  →
              bRepairAction = ItemIsRepairable(self.ItemData) AND ItemIsRepairKit(BP_DraggedItem.ItemData)
              bUseItemFlag = (bUseItem == True)

              Branch (bRepairAction AND bUseItemFlag)
                → True  → TryToRepairItem_BPI → PlaySound2D → Return True

                → False →
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
                    → Branch (es runa)
                        → True →
                            Get Owning Player → Get HUD →
                            Cast To BP_HUD_Game →
                            GET HUD → Cast To UI_HUD →
                            GET CraftingQueue →
                            AddRuneToQueue (
                              RuneIcon: GetItemIcon(BP_DraggedItem.ItemData),
                              AssignDuration: 5.0,
                              TargetContainer: GET Container,
                              TargetSlot: GET SlotNumber,
                              SourceContainer: BP_DraggedItem.FromContainer,
                              SourceSlot: BP_DraggedItem.FromSlot)
                            → Return True

                        → False →
                            TryMoveItemToContainerSlot_BPI →
                            Set bUseItem = (RandomInteger(100) <= bUseItemChance) →
                            DelayAssignment →
                            PlaySound2D → Return True
```

> **Nota:** La intercepción de runas ocurre en el pin False del Branch
> `bRepairAction AND bUseItemFlag`. El flujo normal de TryMoveItemToContainerSlot_BPI
> se preserva intacto para ítems no runa.

> **Nota:** `AssignDuration` está hardcodeado a `5.0` como valor de prueba.
> Debe hacerse configurable en una sesión futura.

---

## Notas de arquitectura

- `UI_ItemSlot` **llama directamente** a `UpdateItemData` desde varios puntos.
  Relevante para el gate `bSlotUpdateEnabled` definido en `03_LIGHTPARADOX_PROJECT_RULES.md`.
- El widget **se auto-actualiza** vía `Construct` y `PreConstruct` — viola Rule 3.3.
  Pendiente de resolver en clase hija `UI_ItemSlot_LP`.
- El sistema de decay usa un timer propio con `SetTimerDelegate → UpdateDecayTick`.
  Este timer vive en el slot, no en `UI_HUD` — viola Rule 3.4.
- `bUseItem` actúa como gate interno del slot para controlar si se muestra el ícono real.

---

## Deuda técnica registrada

| Problema | Regla violada | Estado |
|---|---|---|
| Auto-update en Construct y PreConstruct | Rule 3.3 | Pendiente — resolver en UI_ItemSlot_LP |
| Timer de decay vive en el slot, no en UI_HUD | Rule 3.4 | Pendiente — requiere refactor de decay |
| Ediciones en asset base, no en clase hija | Rule 4.1 | Pendiente — migrar a UI_ItemSlot_LP |
| AssignDuration hardcodeado a 5.0 en OnDrop | — | Pendiente — hacer configurable |

---

*Archivo actualizado — sesión Light Paradox (Lógica 1 — intercepción de runas en OnDrop)*
*Cambios: OnDrop actualizado con flujo completo de intercepción de runas via Does Container Match Tag Query*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
