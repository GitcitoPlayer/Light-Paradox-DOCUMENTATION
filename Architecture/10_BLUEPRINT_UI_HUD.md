# 10 — Blueprint: UI\_HUD

### Widget: UI\_HUD

### Base Asset: EasySurvivalRPGv5

### Fuente: Exports .txt raw + inspección directa — sesión Light Paradox

### Proyecto: Light Paradox · UE 5.4.4

\---

## Contexto

`UI\_HUD` es el widget principal del juego. Contiene como hijos todos los widgets
de la interfaz, incluyendo `UI\_Character`. Es instanciado y agregado al viewport
por `BP\_HUD\_Game` al inicio del juego.

Es el Blueprint donde, según `03\_LIGHTPARADOX\_PROJECT\_RULES.md`, debe vivir
el gate `bSlotUpdateEnabled` en la clase hija `UI\_HUD\_LP`.

El EventGraph implementa la interfaz `BPI\_HUD\_Game`. Cada evento BPI delega
inmediatamente a una función interna homónima. No hay lógica de negocio propia
en el EventGraph, salvo la excepción de `UpdateContainerSlot\_BPI`.

\---

## 1 — Variables relevantes confirmadas

Variables confirmadas en el EventGraph y funciones exportadas:

|Variable|Tipo|Scope|Notas|
|-|-|-|-|
|`CraftingIsOpen`|Boolean|Instancia|True cuando el panel de crafting está abierto|
|`OnlyCraftingQueue`|Boolean|Instancia|Controlado por OpenCrafting. Oculta BtnCraft si True|
|`ShowCharacterState`|Boolean|Instancia|Controlado vía SetShowCharacterState\_BPI|
|`ShowMinimap`|Boolean|Instancia|Controlado vía SetShowMinimap\_BPI|
|`ShowBuildingDurability`|Boolean|Instancia|Controlado vía SetShowBuildingDurability\_BPI|
|`CraftingList`|UI\_CraftingList (widget ref)|Instancia|Referencia al widget de lista de crafteo|
|`SelectedBlueprintInfo`|UI\_SelectedBlueprintInfo (widget ref)|Instancia|Panel de info del blueprint seleccionado|
|`CharacterInformation`|UI\_Character (widget ref)|Instancia|Panel de stats del personaje. Expuesta automáticamente al activar **Is Variable** en el widget dentro del Hierarchy de UI\_HUD|
|`TradingList`|UI\_TradeList (widget ref)|Instancia|Lista de ítems de trading|

> \*\*Nota:\*\* Únicamente se documentan variables con uso directo observable en los exports.
> Otras variables pueden existir en funciones no exportadas en esta sesión.

\---

## 2 — EventGraph: Estructura General

### 2.1 — Patrón de delegación BPI → Función interna

Todos los eventos del EventGraph siguen el mismo patrón:

```
BPI Event (e.g. OpenInventory\_BPI)
  → then → OpenInventory() \[función interna]

BPI Event (UpdateContainerSlot\_BPI) — EXCEPCIÓN
  → then → ItemTransferToData(ItemTransfer) → STR\_ItemData
         → UpdateContainerSlot(Container, Slot, ItemData)
```

|Paso|Descripción|
|-|-|
|BPI Event recibe el call|K2Node\_Event|
|K2Node\_Event → then pin|K2Node\_CallFunction (función interna, mismo nombre sin \_BPI)|
|Parámetros del BPI event|Pasan directamente a la función interna (pins conectados 1:1)|
|Excepción: UpdateContainerSlot\_BPI|Inserta `ItemTransferToData` (pure function de BP\_ItemsLibrary) para convertir STR\_ItemTransferData → STR\_ItemData antes de llegar a UpdateContainerSlot|

### 2.2 — Tick Event (Update Loop)

UI\_HUD tiene **Tick activo**. El Tick event llama tres funciones en secuencia por frame:

```
EventTick
  → UpdateInteractrionText (función interna)
  → UpdateDurabilityBar (función interna)
  → UpdateGrowthBar (función interna)
```

> \*\*Nota:\*\* Esto contrasta con `UI\_ItemSlot` que según Rule 3.4 debe tener Tick
> desactivado. `UI\_HUD` es el orchestrator central y su Tick es intencional.

### 2.3 — Grupos del EventGraph

El EventGraph está organizado en secciones con comentarios de agrupación:

