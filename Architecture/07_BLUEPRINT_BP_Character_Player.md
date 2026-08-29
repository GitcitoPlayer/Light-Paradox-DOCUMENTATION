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
| `Equipment Slots` | **23 Array elements (índices 0-22)** — ✅ corregido, ver Bug 4 en `20_SYSTEM_States.md` | Debe estar sincronizado con E_EquipmentSlot. Ver `06_BLUEPRINT_BP_EquipmentComponent.md`. **Historial:** este array tenía originalmente solo 13 elementos (0-12); los índices 14-22 (HeadRuneWord_2 a HeadRuneWord_10) fueron agregados en la sesión de fix del Bug 4 — antes de eso, los slots de runa duplicados no tenían entrada real en el array de datos del componente aunque `CheckContainerSlotForItem` los validara correctamente. |
| `Slots` | 7 | Número de slots del contenedor base. |
| `Decay Factor` | 1.0 | |
| `Decay Tick Time` | 0.25 | |
| `Store Resources as Items` | True | |
| `Restriction Query` | All Expressions Match → All Tags Match → EasyRPG.Items.Equipment | Query global del contenedor. Todos los ítems deben tener el tag padre de Equipment para entrar. |
| `Restriction Enabled Items` | 0 Array elements | |
| `Restriction Disabled Items` | 0 Array elements | |
| `Container Slot Settings` | 8 Array elements (valor documentado, pendiente reverificar tras fix de Bug 4 — ver nota abajo) | Ver tabla de slots confirmados abajo. |

> **⚠️ Nota pendiente de verificación:** el tamaño de `Container Slot Settings` fue documentado como 8 elementos en una sesión anterior, con nota aparte de que el índice 9 (`PantsRuneWord`) ya estaba confirmado funcional "vía captura" — lo cual es inconsistente con un array de solo 8 elementos (índices 0-7). Dado que se confirmó en la sesión de Bug 4 que `Equipment Slots` sí tenía un desfase de tamaño no detectado durante mucho tiempo, **se recomienda reverificar el tamaño real actual de `Container Slot Settings` en la próxima sesión** y actualizar esta tabla en consecuencia — no asumir que el valor de 8 sigue vigente.

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

El sistema de validación de equipment conecta **ocho** puntos que deben estar sincronizados (ampliado de seis a ocho en la sesión de fix del Bug 4 — ver puntos 7 y 8, nuevos):

```
1. DT_GameplayTags_Items (Data Table)
   → Define el Gameplay Tag en el proyecto

2. E_EquipmentSlot (enum)
   → Define los tipos de slot disponibles y su índice numérico

3. E_EquipmentType (enum)
   → Enum separado, debe contener el mismo valor con el mismo nombre

4. BP_EquipmentComponent → EquipmentSlots (array, instancia en BP_Character_Player)
   → Define qué slots existen en este personaje y en qué orden.
   → ⚠️ El array debe tener tantos elementos como el índice más alto usado en
     E_EquipmentSlot + 1 (actualmente 23, índices 0-22). Un array más corto
     no genera error de compilación ni de consola — el slot simplemente no
     tiene dónde "aterrizar" sus datos aunque las validaciones lógicas
     (CheckContainerSlotForItem) lo acepten. Ver Bug 4 en 20_SYSTEM_States.md
     para el caso real donde esto causó que los atributos de equipamiento
     no se aplicaran para los slots 14-22.

5. BP_EquipmentComponent (o BP_ItemsLibrary) → Item Is Equipment → Local Equipment Types (Map)
   → Mapea Gameplay Tags a valores del enum

6. BP_Character_Player → EquipmentContainer → Container Slot Settings (array)
   → Define la RestrictionQuery de cada slot por índice

7. ⚠️ NUEVO (agregado tras Bug 4) — BP_Character_Player → función
   EquipmentChanged → nodo Switch on E_EquipmentSlot
   → Cada valor nuevo agregado a E_EquipmentSlot necesita su pin de
     ejecución de salida en este Switch CONECTADO al flujo principal
     (normalmente hacia el mismo SET w/ Notify de mesh que usa su slot
     "familia" — por ejemplo, todos los HeadRuneWord_N deben apuntar al
     mismo Set w/ Notify (Equiped Head Mesh) que usa el slot base Head
     Rune Word). Un pin sin conectar no genera error de compilación ni de
     consola — simplemente la ejecución nunca continúa hacia el resto de
     la cadena (Update Footstep Settings → Update Equipment Attributes →
     Bloque A/B de Estados) para ese slot específico. Esta fue la segunda
     causa raíz confirmada del Bug 4.

8. ⚠️ NUEVO (agregado tras Bug 4) — BP_EquipmentComponent → override
   CheckContainerSlotForItem
   → Si el slot nuevo pertenece a una familia de runa múltiple (varios
     slots compartiendo el mismo EquipmentType/tag), requiere una entrada
     en el bloque OR de esta función. Ver Bug 2 y su fix completo en
     06_BLUEPRINT_BP_EquipmentComponent.md.
```

