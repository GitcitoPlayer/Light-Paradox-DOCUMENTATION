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
| `Restriction Query` | ALL( ALL( EasyRPG.Items.Equipment )) | Query global del contenedor. Todos los ítems deben tener el tag padre de Equipment para entrar. |
| `Restriction Enabled Items` | 0 Array elements | |
| `Restriction Disabled Items` | 0 Array elements | |
| `Container Slot Settings` | 8 Array elements | Ver tabla de slots confirmados abajo. |

---

## Container Slot Settings — slots confirmados

Cada entrada del array corresponde al slot del mismo índice en `Equipment Slots`.
El índice de `Container Slot Settings` debe coincidir con el índice de `Equipment Slots`.

| Index | Name | Background | RestrictionQuery |
|---|---|---|---|
| 0 | — | — | ALL( ALL( EasyRPG.Items.Equipment.Head )) |
| 1 | — | — | ALL( ALL( EasyRPG.Items.Equipment.Body )) |
| 2 | — | — | ALL( ALL( EasyRPG.Items.Equipment.Pants )) |
| 3 | — | — | ALL( ALL( EasyRPG.Items.Equipment.Hands )) |
| 4 | — | — | ALL( ALL( EasyRPG.Items.Equipment.Feet )) |
| 5 | Backpack Slot | T_UI_Equip_Backpack | ALL( ALL( EasyRPG.Items.Equipment.Backpack )) |
| 6 | Tool Slot | T_UI_Category_Tools | ALL( ALL( EasyRPG.Items.Equipment.Tool )) |
| 7 | — | — | ALL( ALL( EasyRPG.Items.Equipment.HeadRuneWord )) |

> **Nota:** Los índices 0-4 no tienen Name ni Background confirmados — no fueron expandidos durante la sesión. Los valores se infieren como vacíos o con texturas no documentadas. Requieren verificación futura.

> **Nota:** Al momento de esta sesión solo se confirmó el índice 7 (HeadRuneWord) como slot RuneWord funcional. Los índices 8-12 (BodyRuneWord, PantsRuneWord, HandsRuneWord, BackpackRuneWord, ToolRuneWord) deben seguir el mismo patrón.

---

## Relación entre sistemas para equipment slots

El sistema de validación de equipment conecta cuatro puntos que deben estar sincronizados:

```
1. E_EquipmentSlot (enum)
   → Define los tipos de slot disponibles y su índice numérico

2. BP_EquipmentComponent → EquipmentSlots (array, instancia en BP_Character_Player)
   → Define qué slots existen en este personaje y en qué orden

3. BP_EquipmentComponent → Local Equipment Types (Map en Item Is Equipment)
   → Mapea Gameplay Tags a valores del enum

4. BP_Character_Player → EquipmentContainer → Container Slot Settings (array)
   → Define la RestrictionQuery de cada slot por índice
```

Si cualquiera de estos cuatro puntos no tiene la entrada para un slot nuevo, el slot falla silenciosamente — acepta cualquier ítem sin validar.

---

## Notas de arquitectura

- `BP_Character_Player` no fue exportado como .txt en esta sesión. La documentación de esta sesión proviene de inspección directa en el editor.
- Solo se documentaron los sistemas tocados durante la sesión de debugging de equipment slots. El resto de componentes y funciones de este Blueprint no están documentados aún.
- Las funciones de replicación (`OnRep_Equipped*`) son visibles en el panel My Blueprint pero no fueron inspeccionadas en esta sesión.

---

*Archivo creado — sesión Light Paradox*
*Sistema documentado: EquipmentContainer instance configuration*
