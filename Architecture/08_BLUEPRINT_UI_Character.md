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

| Widget | Slot Number | Visibility default |
|---|---|---|
| `Equipment_HeadRuneSlot` | 7 | Visible / Not Hit-Testable |
| `Equipment_HeadRuneSlot_1` | 14 | Collapsed |
| `Equipment_HeadRuneSlot_2` | 15 | Collapsed |
| `Equipment_HeadRuneSlot_3` | 16 | Collapsed |
| `Equipment_HeadRuneSlot_4` | 17 | Collapsed |
| `Equipment_HeadRuneSlot_5` | 18 | Collapsed |
| `Equipment_HeadRuneSlot_6` | 19 | Collapsed |
| `Equipment_HeadRuneSlot_7` | 20 | Collapsed |
| `Equipment_HeadRuneSlot_8` | 21 | Collapsed |
| `Equipment_HeadRuneSlot_9` | 22 | Collapsed |

> **Nota de Lógica 3:** Los slots 2-10 (`Equipment_HeadRuneSlot_1` al `_9`) fueron
> cambiados a `Collapsed` en el Designer como parte de la implementación de la
> Lógica 3 (asignación de runas en orden). Solo `Equipment_HeadRuneSlot` (Slot 7)
> arranca visible al abrir el Altar.

> **Nota de nomenclatura:** El slot original no tiene sufijo numérico. Los duplicados
> comienzan en `_1` hasta `_9`. Esta convención debe mantenerse para los grupos de
> Body, Pants, Hands, Feet, Backpack y Tool cuando se creen.

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
| `Equipment Scroll Box` | ScrollBox | Contiene los slots de cosméticos. Siempre visible — NO se oculta con ShowRuneSlots/HideRuneSlots. |
| `Head Rune Box` | VerticalBox | Contiene los 10 slots de runa de Head directamente. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |
| `Body Rune Box` | VerticalBox | Contenedor de runas de Body. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |
| `Pants Rune Box` | VerticalBox | Contenedor de runas de Pants. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |
| `Hands Rune Box` | VerticalBox | Contenedor de runas de Hands. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |
| `Feet Rune Box` | VerticalBox | Contenedor de runas de Feet. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |
| `Backpack Rune Box` | VerticalBox | Contenedor de runas de Backpack. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |
| `Tool Rune Box` | VerticalBox | Contenedor de runas de Tool. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |

> **Nota:** `Equipment Scroll Box` fue removido de `ShowRuneSlots` y `HideRuneSlots`
> a petición del cliente — los cosméticos deben estar visibles en todo momento.
> Solo los 7 Rune Box se muestran/ocultan al interactuar con el Altar.

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
| `UpdateRuneSlotVisibility` | **Nueva — creada en sesión Light Paradox (Lógica 3)** |

---

## UpdateEquipmentSlotItem (Función)

**Inputs:** `Container`, `Slot` (Integer), `Item Data`

Contiene:
- Variable local `Equipment Reference`
- Nodo `==` que compara `Container` con `Equipment Reference`
- `Branch` cuya `Condition` recibe el resultado del `==`
- Nodo `Select` que mapea cada valor de `E_EquipmentSlot` a su widget correspondiente
- `Update Item Data` al final del flujo `True`

### Modificaciones agregadas en sesión Lógica 3

Dentro del `Branch True` existente se insertó:

```
Branch True (existente) →
  Item Is Valid (Item Data)
    Is Valid     → [Select → Update Item Data] → Update Rune Slot Visibility
                     (Slot Index: Slot, Item Assigned: True)
    Is Not Valid → Update Rune Slot Visibility
                     (Slot Index: Slot, Item Assigned: False)
```

> **Nota:** `Item Is Valid` es una función pura (sin pin Exec) que devuelve Boolean.
> Se usa conectando su salida `Is Valid` al pin `Condition` de un `Branch` nuevo
> insertado entre el `Branch` existente y el `Update Item Data` original.

### Cómo conectar un slot nuevo en esta función

1. Abrir `UI_Character` en modo **Graph**
2. Entrar a la función `UpdateEquipmentSlotItem`
3. Localizar el nodo `Select`
4. Agregar un nodo `Get [NombreWidget]` para el nuevo slot
5. Conectar ese nodo al pin correspondiente del `Select`
6. Compilar y guardar

> **Nota:** El nodo `Select` tiene un pin por cada valor de `E_EquipmentSlot`.
> Al agregar un valor nuevo a `E_EquipmentSlot`, Unreal agrega automáticamente
> el pin correspondiente al Select. Si el pin no aparece, verificar que el enum
> esté compilado y guardado.

---

## UpdateRuneSlotVisibility (Función nueva — Lógica 3)

**Propósito:** Revelar o colapsar el slot de runa siguiente en la cadena cuando
un slot recibe o pierde un ítem.

**Inputs:**
- `Slot Index` (Integer) — Slot Number del slot que cambió
- `Item Assigned` (Boolean) — True si recibió ítem, False si fue vaciado

**Flujo implementado:**
```
Entry →
  Select (Index: Slot Index) → Return Value (widget del slot siguiente)
    → Is Valid →
        Is Valid exec →
          Branch (Condition: Item Assigned)
            True  → Set Visibility (Target: widget siguiente, Visibility: Visible)
            False → Set Visibility (Target: widget siguiente, Visibility: Collapsed)
        Is Not Valid → [termina — era el último slot]
```

