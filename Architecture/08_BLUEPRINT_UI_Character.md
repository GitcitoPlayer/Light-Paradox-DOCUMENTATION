# 08 — Blueprint: UI_Character
### Widget: UI_Character
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (RuneAltar ScrollBox visibility)

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

> **Nota:** Las variables internas de `UI_Character` no fueron inspeccionadas en esta sesión.
> Solo se documentaron los sistemas tocados durante el trabajo de RuneAltar ScrollBox visibility.

---

## Widgets confirmados

| Widget | Tipo | Notas |
|---|---|---|
| `ScrollBox_EquipmentSlots` | ScrollBox | Contiene todos los slots de equipment. Creado por el usuario en esta sesión. Default visibility: **Collapsed**. Se revela únicamente al abrir BP_Building_Altar. |

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

---

## Funciones confirmadas (panel My Blueprint)

| Función | Notas |
|---|---|
| `UpdateCharacterAttributes` | Recibe struct de atributos y actualiza display |
| `UpdateCharacterState` | No inspeccionada en esta sesión |
| `UpdateAvailableSkillPoints` | No inspeccionada en esta sesión |
| `UpdateEquipmentSlotItem` | Confirma que este widget maneja equipment slots |
| `CategorySelected` | Maneja selección de categoría/tab en el widget |
| `UpdateCharacterSkills` | No inspeccionada en esta sesión |
| `UpdateCharacterSkill` | No inspeccionada en esta sesión |
| `UpdateCharacterLevel` | No inspeccionada en esta sesión |
| `ShowRuneSlots` | **Nueva — creada en sesión Light Paradox** |
| `HideRuneSlots` | **Nueva — creada en sesión Light Paradox** |

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

## Notas de arquitectura

- `UI_Character` no tiene referencia al `BP_Building_Altar` ni al `CraftingComponent`.
  El flujo de visibilidad es unidireccional: el Altar llama al HUD, el HUD llega al widget.
- `ShowRuneSlots` y `HideRuneSlots` son las únicas funciones públicas que controlan
  la visibilidad del ScrollBox. No existe otra ruta para mostrar u ocultar esa área.
- El ScrollBox está `Collapsed` por defecto. Nunca se muestra en el arranque del juego
  a menos que el jugador interactúe con `BP_Building_Altar`.

---

*Archivo creado — sesión Light Paradox (RuneAltar ScrollBox visibility)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