|Sección / Grupo|BPI Events / Triggers|Descripción|
|-|-|-|
|Update Tick|EventTick → UpdateInteractrionText → UpdateDurabilityBar → UpdateGrowthBar|Actualiza texto de interacción y barras por frame|
|Main (Init/Open/Close HUD)|InitPlayerContainers\_BPI, OpenHUD\_BPI, CloseHUD\_BPI|Rutas BPI → funciones internas Open/Close/Init|
|Character|UpdateCharacterInfo\_BPI, UpdateCharacterSkill\_BPI, UpdateCharacterSkills\_BPI|Pasa datos al widget UI\_Character (CharacterInformation)|
|Containers|OpenInventory\_BPI, OpenContainer\_BPI, UpdateContainerSlot\_BPI, BlockHotbarSlot\_BPI, UpdateContainerResource\_BPI, UpdateWeight\_BPI, ResizeContainer\_BPI, UpdateSelectedHotbarSlot\_BPI|Toda la gestión de inventario/hotbar. UpdateContainerSlot usa ItemTransferToData antes de llamar a UpdateContainerSlot interno|
|Crafting|OpenCrafting\_BPI, UpdateBlueprintList\_BPI, UpdateCraftingQueueBlueprint\_BPI, UpdateCraftingQueue\_BPI|Ver sección OpenCrafting Function|
|Dialogue|OpenDialogue\_BPI, CloseDialogue\_BPI, UpdateDialogue\_BPI, QuickSelectDialogueReply\_BPI|Sistema de diálogos y respuestas|
|Building|OpenBuildingMenu\_BPI, CloseBuildingMenu\_BPI, OpenMalletMenu\_BPI, CloseMalletMenu\_BPI|Menús de construcción y mazo|
|Journal|OpenJournal\_BPI, UpdateJournalNotes\_BPI, UpdateActiveQuestNote\_BPI|UpdateJournalNotes llama UpdateActiveQuests y luego UpdateJournalNotes interno. UpdateActiveQuestNote llama UpdateActiveQuest y UpdateJournalNote|
|Trading|OpenTrade\_BPI, UpdateTradeList\_BPI, UpdateSelectedTradingItem\_BPI|Sistema de comercio|
|Maps|OpenMap\_BPI, SetMapEnabled\_BPI, SetMinimapEnabled\_BPI|Control de mapa y minimapa|
|Settings|SetShowCharacterState\_BPI, SetShowMinimap\_BPI, SetShowBuildingDurability\_BPI|Set directo a variables Boolean de instancia|
|Chat|OpenChat\_BPI, CloseChat\_BPI, AddChatMessage\_BPI, UpdateChatMessages\_BPI|Sistema de chat en partida|
|Other|OnItemPicked\_BPI, UpdateSelectedItem\_BPI, OpenInteractionSwitcher\_BPI, OpenCodelock\_BPI, UpdateTargetActor\_BPI, PlayDamageReaction\_BPI, PlayBlockReaction\_BPI, PlayHitReaction\_BPI, AddEventMessage\_BPI, AddStatusEffect\_BPI, RemoveStatusEffect\_BPI, UpdateStatusEffect\_BPI, OpenSubMenu\_BPI, CloseSubMenu\_BPI|Eventos misceláneos: ítem recogido, codelock, reacciones de combate, status effects|

\---

## 3 — Función: OpenCrafting

**Tooltip:** "Open crafting." | **Category:** Crafting

### 3.1 — Inputs

|Parámetro|Tipo|Uso|
|-|-|-|
|`OnlyCraftingQueue`|Boolean|Se guarda en variable de instancia. Controla modo cola|
|`DisableManualCrafting`|Boolean|Controla visibilidad de BtnCraft en SelectedBlueprintInfo|

### 3.2 — Flujo de ejecución

|#|Nodo / Acción|Descripción|
|-|-|-|
|1|FunctionEntry (OnlyCraftingQueue, DisableManualCrafting)|Recibe ambos parámetros bool del caller (OpenCrafting\_BPI)|
|2|Set OnlyCraftingQueue = input|Guarda el parámetro en variable de instancia|
|3|Set CraftingIsOpen = True|Marca el estado de crafting como abierto|
|4|SlotAsCanvasSlot(CraftingList) → SetSize(540, 900)|Redimensiona el panel CraftingList en el canvas|
|5|Get SelectedBlueprintInfo → Get BtnCraft → SetVisibility|Visibilidad de BtnCraft: SelfHitTestInvisible si DisableManualCrafting=False, Hidden si True|
|6|Select node (Index = DisableManualCrafting)|Option 0 (False) = SelfHitTestInvisible, Option 1 (True) = Hidden|

### 3.3 — Detalle: BtnCraft Visibility Logic

```
DisableManualCrafting = False → Option 0 → SelfHitTestInvisible (botón visible e interactuable)
DisableManualCrafting = True  → Option 1 → Hidden (botón invisible)
```

> \*\*Nota:\*\* El Select node usa el bool como index: False = 0, True = 1.
> Patrón estándar de Unreal para Select condicional con Boolean.

### 3.4 — Widget access chain

```
UI\_HUD
  → GET SelectedBlueprintInfo (variable ref a UI\_SelectedBlueprintInfo)
    → GET BtnCraft (variable ref a UI\_Button dentro de UI\_SelectedBlueprintInfo)
      → SetVisibility(InVisibility: Select result)
```

