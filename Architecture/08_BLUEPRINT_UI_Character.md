# 08 — Blueprint: UI_Character
### Widget: UI_Character
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox
### Actualizado: sesión Light Paradox — Orden de remoción de runas (UpdateRuneRemovalLocks)

---

## Contexto

`UI_Character` es el widget que muestra los atributos del personaje y los slots de equipment.
Vive dentro de `UI_HUD` como widget hijo. `UI_HUD` guarda su referencia en la variable
`CharacterInformation` (ver `11_BLUEPRINT_UI_HUD.md`).

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

| Widget | Slot Number | Visibility default | SuccessChance |
|---|---|---|---|
| `Equipment_HeadRuneSlot` | 7 | Collapsed | 100 |
| `Equipment_HeadRuneSlot_1` | 14 | Collapsed | 90 |
| `Equipment_HeadRuneSlot_2` | 15 | Collapsed | 80 |
| `Equipment_HeadRuneSlot_3` | 16 | Collapsed | 70 |
| `Equipment_HeadRuneSlot_4` | 17 | Collapsed | 60 |
| `Equipment_HeadRuneSlot_5` | 18 | Collapsed | 50 |
| `Equipment_HeadRuneSlot_6` | 19 | Collapsed | 40 |
| `Equipment_HeadRuneSlot_7` | 20 | Collapsed | 30 |
| `Equipment_HeadRuneSlot_8` | 21 | Collapsed | 20 |
| `Equipment_HeadRuneSlot_9` | 22 | Collapsed | 10 |

> **Nota:** `SuccessChance` asignado en el Designer de cada slot individualmente.
> Slot 1 = 100% — nunca falla. Confirmado por el cliente.

---

## Widgets adicionales confirmados

| Widget | Tipo | Notas |
|---|---|---|
| `Equipment Scroll Box` | ScrollBox | Contiene los slots de cosméticos. Siempre visible. |
| `Head Rune Box` | VerticalBox | Contiene los 10 slots de runa de Head. Se muestra/oculta con ShowRuneSlots/HideRuneSlots. |
| `Body Rune Box` | VerticalBox | Pendiente de implementar |
| `Pants Rune Box` | VerticalBox | Pendiente de implementar |
| `Hands Rune Box` | VerticalBox | Pendiente de implementar |
| `Feet Rune Box` | VerticalBox | Pendiente de implementar |
| `Backpack Rune Box` | VerticalBox | Pendiente de implementar |
| `Tool Rune Box` | VerticalBox | Pendiente de implementar |

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

---

## Funciones confirmadas (panel My Blueprint)

| Función | Notas |
|---|---|
| `UpdateCharacterAttributes` | Recibe struct de atributos y actualiza display |
| `UpdateCharacterState` | No inspeccionada en esta sesión |
| `UpdateAvailableSkillPoints` | No inspeccionada en esta sesión |
| `UpdateEquipmentSlotItem` | Maneja la actualización visual de slots de equipment |
| `CategorySelected` | Maneja selección de categoría/tab en el widget |
| `UpdateCharacterSkills` | No inspeccionada en esta sesión |
| `UpdateCharacterSkill` | No inspeccionada en esta sesión |
| `UpdateCharacterLevel` | No inspeccionada en esta sesión |
| `ShowRuneSlots` | Nueva — creada en sesión Light Paradox |
| `HideRuneSlots` | Nueva — creada en sesión Light Paradox |
| `UpdateRuneSlotVisibility` | Nueva — creada en sesión Light Paradox (Lógica 3) |
| `GetNextRuneSlot` | Nueva — creada en sesión Light Paradox (Lógica 1 rediseño) |
| `UpdateRuneRemovalLocks` | ✅ **Nueva — sesión Orden de remoción de runas** |

---

## GetNextRuneSlot (Función nueva — Lógica 1 rediseño)

**Propósito:** Dado el SlotNumber del slot actual, devuelve la referencia al widget
del siguiente slot en la cadena de runas. Usada en `OnDrop` de `UI_ItemSlot` para
identificar qué slot bloquear con candado en caso de éxito.

**Inputs:**
- `CurrentSlotNumber` (Integer) — SlotNumber del slot donde se asignó la runa

**Outputs:**
- `NextSlot` (UI_ItemSlot Object Reference) — widget del siguiente slot

**Flujo implementado:**
```
Entry →
  Select (Index: CurrentSlotNumber)
    7  → GET Equipment_HeadRuneSlot_1
    14 → GET Equipment_HeadRuneSlot_2
    15 → GET Equipment_HeadRuneSlot_3
    16 → GET Equipment_HeadRuneSlot_4
    17 → GET Equipment_HeadRuneSlot_5
    18 → GET Equipment_HeadRuneSlot_6
    19 → GET Equipment_HeadRuneSlot_7
    20 → GET Equipment_HeadRuneSlot_8
    21 → GET Equipment_HeadRuneSlot_9
    22 → None (último slot, no hay siguiente)
  → Return Node (NextSlot: Return Value del Select)
```

