# 07 — Blueprint: BP_Character_Player
### Blueprint: BP_Character_Player
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox

---

## Contexto

`BP_Character_Player` es el Blueprint del personaje controlado por el jugador.
Contiene las instancias de todos los componentes de inventario y equipamiento.
Es el lugar donde `ContainerSlotSettings` se configura por instancia para el sistema de equipment.

---

## Componentes relevantes confirmados

| Componente | Tipo | Notas |
|---|---|---|
| `EquipmentContainer` | BP_EquipmentComponent | Gestiona el equipamiento activo del personaje. Es aquí donde se configura ContainerSlotSettings. |
| `InventoryContainer` | BP_ContainerComponent | Gestiona el inventario general. No documentado en esta sesión. |
| `AttributesComponent` | — | No documentado en esta sesión. |
| `AbilitySystemComponent` | — | No documentado en esta sesión. |

---

## EquipmentContainer — configuración de instancia

`EquipmentContainer` es seleccionable desde el panel **Components** de `BP_Character_Player`.
Al seleccionarlo, el panel **Details** expone sus variables Instance Editable.

### Variables de instancia confirmadas

| Variable | Valor confirmado | Notas |
|---|---|---|
| `Equipment Slots` | 13 Array elements (índices 0-12) | Debe estar sincronizado con E_EquipmentSlot. Ver 06_BLUEPRINT_BP_EquipmentComponent.md |
| `Slots` | 7 | Número de slots del contenedor base. |
| `Decay Factor` | 1.0 | |
| `Decay Tick Time` | 0.25 | |
| `Store Resources as Items` | True | |
| `Restriction Query` | All Expressions Match → All Tags Match → EasyRPG.Items.Equipment | Query global del contenedor. Todos los ítems deben tener el tag padre de Equipment para entrar. |
| `Restriction Enabled Items` | 0 Array elements | |
| `Restriction Disabled Items` | 0 Array elements | |
| `Container Slot Settings` | 8 Array elements | Ver tabla de slots confirmados abajo. |

---

## Container Slot Settings — slots confirmados

Cada entrada del array corresponde al slot del mismo índice en `Equipment Slots`.
El índice de `Container Slot Settings` debe coincidir con el índice de `Equipment Slots`.

La `RestrictionQuery` se configura en el **Tag Query Editor** de Unreal con la siguiente estructura:

```
Root Expression: All Expressions Match
  └── Index [0]: All Tags Match
        └── Tags: EasyRPG.Items.Equipment.[NombreSlot]
```

| Index | Name | Background | RestrictionQuery (Tag) |
|---|---|---|---|
| 0 | — | — | EasyRPG.Items.Equipment.Head |
| 1 | — | — | EasyRPG.Items.Equipment.Body |
| 2 | — | — | EasyRPG.Items.Equipment.Pants |
| 3 | — | — | EasyRPG.Items.Equipment.Hands |
| 4 | — | — | EasyRPG.Items.Equipment.Feet |
| 5 | Backpack Slot | T_UI_Equip_Backpack | EasyRPG.Items.Equipment.Backpack |
| 6 | Tool Slot | T_UI_Category_Tools | EasyRPG.Items.Equipment.Tool |
| 7 | — | — | EasyRPG.Items.Equipment.HeadRuneWord |

> **Nota:** Los índices 0-4 y 7 no tienen Name ni Background confirmados — no fueron expandidos durante la sesión. Los valores se infieren como vacíos o con texturas no documentadas. Requieren verificación futura.

> **Nota:** Al momento de la última sesión el índice 9 (`PantsRuneWord`) fue confirmado como slot funcional via captura del Tag Query Editor. Los índices 8-12 deben seguir el mismo patrón.

---

## Formato del Tag Query Editor

Confirmado via inspección directa del editor (captura sesión Light Paradox):

```
Tag Query Editor: RestrictionQuery EquipmentContainer_GEN_VARIABLE

Query
  Root Expression: All Expressions Match
  Expressions: 1 Array element
    Index [0]: All Tags Match
      Tags: EasyRPG.Items.Equipment.[NombreSlot]
```

Este es el formato que Unreal muestra al abrir una entrada funcional de `Container Slot Settings`.
Al crear una entrada nueva, replicar exactamente esta estructura.

> **Corrección:** La documentación anterior usaba la notación abreviada `ALL( ALL( ... ))`.
> Esa notación era una aproximación textual — no refleja los labels reales del editor.
> El formato correcto es el descrito arriba con los labels `All Expressions Match` y `All Tags Match`.

---

## Relación entre sistemas para equipment slots

El sistema de validación de equipment conecta seis puntos que deben estar sincronizados:

```
1. DT_GameplayTags_Items (Data Table)
   → Define el Gameplay Tag en el proyecto

2. E_EquipmentSlot (enum)
   → Define los tipos de slot disponibles y su índice numérico

3. E_EquipmentType (enum)
   → Enum separado, debe contener el mismo valor con el mismo nombre

4. BP_EquipmentComponent → EquipmentSlots (array, instancia en BP_Character_Player)
   → Define qué slots existen en este personaje y en qué orden

5. BP_EquipmentComponent (o BP_ItemsLibrary) → Item Is Equipment → Local Equipment Types (Map)
   → Mapea Gameplay Tags a valores del enum

6. BP_Character_Player → EquipmentContainer → Container Slot Settings (array)
   → Define la RestrictionQuery de cada slot por índice
```

Si cualquiera de estos seis puntos no tiene la entrada para un slot nuevo, el slot falla silenciosamente — acepta cualquier ítem sin validar, o no aparece en la UI.

Los pasos 9 y 10 del instructivo de creación de slots documentan además los cambios requeridos en `UI_Character` — ver **08_BLUEPRINT_UI_Character.md**.

---

## Notas de arquitectura

- `BP_Character_Player` no fue exportado como .txt en esta sesión. La documentación proviene de inspección directa en el editor.
- Solo se documentaron los sistemas tocados durante la sesión de debugging de equipment slots. El resto de componentes y funciones de este Blueprint no están documentados aún.
- Las funciones de replicación (`OnRep_Equipped*`) son visibles en el panel My Blueprint pero no fueron inspeccionadas en esta sesión.

---

*Archivo actualizado — sesión Light Paradox*
*Cambios: formato correcto de RestrictionQuery confirmado via Tag Query Editor, DT_GameplayTags_Items y E_EquipmentType agregados al mapa de sincronización, referencia a UI_Character agregada*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