**Implementación final — Select por enum `E_EquipmentSlot` dentro de CollapseGraph:**

El `Select` original de tipo Integer fue reemplazado por un `Select` de tipo enum
colapsado en un `CollapseGraph` para organización. El `Index` recibe `Slot Index`
y los pins del enum mapean directamente los Slot Numbers correctos:

| Pin del enum | Widget conectado | Slot Number |
|---|---|---|
| `Head Rune Word` | `Equipment_HeadRuneSlot_1` | 14 |
| `Head Rune Word 2` | `Equipment_HeadRuneSlot_2` | 15 |
| `Head Rune Word 3` | `Equipment_HeadRuneSlot_3` | 16 |
| `Head Rune Word 4` | `Equipment_HeadRuneSlot_4` | 17 |
| `Head Rune Word 5` | `Equipment_HeadRuneSlot_5` | 18 |
| `Head Rune Word 6` | `Equipment_HeadRuneSlot_6` | 19 |
| `Head Rune Word 7` | `Equipment_HeadRuneSlot_7` | 20 |
| `Head Rune Word 8` | `Equipment_HeadRuneSlot_8` | 21 |
| `Head Rune Word 9` | `Equipment_HeadRuneSlot_9` | 22 |
| Todos los demás pins | sin conectar — `Is Valid` los filtra | — |

> **Nota de diseño:** El pin `Head Rune Word` (Slot Number 7, el primer slot)
> revela al slot siguiente (`Equipment_HeadRuneSlot_1`), no a sí mismo.
> El último slot (`Head Rune Word 10`, Slot Number 22) no tiene widget destino —
> el nodo `Is Valid` absorbe el None sin error.

> **✅ Lógica 3 — COMPLETADA Y VERIFICADA EN PLAY:**
> - Asignar runa en slot N → slot N+1 se revela (`Visible`)
> - Quitar runa del slot N → slot N+1 se colapsa (`Collapsed`)
> - La cadena funciona correctamente de slot 1 a slot 10

---

## ShowRuneSlots (Función)
**Propósito:** Revelar los contenedores de runa al abrir BP_Building_Altar.
**Llamada desde:** BP_Building_Altar → On Opened (CraftingComponent) → cadena Cast → aquí.

```
Entry →
  Set Visibility (Target: Head Rune Box, Visibility: Visible)
  Set Visibility (Target: Body Rune Box, Visibility: Visible)
  Set Visibility (Target: Pants Rune Box, Visibility: Visible)
  Set Visibility (Target: Hands Rune Box, Visibility: Visible)
  Set Visibility (Target: Feet Rune Box, Visibility: Visible)
  Set Visibility (Target: Backpack Rune Box, Visibility: Visible)
  Set Visibility (Target: Tool Rune Box, Visibility: Visible)
```

> **Nota:** `Equipment Scroll Box` fue removido de esta función. Los cosméticos
> son siempre visibles.

---

## HideRuneSlots (Función)
**Propósito:** Colapsar los contenedores de runa al cerrar BP_Building_Altar.
**Llamada desde:** BP_Building_Altar → On Closed (CraftingComponent) → cadena Cast → aquí.

```
Entry →
  Set Visibility (Target: Head Rune Box, Visibility: Collapsed)
  Set Visibility (Target: Body Rune Box, Visibility: Collapsed)
  Set Visibility (Target: Pants Rune Box, Visibility: Collapsed)
  Set Visibility (Target: Hands Rune Box, Visibility: Collapsed)
  Set Visibility (Target: Feet Rune Box, Visibility: Collapsed)
  Set Visibility (Target: Backpack Rune Box, Visibility: Collapsed)
  Set Visibility (Target: Tool Rune Box, Visibility: Collapsed)
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
  la visibilidad de los Rune Box. No existe otra ruta para mostrar u ocultar esa área.
- Todos los Rune Box están `Collapsed` por defecto. Nunca se muestran en el arranque
  del juego a menos que el jugador interactúe con `BP_Building_Altar`.
- Los widgets de slot son referencias individuales, no un array. Cada slot nuevo requiere
  conexión manual en `Set Container Reference` y en `UpdateEquipmentSlotItem`.
- Los nodos de `Set Container Reference` del EventGraph no están colapsados actualmente.
  Pendiente evaluar organización con Collapse to Function o Comment Box por grupo cuando
  se completen los grupos de Body, Pants, Hands, Feet, Backpack y Tool.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Select de UpdateRuneSlotVisibility | Resuelto usando Select por enum E_EquipmentSlot dentro de CollapseGraph. Slot Numbers del enum mapean correctamente a cada widget destino. | **Resuelto** |
| Flujo completo de Set Container Reference no documentado | Solo se documentó el punto de conexión del nuevo slot | Pendiente |
| Nodos de Set Container Reference sin colapsar | Evaluar organización por grupo al completar todos los grupos de runas | Pendiente |

---

*Archivo actualizado — sesión Light Paradox (Lógica 3 — COMPLETADA)*
*Cambios: UpdateRuneSlotVisibility actualizada con implementación final por enum, Select Integer reemplazado por Select enum dentro de CollapseGraph, deuda técnica de Select marcada como resuelta, Lógica 3 verificada en Play*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