> **Nota:** El Select usa Integer como índice — mismo patrón que UpdateRuneSlotVisibility.
> Enum E_EquipmentSlot fue evaluado y descartado porque UI_ItemSlot no expone esa variable.
> Para cualquier `CurrentSlotNumber` fuera de `7, 14–22` devuelve `None` — por eso `OnDrop`
> debe validar el slot destino antes de llamarla (ver `04_BLUEPRINT_UI_ItemSlot.md`).

---

## UpdateRuneRemovalLocks (Función — ✅ nueva, orden de remoción de runas)

**Propósito:** recalcular qué slots de runa deben quedar bloqueados para que las runas solo
puedan quitarse en orden descendente (la última asignada primero). Un slot queda bloqueado
si **él y el slot siguiente tienen ítem**. El último slot de la cadena nunca se bloquea.

**Input:**
- `Container` — mismo tipo que el input `Container` de `UpdateEquipmentSlotItem`
  (se espera `BP_ContainerComponent` Object Reference; tipo exacto no re-verificado al documentar).

**Local Variable:**
- `LocalRuneSlots` — Array de `UI_ItemSlot` (Object Reference). Se usa variable local (y no el
  output directo de un nodo puro) siguiendo la lección del Bug 3 (`20_SYSTEM_States.md`).

**Flujo implementado:**
```
Function Entry (Container) →
  SET LocalRuneSlots ← Make Array (10 pines, en este orden exacto):
      [0] Equipment_HeadRuneSlot      (7)
      [1] Equipment_HeadRuneSlot_1    (14)
      [2] Equipment_HeadRuneSlot_2    (15)
      [3] Equipment_HeadRuneSlot_3    (16)
      [4] Equipment_HeadRuneSlot_4    (17)
      [5] Equipment_HeadRuneSlot_5    (18)
      [6] Equipment_HeadRuneSlot_6    (19)
      [7] Equipment_HeadRuneSlot_7    (20)
      [8] Equipment_HeadRuneSlot_8    (21)
      [9] Equipment_HeadRuneSlot_9    (22)
  → For Loop (First Index: 0, Last Index: 8)
      Loop Body →
        Slot actual   = Get (a copy) LocalRuneSlots [Index]
        Slot siguiente = Get (a copy) LocalRuneSlots [Index + 1]
        NextOcupado   = Item Is Valid ( ítem del slot siguiente )
        ActualOcupado = Item Is Valid ( ítem del slot actual )
        AND (ActualOcupado, NextOcupado)
        → SetRemovalLocked (Target: slot actual, bLocked: resultado del AND)
      Completed →
        Get (a copy) LocalRuneSlots [9] → SetRemovalLocked (bLocked: False)
```

> **Cómo se obtiene el ítem de cada slot (dos variantes propuestas en sesión):**
> 1. **Preferida:** `Get Item` (función de `BP_ContainerComponent`; Target = input `Container`,
>    Slot = `Get Slot Number` del widget) → `Item Is Valid`. Lee la fuente de verdad.
> 2. Alternativa: `Get Item Data` del propio widget → `Item Is Valid`. Depende de que
>    `UpdateEquipmentSlotItem` ya haya actualizado `ItemData` del slot.
>
> **No quedó registrado cuál de las dos usó el usuario.** Documentar la variante real la
> próxima vez que se abra la función. `ActualOcupado` evita bloquear un slot vacío si queda un
> hueco heredado de pruebas anteriores.

**Dónde se llama** (`UpdateEquipmentSlotItem`, ver siguiente sección): un solo nodo
`UpdateRuneRemovalLocks`, con su exec alimentado por el `then` de **ambos** nodos
`Update Rune Slot Visibility` (rama Item Is Valid y rama Is Not Valid). Su input `Container`
se conecta al `Container` del `Function Entry` de `UpdateEquipmentSlotItem`.

> **Estado: ✅ Funcional en juego** (confirmado por el usuario). Las pruebas puntuales de
> click derecho sobre un slot bloqueado y del escenario original del Bug 5 no fueron
> reportadas por separado — ver pendientes.

---

## UpdateEquipmentSlotItem (Función)

**Inputs:** `Container`, `Slot` (Integer), `Item Data`

### Modificaciones agregadas en sesión Lógica 3 y sesión Orden de remoción

```
Branch True (existente) →
  Item Is Valid (Item Data)
    Is Valid     → [Select → Update Item Data] → Update Rune Slot Visibility
                     (Slot Index: Slot, Item Assigned: True)  ──┐
    Is Not Valid → Update Rune Slot Visibility                   │
                     (Slot Index: Slot, Item Assigned: False) ──┤
                                                                 ▼
                                      ✅ NUEVO → UpdateRuneRemovalLocks (Container)
```

