# 08 — Blueprint: UI_Character
### Widget: UI_Character
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox

---

## Contexto

`UI_Character` es el widget que muestra los atributos del personaje y los slots de equipment.
Vive dentro de `UI_HUD` como widget hijo. `UI_HUD` guarda su referencia en la variable
`CharacterInformation` (ver `10_BLUEPRINT_UI_HUD.md`).

---

## Variables relevantes confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `CharacterInformation` | UI_Character (Self) | Referencia expuesta desde UI_HUD al activar Is Variable en el widget |

---

## Widgets de slot de equipment

Los slots de equipment son widgets individuales dentro del Hierarchy de `UI_Character`.
No son un array dinámico — cada slot es un widget nombrado explícitamente.

### Convención de nomenclatura confirmada

```
Equipment_[NombreSlot]Slot
Equipment_[NombreSlot]Slot_1  (segundo slot del mismo tipo)
Equipment_[NombreSlot]Slot_2  (tercer slot del mismo tipo)
... y así sucesivamente
```

### Slots de Head Rune Word confirmados

| Widget | Slot Number |
|---|---|
| `Equipment_HeadRuneSlot` | 7 |
| `Equipment_HeadRuneSlot_1` | 14 |
| `Equipment_HeadRuneSlot_2` | 15 |
| `Equipment_HeadRuneSlot_3` | 16 |
| `Equipment_HeadRuneSlot_4` | 17 |
| `Equipment_HeadRuneSlot_5` | 18 |
| `Equipment_HeadRuneSlot_6` | 19 |
| `Equipment_HeadRuneSlot_7` | 20 |
| `Equipment_HeadRuneSlot_8` | 21 |
| `Equipment_HeadRuneSlot_9` | 22 |

> **Nota de nomenclatura:** El slot original no tiene sufijo numérico. Los duplicados comienzan en `_1` hasta `_9`. Esta convención debe mantenerse para los grupos de Body, Pants, Hands, Feet, Backpack y Tool cuando se creen.

### Cómo agregar un widget de slot nuevo

1. Abrir `UI_Character` en modo **Designer**
2. En el panel **Hierarchy**, localizar el widget del slot más similar al nuevo
3. Duplicar ese widget
4. Renombrar el duplicado siguiendo la convención: `Equipment_[NombreSlot]Slot_[N]`
5. Asignar el `Slot Number` al índice numérico correspondiente (debe coincidir con el índice en `Equipment Slots` de `BP_Character_Player`)
6. Verificar que **Is Variable** esté activado en el panel Details

---

## Widgets adicionales confirmados

| Widget | Tipo | Notas |
|---|---|---|
| `ScrollBox_EquipmentSlots` | ScrollBox | Contiene todos los slots de equipment. Default visibility: **Collapsed**. Se revela únicamente al abrir BP_Building_Altar. |

> **Nombre exacto del ScrollBox pendiente de confirmar.** El nombre usado arriba es referencial.
> Verificar nombre real en el panel Hierarchy de UI_Character.

---

## EventGraph

### Event Construct
```
Event Construct →
  Get Owning Player →
  Get Character Attributes BPI (Target: Owning Player) →
    Attributes →
  Update Character Attributes (Target: self, Attributes) →
  Category Selected (Target: self, Button: Btn Stats, Play Sound: false)
```

> **Nota:** `UI_Character` accede al `PlayerController` desde su construcción vía
> `Get Owning Player`. No guarda referencia explícita al HUD ni al Altar.

### Lógica de conexión de slots — Set Container Reference

En el EventGraph de `UI_Character` existe lógica conectada al nodo `Set Container Reference`
que asigna la referencia de contenedor a cada widget de slot individual.

Los slots `Equipment_HeadRuneSlot_1` al `Equipment_HeadRuneSlot_9` están conectados
en cadena Exec dentro de esta lógica.

Al agregar slots nuevos de otro grupo (Body, Pants, etc.):
1. En el EventGraph, localizar el nodo `Set Container Reference`
2. Agregar un nodo `Get [NombreWidget]` para cada slot nuevo
3. Conectar ese nodo al pin `Target` de `Set Container Reference`
4. Conectar en cadena Exec después del último slot existente

---

## Funciones confirmadas (panel My Blueprint)

| Función | Notas |
|---|---|
| `UpdateCharacterAttributes` | Recibe struct de atributos y actualiza display |
| `UpdateCharacterState` | No inspeccionada en esta sesión |
| `UpdateAvailableSkillPoints` | No inspeccionada en esta sesión |
| `UpdateEquipmentSlotItem` | Maneja la actualización visual de slots de equipment. Ver sección abajo. |
| `CategorySelected` | Maneja selección de categoría/tab en el widget |
| `UpdateCharacterSkills` | No inspeccionada en esta sesión |
| `UpdateCharacterSkill` | No inspeccionada en esta sesión |
| `UpdateCharacterLevel` | No inspeccionada en esta sesión |
| `ShowRuneSlots` | **Nueva — creada en sesión Light Paradox** |
| `HideRuneSlots` | **Nueva — creada en sesión Light Paradox** |

