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
| `EquipmentSlots` | Array de E_EquipmentSlot | Define qué slots de equipamiento existen y en qué orden. El índice del array corresponde al Slot Number usado por los widgets UI_ItemSlot. Instance Editable — configurable desde el panel Details del componente directamente en BP_EquipmentComponent, y también desde su instancia en BP_Character_Player. |

> **Nota:** `ContainerSlotSettings` no se define aquí — se hereda de `BP_ContainerComponent` y se configura por instancia en `BP_Character_Player`.

---

## Enums relacionados

### E_EquipmentSlot
Define los tipos de slot de equipamiento y su índice numérico.
El orden de los valores determina el índice — **nunca reordenar valores existentes**.

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
| 11 | FeetRuneWord |
| 12 | BackpackRuneWord |
| 13 | ToolRuneWord |
| 14 | HeadRuneWord_2 |
| 15 | HeadRuneWord_3 |
| 16 | HeadRuneWord_4 |
| 17 | HeadRuneWord_5 |
| 18 | HeadRuneWord_6 |
| 19 | HeadRuneWord_7 |
| 20 | HeadRuneWord_8 |
| 21 | HeadRuneWord_9 |
| 22 | HeadRuneWord_10 |

> **Nota:** El valor `Hello` que existía en índice 13 fue renombrado a `HeadRuneWord_2` durante la sesión de creación de slots de runas múltiples.

### E_EquipmentType
Enum separado de `E_EquipmentSlot`. Confirmado como asset independiente a partir de inspección directa.
Contiene los mismos Display Names que `E_EquipmentSlot` — se mantiene sincronizado manualmente al agregar slots nuevos.
El orden de los valores determina el índice — **nunca reordenar valores existentes**.

**Confirmado esta sesión (Bug 2 — segundo slot de runa):** `Head Rune Word` tiene índice **7** en `E_EquipmentType`, idéntico a su índice en `E_EquipmentSlot`. Ambos enums están sincronizados en este valor.

> **Nota:** `E_EquipmentType` contiene los mismos valores que `E_EquipmentSlot` incluyendo `HeadRuneWord_2` al `HeadRuneWord_10` agregados en la misma sesión.

---

## Regla crítica — orden de enumeradores

Al agregar un valor nuevo a `E_EquipmentSlot` o `E_EquipmentType`:
- **Siempre agregar al final** usando Add Enumerator
- **Nunca reordenar** valores existentes
- Reordenar rompe la correspondencia índice ↔ slot en todos los sistemas que dependen del orden numérico

---

## Funciones relevantes

### Item Is Equipment (Función)
*Devuelve true si el ítem es de tipo equipment. También devuelve el EquipmentType correspondiente.*

> **Nota:** Esta función puede vivir en `BP_EquipmentComponent` o en `BP_ItemsLibrary` como función compartida.
> La ubicación exacta debe verificarse en el editor — buscar la función en ambos Blueprints.
> Hasta confirmar, se documenta en este archivo por ser donde fue hallada originalmente.

**Outputs:**
- `Result` (Boolean)
- `EquipmentType` — **CORRECCIÓN (sesión Bug 2, Light Paradox):** el tipo real de este pin es **`E_EquipmentType` (Enum Reference)**, confirmado por tooltip directo en el editor. La documentación anterior lo listaba como `E_EquipmentSlot / Byte` — esa anotación era una inferencia no confirmada y quedó **corregida**. Al conectar este pin a un nodo `Equal`, Unreal debe resolver automáticamente el nodo especializado con dropdown de Display Names (no un `Equal (Byte)` genérico). Si el dropdown no aparece, usar `Refresh Nodes` sobre el nodo `Equal`.

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
| EasyRPG.Items.Equipment.PantsRuneWord | PantsRuneWord |