---

## UpdateRuneSlotVisibility (Función — Lógica 3)

**Propósito:** Revelar o colapsar el slot de runa siguiente cuando un slot recibe o pierde un ítem.

**Inputs:**
- `Slot Index` (Integer)
- `Item Assigned` (Boolean)

**Implementación:** Select por enum `E_EquipmentSlot` dentro de CollapseGraph.

| Pin del enum | Widget conectado | Slot Number |
|---|---|---|
| `Head` | `Equipment_HeadRuneSlot` | 7 |
| `Head Rune Word` | `Equipment_HeadRuneSlot_1` | 14 |
| `Head Rune Word 2` | `Equipment_HeadRuneSlot_2` | 15 |
| `Head Rune Word 3` | `Equipment_HeadRuneSlot_3` | 16 |
| `Head Rune Word 4` | `Equipment_HeadRuneSlot_4` | 17 |
| `Head Rune Word 5` | `Equipment_HeadRuneSlot_5` | 18 |
| `Head Rune Word 6` | `Equipment_HeadRuneSlot_6` | 19 |
| `Head Rune Word 7` | `Equipment_HeadRuneSlot_7` | 20 |
| `Head Rune Word 8` | `Equipment_HeadRuneSlot_8` | 21 |
| `Head Rune Word 9` | `Equipment_HeadRuneSlot_9` | 22 |

---

## ShowRuneSlots (Función)

```
Entry →
  Set Visibility (Head/Body/Pants/Hands/Feet/Backpack/Tool Rune Box → Visible)
```

## HideRuneSlots (Función)

```
Entry →
  Set Visibility (Head/Body/Pants/Hands/Feet/Backpack/Tool Rune Box → Collapsed)
```

---

## Decisiones de diseño confirmadas

| Decisión | Detalle |
|---|---|
| Runas se asignan en orden ascendente | Slot 1 primero, luego 2, etc. |
| Runas se quitan en orden descendente | Slot 10 primero, luego 9, etc. **Implementado con `bRemovalLocked` + `RemovalLockOverlay`** (no con `IsBlocked`/`bIsLocked`). |
| Bloqueo por orden: independiente del cooldown | `bRemovalLocked` y el candado de cooldown (`LockIcon`, vía `IsBlocked`) son sistemas separados. |
| Slots estáticos (no dinámicos) | Ya existen todos en el Hierarchy. Evaluación de dinámicos cuando el sistema esté estable y se replique para Body/Pants/etc. |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Bug — Quitar cosmético colapsa solo un slot aunque haya varios asignados | **No cubierto** por `UpdateRuneRemovalLocks`. Opción: agregar el slot del cosmético Head al inicio del array `LocalRuneSlots` si el cliente quiere impedir quitarlo con runas asignadas | **Bug abierto** |
| Bug — Quitar runas en orden incorrecto rompe cadena de visibilidad | ✅ **RESUELTO esta sesión** con `bRemovalLocked` + `UpdateRuneRemovalLocks`. Nota anterior ("Opción B via IsBlocked/bIsLocked") reemplazada | ✅ Resuelto |
| Interior de `UpdateRuneRemovalLocks`: variante de lectura de ítem no registrada | `Get Item` (Container) vs `Get Item Data` (widget) | ⏳ Documentar variante real |
| Replicar `UpdateRuneRemovalLocks` para Body/Pants/Hands/Feet/Backpack/Tool | Un `Make Array` propio por familia, con el mismo patrón | ⏳ Fase 1 de `16_SYSTEM_RuneBinding_WeaponCosmetic.md` |
| Validar escenario original del Bug 5 con el bloqueo por orden activo | Se espera que el orden forzado impida la cascada (inferencia, ver `20_SYSTEM_States.md`) | ⏳ Pendiente validación explícita |
| Nodos de Set Container Reference sin colapsar | Evaluar organización por grupo al completar todos los grupos | Pendiente |
| Grupos Body/Pants/Hands/Feet/Backpack/Tool pendientes | Replicar cuando Head esté estable | ⏳ Pendiente |
| 🐞 Runa soltada sobre slot de runa ocupado dispara cooldown | Vive en `UI_ItemSlot.OnDrop`, ver `04_BLUEPRINT_UI_ItemSlot.md` | 🔴 Abierto |

---

*Archivo actualizado — sesión Light Paradox (Orden de remoción de runas)*
*Cambios: función `UpdateRuneRemovalLocks` documentada; conexión en `UpdateEquipmentSlotItem`; bug de orden de remoción marcado resuelto (solución: `bRemovalLocked`, no `IsBlocked`/`bIsLocked`); pendientes nuevos registrados.*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