> ⚠️ `SelectedBlueprintInfo` es una referencia directa desde `UI\_HUD`. Si el widget
> no está inicializado cuando `OpenCrafting` es llamado, `BtnCraft` será null y
> `SetVisibility` fallará silenciosamente.

\---

## 4 — Rol en la cadena de visibilidad del RuneAltar

`UI\_HUD` es el contenedor que permite llegar a `CharacterInformation` (`UI\_Character`)
desde fuera del widget:

```
Cast To BP\_HUD\_Game
  → Get HUD (variable de BP\_HUD\_Game, tipo User Widget)
  → Cast To UI\_HUD
  → Get Character Information   ← vive aquí
  → Show Rune Slots / Hide Rune Slots
```

El Cast To UI\_HUD es necesario porque la variable `HUD` en `BP\_HUD\_Game` está
declarada como `User Widget Object Reference` genérico, no como `UI\_HUD` tipado.

\---

## 5 — Notas de Arquitectura

|Nota|Detalle|
|-|-|
|Patrón BPI → Función interna|Todos los eventos del EventGraph siguen el mismo patrón: BPI event recibe el call y lo delega inmediatamente a una función interna homónima sin \_BPI. No hay lógica intermedia en el EventGraph, salvo UpdateContainerSlot.|
|UpdateContainerSlot\_BPI (excepción)|Inserta `ItemTransferToData` (BP\_ItemsLibrary, pure function) entre el BPI event y UpdateContainerSlot. Convierte STR\_ItemTransferData a STR\_ItemData antes de llegar al slot.|
|Tick en UI\_HUD|UI\_HUD tiene Tick activo. En cada frame llama: UpdateInteractrionText → UpdateDurabilityBar → UpdateGrowthBar. No es el mismo patrón que UI\_ItemSlot (que según Rule 3.4 debe tener Tick desactivado).|
|OpenCrafting — BtnCraft visibility|La visibilidad de BtnCraft (dentro de SelectedBlueprintInfo) se controla por DisableManualCrafting usando un Select node. Lógica UI adicional fuera del patrón simple de delegación.|
|CharacterInformation widget ref|UI\_HUD guarda una referencia directa a UI\_Character (CharacterInformation). UpdateCharacterSkill\_BPI y UpdateCharacterSkills\_BPI llaman directamente a funciones de ese widget usando la referencia.|
|Deuda técnica — base asset editado directamente|Según Rule 4.1 de 03\_LIGHTPARADOX\_PROJECT\_RULES.md, los cambios deben hacerse en clases hijo (UI\_HUD\_LP). UI\_HUD aún no tiene clase hija documentada. Pendiente resolver.|
|bSlotUpdateEnabled — ausente en exports|El gate bSlotUpdateEnabled definido en 03\_LIGHTPARADOX\_PROJECT\_RULES.md no aparece en estos exports. Debe vivir en la clase hija UI\_HUD\_LP, no en el base. Correctamente pendiente.|

\---

## 6 — Relación con la arquitectura del proyecto

Según `03\_LIGHTPARADOX\_PROJECT\_RULES.md`:

|Clase base|Clase hija Light Paradox|Estado|
|-|-|-|
|`UI\_HUD` (base ESRPGv5)|`UI\_HUD\_LP`|**Pendiente de crear**|

El gate `bSlotUpdateEnabled` y la función `SetSlotUpdateEnabled` deben implementarse
en `UI\_HUD\_LP`, no en `UI\_HUD` directamente.

\---

## 7 — Deuda Técnica y Pendientes

|Ítem|Estado / Acción requerida|
|-|-|
|Clase hija UI\_HUD\_LP|No documentada aún. Debe crearse según Rule 4.1 para agregar bSlotUpdateEnabled sin modificar el base asset.|
|bSlotUpdateEnabled gate|Ausente en estos exports. Debe implementarse en UI\_HUD\_LP.SetSlotUpdateEnabled y UI\_HUD\_LP.UpdateInventorySlot según 03\_LIGHTPARADOX\_PROJECT\_RULES.md.|
|Funciones no exportadas|UpdateInteractrionText, UpdateDurabilityBar, UpdateGrowthBar, OpenInventory, CloseInventory, etc. no fueron exportadas en esta sesión. Requieren sesión futura.|
|Edición directa en base asset|UI\_HUD fue modificado directamente. Pendiente migrar a UI\_HUD\_LP para cumplir Rule 4.1.|

\---

*Archivo actualizado — sesión Light Paradox
Cambios: contenido del .docx integrado — EventGraph completo con grupos BPI, patrón de delegación, Tick event, función OpenCrafting con flujo detallado, notas de arquitectura expandidas
Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*

