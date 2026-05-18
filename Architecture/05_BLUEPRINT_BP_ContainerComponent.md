# 05 — Blueprint: BP_ContainerComponent
### Componente: BP_ContainerComponent
### Base Asset: EasySurvivalRPGv5
### Fuente: Exports .txt raw — sesión Light Paradox

---

## TryMoveItemToContainerSlot (Función)
*Intento de mover un objeto de un slot del contenedor a otro slot del contenedor destino.*

**Inputs de la función:**
- `FromSlot` (Integer)
- `ToContainer` (BP_ContainerComponent_C Object Reference)
- `ToSlot` (Integer)
- `Amount` (Integer)

**Outputs de la función:**
- `then` (Exec)

### Flujo de la función:

→ **Entry Point**
  → Asignación de variables locales (`LocalFromContainer` = Self, `LocalFromSlot` = FromSlot, etc.)
    → **Branch** (Comprueba si `FromContainer != ToContainer` AND `FromSlot != ToSlot`)
      → **True**
        → `LocalFromItem` = GetItem(`LocalFromSlot`)
          → `LocalToItem` = GetItem(`LocalToSlot`)
            → **Branch** (Comprueba si `ItemIsValid(LocalFromItem)` es verdadero)
              → **True**
                → **Branch** (Comprueba si `ItemsCanStack(LocalFromItem, LocalToItem)` es verdadero)
                  → **True** (Los objetos se pueden apilar)
                    → `AmountRemaining` = AddAmountToSlot(`LocalToSlot`, `Amount`)
                      → `AmountToRemove` = `Amount - AmountRemaining`
                        → RemoveAmountFromSlot(`LocalFromSlot`, `AmountToRemove`)
                          → `LocalUseMaxAmount` = (`Amount` > 0)
                          → **Return**
                  → **False** (Los objetos NO se pueden apilar)
                    → **Branch** (Comprueba si `ItemIsValid(LocalToItem)` es falso → el slot destino está vacío)
                      → **True** (Slot destino vacío)
                        → `LocalToItem` = SetItem(`LocalToSlot`, `LocalFromItem`)
                          → `AmountToMove` = Clamp(`Amount`, Min=0, Max=Cantidad)
                            → RemoveAmountFromSlot(`LocalFromSlot`, `AmountToMove`)
                              → **Return**
                      → **False** (Slot destino ocupado por otro objeto no apilable)
                        → RemoveAmountFromSlot(`LocalFromSlot`, `Amount`)
                          → **Return**
              → **False** (El objeto de origen no es válido)
                → RemoveAmountFromSlot(`LocalFromSlot`, `Amount`)
                  → **Return**

---

## CheckAndUpdateSlots (Función)
*Comprueba los slots cambiados y luego los actualiza.*

**Inputs de la función:**
- `then` (Exec)

### Flujo de la función:

→ **Entry Point**
  → **Branch** (Comprueba si `Length(ChangedSlots) > 0`)
    → **True**
      → **ForEachLoop** (Itera sobre `ChangedSlots`)
        → `LocalSlot` = `Array Element`
          → `LocalItem` = GetItem(`LocalSlot`)
            → **ForEachLoop** (Itera sobre `ActivePlayers`)
              → Llamada a interfaz `UpdateContainerSlot_BPI` en cada jugador (con Self, `LocalSlot`, `LocalItem`)
          → **Completed**
            → CalculateWeight()
              → Array_Clear(`ChangedSlots`)
    → **False**
      → SetComponentTickEnabled(false)
        → **Return**

---

## CheckContainerSlotForItem — Implementación Parent (Función)
*Fuente: BP_ContainerComponent base. Valida si un ítem puede ser asignado a un slot específico evaluando restricciones definidas en ContainerSlotSettings.*

### Variables relevantes

| Variable | Tipo | Notas |
|---|---|---|
| `ContainerSlotSettings` | Array de STR_ContainerSlotSettings | Define restricciones por slot. Instance Editable. Default: 0 elementos en la clase base — se configura por instancia. |

### STR_ContainerSlotSettings — campos confirmados

| Campo | Tipo | Notas |
|---|---|---|
| `Name` | String | Nombre display del slot |
| `Description` | String | Descripción opcional |
| `Background` | Texture2D | Textura de fondo del widget del slot |
| `RestrictionQuery` | Gameplay Tag Query | Define qué tags debe tener un ítem para ser aceptado en este slot |
| `RestrictionEnabledItems` | Array | Ítems específicos permitidos (normalmente vacío) |
| `RestrictionDisabledItems` | Array | Ítems específicos bloqueados (normalmente vacío) |

### Flujo de la función:

```
Entry (Slot: Integer, Item: STR_ItemData)
  → GET LocalItem desde contenedor
  → IS VALID INDEX (Slot)
      → False → [termina, comportamiento no validado]
  → GET ContainerSlotSettings[Slot] → LocalSlotSettings
  → Branch (LocalSlotSettings es válido)
      → False → [termina, sin restricción aplicada]
      → True  →
          Break STR_ContainerSlotSettings
            → LocalRestrictionQuery
            → LocalEnabledItems
            → LocalDisabledItems
          GET LocalRestrictionQuery
          SET LocalEnabledItems
          SET LocalDisabledItems
          → For Each Loop (LocalDisabledItems)
              → Item Handles Are Equal
                  → Get Item Handle
          → Get Item Tags (Item) → ItemTags
          → Branch
              → Does Container Match Tag Query
                  (Tag Container: ItemTags, Tag Query: LocalRestrictionQuery)
                  → Return Value
          → IS VALID INDEX
          → For Each Loop
              → Item Handles Are Equal
                  → Get Item Handle (LocalItem)
          → Branch → Return Node (Result)
```

### Notas de arquitectura

- `ContainerSlotSettings` tiene **Instance Editable** activado. El array base tiene **0 elementos** — cada instancia del componente define sus propios slots en el panel Details del Blueprint que lo contiene.
- Cuando el índice del slot no tiene entrada en `ContainerSlotSettings`, el sistema no aplica restricción. Cualquier ítem es aceptado. Este es el comportamiento observado en slots nuevos sin configurar.
- La validación por `RestrictionQuery` usa **Gameplay Tag Query** con formato `ALL( ALL( TagName ))`. Este formato debe respetarse exactamente al configurar slots nuevos.
- Esta función es el **Parent** que llama `BP_EquipmentComponent` en su override de `CheckContainerSlotForItem`.

---

*Archivo actualizado — sesión Light Paradox*
*Secciones agregadas: CheckContainerSlotForItem Parent, STR_ContainerSlotSettings*