---

## UpdateEquipmentSlotItem (Función)

Contiene un nodo **Select** que mapea cada tipo de slot a su widget correspondiente.
Cada slot nuevo debe ser conectado a este Select al ser creado.

Los slots `Equipment_HeadRuneSlot_1` al `Equipment_HeadRuneSlot_9` están conectados
a los pins `HeadRuneWord_2` al `HeadRuneWord_10` del nodo Select.

### Cómo conectar un slot nuevo en esta función

1. Abrir `UI_Character` en modo **Graph**
2. Entrar a la función `UpdateEquipmentSlotItem`
3. Localizar el nodo `Select`
4. Agregar un nodo `Get [NombreWidget]` para el nuevo slot
5. Conectar ese nodo al pin correspondiente del `Select`
6. Compilar y guardar

> **Nota:** El nodo `Select` tiene un pin por cada tipo de slot de equipment.
> Al agregar un valor nuevo a `E_EquipmentSlot`, Unreal agrega automáticamente el pin correspondiente al Select.
> Si el pin no aparece, verificar que el enum esté compilado y guardado.

---

## ShowRuneSlots (Función nueva)
**Propósito:** Revelar el ScrollBox de equipment slots al abrir BP_Building_Altar.
**Llamada desde:** BP_Building_Altar → On Opened (CraftingComponent) → cadena Cast → aquí.

```
Entry →
  Set Visibility (Target: ScrollBox_EquipmentSlots, Visibility: Visible)
```

---

## HideRuneSlots (Función nueva)
**Propósito:** Colapsar el ScrollBox de equipment slots al cerrar BP_Building_Altar.
**Llamada desde:** BP_Building_Altar → On Closed (CraftingComponent) → cadena Cast → aquí.

```
Entry →
  Set Visibility (Target: ScrollBox_EquipmentSlots, Visibility: Collapsed)
```

> **Regla aplicada:** Visibility: Collapsed (no Hidden) para cumplir Rule 1.5 de
> `03_LIGHTPARADOX_PROJECT_RULES.md` — no enviar datos a slots no visibles.

---

## Flujo completo para agregar un slot nuevo en UI_Character

```
1. Modo Designer
   → Duplicar widget de slot existente en el Hierarchy
   → Renombrar siguiendo convención: Equipment_[NombreSlot]Slot_[N]
   → Asignar Slot Number correcto
   → Verificar Is Variable activado

2. Modo Graph → EventGraph
   → Localizar Set Container Reference
   → Agregar Get [NombreWidget]
   → Conectar al pin Target
   → Conectar en cadena Exec

3. Modo Graph → UpdateEquipmentSlotItem
   → Localizar nodo Select
   → Agregar Get [NombreWidget]
   → Conectar al pin [NombreSlot] del Select

4. Compilar y guardar
```

---

## Notas de arquitectura

- `UI_Character` no tiene referencia al `BP_Building_Altar` ni al `CraftingComponent`.
  El flujo de visibilidad es unidireccional: el Altar llama al HUD, el HUD llega al widget.
- `ShowRuneSlots` y `HideRuneSlots` son las únicas funciones públicas que controlan
  la visibilidad del ScrollBox. No existe otra ruta para mostrar u ocultar esa área.
- El ScrollBox está `Collapsed` por defecto. Nunca se muestra en el arranque del juego
  a menos que el jugador interactúe con `BP_Building_Altar`.
- Los widgets de slot son referencias individuales, no un array. Cada slot nuevo requiere
  conexión manual en `Set Container Reference` y en `UpdateEquipmentSlotItem`.
- Los nodos de `Set Container Reference` del EventGraph no están colapsados actualmente.
  Pendiente evaluar organización con Collapse to Function o Comment Box por grupo cuando
  se completen los grupos de Body, Pants, Hands, Feet, Backpack y Tool.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Nombre exacto de ScrollBox_EquipmentSlots sin confirmar | Verificar en panel Hierarchy | Pendiente |
| Flujo completo de Set Container Reference no documentado | Solo se documentó el punto de conexión del nuevo slot | Pendiente |
| Nodos de Set Container Reference sin colapsar | Evaluar organización por grupo al completar todos los grupos de runas | Pendiente |

---

*Archivo actualizado — sesión Light Paradox (HeadRuneWord slots múltiples)*
*Cambios: tabla de slots HeadRuneWord confirmada (Equipment_HeadRuneSlot al _9, Slot Numbers 7 y 14-22), convención de nomenclatura para slots múltiples del mismo tipo, nota sobre UpdateEquipmentSlotItem y Set Container Reference actualizadas, deuda técnica de organización de nodos registrada*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