Si cualquiera de estos ocho puntos no tiene la entrada para un slot nuevo, el slot falla silenciosamente — acepta cualquier ítem sin validar, no aparece en la UI, o (el caso confirmado en Bug 4) queda visualmente equipado pero sin aplicar sus datos/atributos reales.

**Checklist rápida al crear un slot de equipamiento nuevo (por ejemplo, al replicar el sistema de runas múltiples a Body/Pants/Hands/Feet/Backpack/Tool según `16_SYSTEM_RuneBinding_WeaponCosmetic.md`, Fase 1):**

1. Agregar el Gameplay Tag en `DT_GameplayTags_Items`.
2. Agregar el valor nuevo al final de `E_EquipmentSlot` (nunca reordenar).
3. Agregar el mismo valor, mismo nombre, al final de `E_EquipmentType` (nunca reordenar).
4. **Expandir `EquipmentSlots`** en el panel Details del componente hasta cubrir el nuevo índice — verificar el tamaño real, no asumir que ya está sincronizado.
5. Si el tag es compartido por una familia de slots duplicados, confirmar que `Local Equipment Types` (Map en `Item Is Equipment`) ya cubre ese tag (probablemente no necesita entrada nueva si ya existe la del slot base de esa familia — ver nota en `06_BLUEPRINT_BP_EquipmentComponent.md`).
6. Agregar la entrada correspondiente en `Container Slot Settings` con su `RestrictionQuery`.
7. **Conectar el pin de ejecución del nuevo valor en `Switch on E_EquipmentSlot`** dentro de `EquipmentChanged` (`BP_Character_Player`) — verificar visualmente que no quede suelto.
8. Si es parte de una familia de runa múltiple, agregar el bloque `OR` correspondiente en `CheckContainerSlotForItem` (`BP_EquipmentComponent`).

Los pasos 9 y 10 del instructivo de creación de slots documentan además los cambios requeridos en `UI_Character` — ver **08_BLUEPRINT_UI_Character.md**.

---

## Notas de arquitectura

- `BP_Character_Player` no fue exportado como .txt en esta sesión. La documentación proviene de inspección directa en el editor.
- Solo se documentaron los sistemas tocados durante la sesión de debugging de equipment slots. El resto de componentes y funciones de este Blueprint no están documentados aún.
- Las funciones de replicación (`OnRep_Equipped*`) son visibles en el panel My Blueprint pero no fueron inspeccionadas en esta sesión.
- **Nuevo (Bug 4):** dentro de `EquipmentChanged`, tras el `Switch on E_EquipmentSlot`, todas las ramas convergen en un flujo lineal: `Update Footstep Settings → Update Equipment Attributes → [Bloque A/B de Estados, ver 20_SYSTEM_States.md]`. La cadena `Get Equipment Items → Exclude Broken Items → Get Items Equipment Attributes` (funciones puras que alimentan `Update Equipment Attributes`) fue inspeccionada y confirmada funcionalmente correcta durante el diagnóstico del Bug 4 — pendiente solo de documentación formal (ver Deuda Técnica en `20_SYSTEM_States.md`).

---

## Deuda técnica registrada

| Problema | Estado |
|---|---|
| `Equipment Slots` con tamaño insuficiente (13 de 23 elementos necesarios) | ✅ **Resuelto en sesión de fix del Bug 4** — ver `20_SYSTEM_States.md` |
| Pines de ejecución sin conectar en `Switch on E_EquipmentSlot` para HeadRuneWord_2 a HeadRuneWord_10 | ✅ **Resuelto en sesión de fix del Bug 4** |
| Tamaño real de `Container Slot Settings` sin reverificar tras el fix de Bug 4 | ⏳ Pendiente — ver nota en la tabla de variables de instancia arriba |
| Documentación de `Get Equipment Items`, `Exclude Broken Items`, `Get Items Equipment Attributes`, `Add Attribute`, `Update Equipment Attributes` | ⏳ Pendiente formalizar — ya inspeccionadas y confirmadas correctas, falta solo redactar en un .md dedicado |

---

*Archivo actualizado — sesión Light Paradox (fix Bug 4)*
*Cambios: Equipment Slots corregido de 13 a 23 elementos documentado; checklist de sincronización ampliada de 6 a 8
puntos (nuevo punto 7: conexión de pines de ejecución en Switch on E_EquipmentSlot dentro de EquipmentChanged;
nuevo punto 8: bloque OR en CheckContainerSlotForItem para familias de runa múltiple); checklist rápida de pasos
para crear un slot de equipamiento nuevo agregada; nota de verificación pendiente sobre el tamaño real de
Container Slot Settings*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
