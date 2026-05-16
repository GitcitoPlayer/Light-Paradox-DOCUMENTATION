# 05_BLUEPRINT_BP_ContainerComponent.md

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
                          → `LocalUseMaxAmount` = (`Amount` > 0) // Si se usó la cantidad máxima
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
      → SetComponentTickEnabled(false) // Desactiva el tick si no hay cambios
        → **Return**