> **Nota:** Los slots `HeadRuneWord_2` al `HeadRuneWord_10` no requieren entradas nuevas en este Map. Todos comparten el tag `EasyRPG.Items.Equipment.HeadRuneWord` y la entrada existente los cubre — esto significa que `Item Is Equipment` siempre devuelve `EquipmentType = HeadRuneWord (7)` para cualquier Rune Word de tipo Head, sin importar el slot físico de destino (7, 14, 15... 22). Ver **Rule crítica del Bug 2** más abajo — esta es la causa raíz del bug de slots duplicados.

> **Nota pendiente:** Los tags HandsRuneWord, BackpackRuneWord, ToolRuneWord deben agregarse siguiendo el mismo patrón si sus slots están activos.

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

> **Advertencia:** El Return Node del False path tiene `EquipmentType = Head` hardcodeado.
> Si un ítem no tiene tag de equipment reconocido, la función devuelve Head como tipo por defecto.
> Esto no causa bugs visibles porque `CheckContainerSlotForItem` evalúa `Result = false` primero.

---

### CheckContainerSlotForItem — Override (Función)
*Override del Parent (BP_ContainerComponent). Agrega validación de tag de equipment sobre la validación base.*

*Tooltip original: "Returns true if item equipment slot type is equal to slot."*

#### Flujo del override — ✅ ACTUALIZADO tras fix Bug 2 (sesión Light Paradox)

```
Entry (Slot: Integer, Item: STR_ItemData)
  → Parent: Check Container Slot for Item
      (Slot, Item) → Result (Boolean) → [al AND, pin 1]

  → Item Is Equipment (Item)
      → Result (Boolean) → [al AND, pin 2]
      → EquipmentType (Enum E_EquipmentType) →

          [Resultado A] Equal (Enum) — EquipmentType == Slot (comparación directa, lógica original)

          [Resultado B]
            Equal (Enum) — EquipmentType == Head Rune Word (índice 7)  → B1
            Slot >= 14 (Integer >= Integer)                            → B2
            Slot <= 22 (Integer <= Integer)                            → B3
            AND (B1, B2, B3) → Resultado B

          OR (Resultado A, Resultado B) → [al AND, pin 3]

  → AND (3 pins: Parent Result, Item Is Equipment Result, OR Result)
      → Condition → Branch
  → Branch
      → True  → Return Node (Result = true)
      → False → Return Node (Result = false)
```

**Fórmula lógica final:**
```
Result = ParentResult
     AND ItemIsEquipmentResult
     AND ( (EquipmentType == Slot)
           OR (EquipmentType == HeadRuneWord AND Slot >= 14 AND Slot <= 22) )
```

#### Notas de arquitectura

- El nodo `Equal` compara `EquipmentType` (Enum `E_EquipmentType`) contra `Slot` (Integer del índice). Para la comparación directa esto funciona porque los índices del array `EquipmentSlots` coinciden numéricamente con los valores del enum `E_EquipmentType`.
- **Causa raíz del Bug 2 (resuelto esta sesión):** el sistema fue diseñado originalmente asumiendo relación 1:1 entre `EquipmentType` y `Slot`. El sistema de runas múltiples (`HeadRuneWord_2` al `HeadRuneWord_10`, slots 14-22) rompió esa relación porque todos esos slots comparten el mismo tag de Gameplay (`EasyRPG.Items.Equipment.HeadRuneWord`) y por lo tanto `Item Is Equipment` siempre devuelve `EquipmentType = HeadRuneWord (7)` sin importar a cuál de los 9 slots duplicados vaya el ítem. La comparación directa `EquipmentType == Slot` solo era verdadera para el slot 7 — para 14 al 22 siempre fallaba, bloqueando la validación de `CheckContainerSlotForItem` y en consecuencia impidiendo que el ítem quedara registrado correctamente en el `EquipmentContainer`, lo cual cortaba en cascada la aplicación de atributos/Estados vía `EquipmentChanged`.
- **Fix aplicado:** se agregó una condición `OR` adicional que acepta el caso especial de `EquipmentType == HeadRuneWord` combinado con `Slot` dentro del rango 14–22 (los 9 slots duplicados de Head Rune Word). La validación original para el resto de slots (Head, Body, Pants, Hands, Feet, Backpack, Tool, y HeadRuneWord slot 7 base) permanece intacta.
- El Parent evalúa `ContainerSlotSettings[Slot].RestrictionQuery`. Si el slot no tiene entrada en `ContainerSlotSettings`, el Parent no aplica restricción y puede devolver `True`, permitiendo que cualquier ítem pase la validación aunque `Item Is Equipment` y el chequeo de tipo fallen.

