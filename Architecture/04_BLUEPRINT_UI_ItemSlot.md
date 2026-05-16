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

> **Nota:** La condición `bUseItem == True` es el "Alexito's Item Lock". Si es False, el slot muestra el BackgroundTexture en lugar del ícono del ítem.

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

        LocalAmount = Select (bControl):
          False → Select (bShift: False = Amount, True = Amount/2... ver nota)
          [anidado con bHasMultiple para resolver cantidad final]

        Set LocalAmount = resultado →
        PlaySound2D (Select(bSplitMode: SplitSound / MoveSound)) →
        CreateWidget (UI_DraggedItem, ItemIcon: GetItemIcon(ItemData), ItemAmount: LocalAmount) →
        CreateDragDropOperation (BP_DraggedItem, DefaultDragVisual: UI_DraggedItem,
          ItemData, FromContainer: Container, FromSlot: SlotNumber,
          FromSlotType: ItemSlotType, Amount: LocalAmount) →
        Return (Operation: BP_DraggedItem)
```

> **Nota cantidad:** La lógica de selección de cantidad usa tres Select nodes anidados:
> - `bHasMultiple` → si Amount <= 1, siempre Amount completo
> - `bControl` → si True, Amount / 2
> - `bShift` → si True, 1 (un solo ítem)
> - Sin modificadores → Amount completo

---

## OnDrop
**Tooltip:** Try to move item to slot.

```
Entry (Operation, PointerEvent) →
  DynamicCast (Operation → BP_DraggedItem)
    → Cast Failed →
        DynamicCast (Operation → BP_AbstractItem)
          → Cast Failed → Return False
          → Cast Success →
              GetOwningPlayer →
              TryToAddAbstractItemToSlot_BPI (Player,
                Item: ItemDataToTransfer(BP_AbstractItem.ItemData),
                Container, Slot: SlotNumber) →
              PlaySound2D (MoveSound) →
              Return True

    → Cast Success →
        Branch (NOT IsBlocked)
          → False → [nada / flujo termina]
          → True  →
              Get self.ItemData, Get BP_DraggedItem.ItemData
              ItemIsRepairable(self.ItemData) → bRepairable
              ItemIsRepairKit(BP_DraggedItem.ItemData) → bIsRepairKit
              bRepairable AND bIsRepairKit → bRepairAction

              bUseItem == True → bUseItemFlag  [Alexito's Lock]

              Branch (bRepairAction AND bUseItemFlag)
                → True  →
                    GetOwningPlayer →
                    TryToRepairItem_BPI (Player,
                      FromContainer: BP_DraggedItem.FromContainer,
                      FromSlot: BP_DraggedItem.FromSlot,
                      ToContainer: Container,
                      ToSlot: SlotNumber) →
                    PlaySound2D (MoveSound) →
                    Return True

                → False →
                    GetOwningPlayer →
                    TryMoveItemToContainerSlot_BPI (Player,
                      FromContainer: BP_DraggedItem.FromContainer,
                      FromSlot: BP_DraggedItem.FromSlot,
                      FromSlotType: BP_DraggedItem.FromSlotType,
                      ToContainer: Container,
                      ToSlot: SlotNumber,
                      Amount: BP_DraggedItem.Amount) →
                    Set bUseItem = (RandomInteger(100) <= bUseItemChance)  [Alexito's Chance] →
                    DelayAssignment →
                    PlaySound2D (MoveSound) →
                    Return True
```

> **Nota:** "Alexito's Lock" y "Alexito's Chance" son comentarios visibles en el export original. Son mecanismos de control heredados del asset base que limitan cuándo puede ejecutarse el `UpdateItemData` post-drop.

---

## Notas de arquitectura

- `UI_ItemSlot` **llama directamente** a `UpdateItemData` desde varios puntos (EventGraph, SetContainerReference, DelayAssignment). Esto es relevante para el gate `bSlotUpdateEnabled` definido en `03_LIGHTPARADOX_PROJECT_RULES.md`.
- El widget **se auto-actualiza** vía `Construct` y `PreConstruct`, lo cual viola **Rule 3.3** del documento de reglas del proyecto (no self-update logic).
- El sistema de decay usa un timer propio con `SetTimerDelegate → UpdateDecayTick`. Este timer **vive en el slot**, no en `UI_HUD`.
- `bUseItem` actúa como gate interno del slot para controlar si se muestra el ícono real o el fondo vacío.

---