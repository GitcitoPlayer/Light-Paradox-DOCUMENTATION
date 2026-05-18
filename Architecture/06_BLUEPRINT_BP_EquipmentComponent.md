# 06 — Blueprint: BP_EquipmentComponent
### Componente: BP_EquipmentComponent
### Base Asset: EasySurvivalRPGv5
### Fuente: Exports .txt raw + inspección directa — sesión Light Paradox

---

## Contexto

`BP_EquipmentComponent` hereda de `BP_ContainerComponent`.
Gestiona el equipamiento del personaje — qué ítems están equipados en qué slots corporales.
Se instancia dentro de `BP_Character_Player` como componente `EquipmentContainer`.

---

## Variables relevantes

| Variable | Tipo | Notas |
|---|---|---|
| `EquipmentSlots` | Array de E_EquipmentSlot | Define qué slots de equipamiento existen y en qué orden. El índice del array corresponde al Slot Number usado por los widgets UI_ItemSlot. |

> **Nota:** `ContainerSlotSettings` no se define aquí — se hereda de `BP_ContainerComponent` y se configura por instancia en `BP_Character_Player`.

---

## E_EquipmentSlot — valores confirmados

| Índice | Display Name |
|---|---|
| 0 | Head |
| 1 | Body |
| 2 | Pants |
| 3 | Hands |
| 4 | Feet |
| 5 | Backpack |
| 6 | Tool |
| 7 | HeadRuneWord |
| 8 | BodyRuneWord |
| 9 | PantsRuneWord |
| 10 | HandsRuneWord |
| 11 | BackpackRuneWord |
| 12 | ToolRuneWord |
| 13 | Hello |

> **Nota:** El valor `Hello` (índice 13) es un valor de prueba. No tiene slot funcional asignado. No documentado como sistema activo.

---

## Funciones relevantes

### Item Is Equipment (Función)
*Devuelve true si el ítem es de tipo equipment. También devuelve el EquipmentType correspondiente.*

**Outputs:**
- `Result` (Boolean)
- `EquipmentType` (E_EquipmentSlot / Byte)

#### Variable local clave

| Variable | Tipo | Notas |
|---|---|---|
| `Local Equipment Types` | Map (Gameplay Tag → Byte) | Mapea cada Gameplay Tag de equipment a su valor de E_EquipmentSlot correspondiente. Variable local de la función. Se configura en Default Value dentro de la función. |

#### Local Equipment Types — valores confirmados

| Key (Gameplay Tag) | Value (E_EquipmentSlot) |
|---|---|
| EasyRPG.Items.Equipment.Head | Head |
| EasyRPG.Items.Equipment.Body | Body |
| EasyRPG.Items.Equipment.Pants | Pants |
| EasyRPG.Items.Equipment.Hands | Hands |
| EasyRPG.Items.Equipment.Feet | Feet |
| EasyRPG.Items.Equipment.Backpack | Backpack |
| EasyRPG.Items.Equipment.Tool | Tool |
| EasyRPG.Items.Equipment.HeadRuneWord | HeadRuneWord |
| EasyRPG.Items.Equipment.BodyRuneWord | BodyRuneWord |

> **Nota:** Al momento de esta sesión solo se confirmaron 9 entradas. Los tags PantsRuneWord, HandsRuneWord, BackpackRuneWord, ToolRuneWord deben agregarse siguiendo el mismo patrón si sus slots están activos.

#### Flujo de la función:

```
Entry (Item: STR_ItemData)
  → Break STR_ItemData → Item Tags
  → For Each Loop sobre KEYS de Local Equipment Types
      → Array Element (Key: Gameplay Tag)
      → GET Value desde Local Equipment Types usando Array Element como Key
      → Has Tag (Tag Container: ItemTags, Tag: Array Element)
          → Return Value (Boolean)
      → Branch (Has Tag Result)
          → True  →
              Return Node
                Result = true
                EquipmentType = GET Value
          → False → [continúa loop]
  → Completed (ningún tag hizo match) →
      Return Node
        Result = false
        EquipmentType = Head  ← valor hardcodeado por defecto
```

> **Advertencia:** El Return Node del False path tiene `EquipmentType = Head` hardcodeado. Si un ítem no tiene tag de equipment reconocido, la función devuelve Head como tipo por defecto. Esto no causa bugs visibles porque `CheckContainerSlotForItem` evalúa `Result = false` primero.

---

### CheckContainerSlotForItem — Override (Función)
*Override del Parent (BP_ContainerComponent). Agrega validación de tag de equipment sobre la validación base.*

*Tooltip original: "Returns true if item equipment slot type is equal to slot."*

#### Flujo del override:

```
Entry (Slot: Integer, Item: STR_ItemData)
  → Parent: Check Container Slot for Item
      (Slot, Item) → Result (Boolean) → [al AND]
  → Item Is Equipment (Item)
      → Result (Boolean) → [al AND]
      → EquipmentType (Byte) → [al ==]
  → == (EquipmentType, Slot)
      → Result (Boolean) → [al AND]
  → AND (3 pins)
      → Condition → Branch
  → Branch
      → True  → Return Node (Result = true)
      → False → Return Node (Result = false)
```

#### Notas de arquitectura

- El nodo `==` compara `EquipmentType` (Byte del enum) contra `Slot` (Integer del índice). Esto funciona porque los índices del array `EquipmentSlots` coinciden numéricamente con los valores Byte del enum `E_EquipmentSlot`.
- El sistema depende de que el **orden del enum** y el **orden del array** estén sincronizados. Si se reordena el enum, la validación se rompe.
- El Parent evalúa `ContainerSlotSettings[Slot].RestrictionQuery`. Si el slot no tiene entrada en `ContainerSlotSettings`, el Parent no aplica restricción y puede devolver `True`, permitiendo que cualquier ítem pase la validación aunque `Item Is Equipment` y el `==` fallen.

---

## Notas de arquitectura general

- `BP_EquipmentComponent` **no define** `ContainerSlotSettings` internamente. Esa configuración vive en la instancia del componente dentro de `BP_Character_Player`.
- Agregar un slot nuevo requiere modificaciones en **tres lugares distintos** en este componente: `E_EquipmentSlot` (enum), `EquipmentSlots` array, y `Local Equipment Types` Map. Ver **07_BLUEPRINT_BP_Character_Player.md** para la cuarta modificación requerida en la instancia.
- Ediciones realizadas directamente en el asset base de ESRPGv5 — no en clase hijo. Deuda técnica pendiente de resolver según Rule 4.1 de `03_LIGHTPARADOX_PROJECT_RULES.md`.

---

*Archivo creado — sesión Light Paradox*
*Sistema documentado: Equipment slot validation pipeline*