#### ⚠️ Patrón a replicar para otras familias de runa (Body/Pants/Hands/Feet/Backpack/Tool)

Cuando se repliquen los slots de runa múltiple para otras piezas de equipamiento (ver `16_SYSTEM_RuneBinding_WeaponCosmetic.md`, Fase 1), este mismo override necesitará **un bloque `OR` adicional por cada familia**, cada uno con su propio par de literales `Equal (Enum)` + rango `>=`/`<=` de índices, todos alimentando al mismo `OR` acumulado (o anidando `OR`s en cadena).

**Advertencia de escalabilidad:** si el número de familias de runa crece más allá de 2-3, el árbol de nodos `OR`/`AND`/`Equal` hardcodeados se vuelve difícil de mantener visualmente. En ese punto — **no antes** — vale la pena evaluar migrar esta validación a una tabla de rangos por `EquipmentType` (por ejemplo un Map o Data Table: `EquipmentType → [SlotMin, SlotMax]`) consultada dinámicamente en vez de literales por nodo. Esto es una reestructura mayor y **no está justificada todavía** — se documenta aquí únicamente como referencia a futuro, siguiendo la prioridad del proyecto de preferir el mínimo cambio necesario.

---

## Notas de arquitectura general

- `BP_EquipmentComponent` **no define** `ContainerSlotSettings` internamente. Esa configuración vive en la instancia del componente dentro de `BP_Character_Player`.
- Agregar un slot nuevo requiere modificaciones en **cuatro lugares** dentro de este Blueprint y sus assets relacionados: `E_EquipmentSlot` (enum), `E_EquipmentType` (enum), `EquipmentSlots` array (instancia en BP_EquipmentComponent), y `Local Equipment Types` Map. Ver **07_BLUEPRINT_BP_Character_Player.md** para las modificaciones adicionales requeridas en la instancia y en UI.
- **Nuevo (Bug 2):** además de esos cuatro lugares, si el slot nuevo pertenece a una familia de runa múltiple (varios slots compartiendo el mismo tag/EquipmentType), también requiere una entrada en el bloque `OR` de `CheckContainerSlotForItem` — ver sección de arriba.
- Ediciones realizadas directamente en el asset base de ESRPGv5 — no en clase hijo. Deuda técnica pendiente de resolver según Rule 4.1 de `03_LIGHTPARADOX_PROJECT_RULES.md`.

---

## Deuda técnica registrada

| Problema | Regla violada | Estado |
|---|---|---|
| Ediciones en asset base, no en clase hija | Rule 4.1 | Pendiente |
| Ubicación exacta de Item Is Equipment sin confirmar (BP_EquipmentComponent vs BP_ItemsLibrary) | — | Pendiente verificación |
| Bug 2 — CheckContainerSlotForItem fallaba para slots de runa duplicados (14-22) | — | ✅ **RESUELTO esta sesión** — ver sección "CheckContainerSlotForItem — Override" arriba |
| Patrón OR hardcodeado no escala bien más allá de 2-3 familias de runa | — | ⏳ Registrado como riesgo futuro, sin acción requerida todavía |

---

*Archivo actualizado — sesión Light Paradox (Bug 2 — RESUELTO)*
*Cambios: corrección del tipo real de EquipmentType (E_EquipmentType Enum, no Byte plano como se documentaba antes), fix completo de CheckContainerSlotForItem documentado con fórmula lógica final, nota de escalabilidad para futuras familias de runa*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
