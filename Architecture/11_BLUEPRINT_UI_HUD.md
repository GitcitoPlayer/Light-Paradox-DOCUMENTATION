# 11 — Blueprint: UI_HUD
### Widget: UI_HUD
### Base Asset: EasySurvivalRPGv5
### Fuente: Exports .txt raw + inspección directa — sesión Light Paradox
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto

`UI_HUD` es el widget principal del juego. Contiene como hijos todos los widgets
de la interfaz, incluyendo `UI_Character`. Es instanciado y agregado al viewport
por `BP_HUD_Game` al inicio del juego.

Es el Blueprint donde, según `03_LIGHTPARADOX_PROJECT_RULES.md`, debe vivir
el gate `bSlotUpdateEnabled` en la clase hija `UI_HUD_LP`.

El EventGraph implementa la interfaz `BPI_HUD_Game`. Cada evento BPI delega
inmediatamente a una función interna homónima. No hay lógica de negocio propia
en el EventGraph, salvo la excepción de `UpdateContainerSlot_BPI`.

---

## 1 — Variables relevantes confirmadas

Variables confirmadas en el EventGraph y funciones exportadas:

| Variable | Tipo | Scope | Notas |
|---|---|---|---|
| `CraftingIsOpen` | Boolean | Instancia | True cuando el panel de crafting está abierto |
| `OnlyCraftingQueue` | Boolean | Instancia | Controlado por OpenCrafting. Oculta BtnCraft si True |
| `ShowCharacterState` | Boolean | Instancia | Controlado vía SetShowCharacterState_BPI |
| `ShowMinimap` | Boolean | Instancia | Controlado vía SetShowMinimap_BPI |
| `ShowBuildingDurability` | Boolean | Instancia | Controlado vía SetShowBuildingDurability_BPI |
| `CraftingList` | UI_CraftingList (widget ref) | Instancia | Referencia al widget de lista de crafteo |
| `SelectedBlueprintInfo` | UI_SelectedBlueprintInfo (widget ref) | Instancia | Panel de info del blueprint seleccionado |
| `CharacterInformation` | UI_Character (widget ref) | Instancia | Panel de stats del personaje. Expuesta automáticamente al activar **Is Variable** en el widget dentro del Hierarchy de UI_HUD |
| `TradingList` | UI_TradeList (widget ref) | Instancia | Lista de ítems de trading |
| `CraftingQueue` | UI_CraftingQueue (widget ref, inferida) | Instancia | Widget de cola de crafteo. Accedido vía UpdateCraftingQueue_BPI y UpdateCraftingQueueBlueprint_BPI. Ver `13_WIDGET_UI_CraftingQueue.md` |

> **Nota:** Únicamente se documentan variables con uso directo observable en los exports.
> Otras variables pueden existir en funciones no exportadas en esta sesión.

---

## 2 — EventGraph: Estructura General

### 2.1 — Patrón de delegación BPI → Función interna

Todos los eventos del EventGraph siguen el mismo patrón:

```
BPI Event (e.g. OpenInventory_BPI)
  → then → OpenInventory() [función interna]

BPI Event (UpdateContainerSlot_BPI) — EXCEPCIÓN
  → then → ItemTransferToData(ItemTransfer) → STR_ItemData
         → UpdateContainerSlot(Container, Slot, ItemData)
```

| Paso | Descripción |
|---|---|
| BPI Event recibe el call | K2Node_Event |
| K2Node_Event → then pin | K2Node_CallFunction (función interna, mismo nombre sin _BPI) |
| Parámetros del BPI event | Pasan directamente a la función interna (pins conectados 1:1) |
| Excepción: UpdateContainerSlot_BPI | Inserta `ItemTransferToData` (pure function de BP_ItemsLibrary) para convertir STR_ItemTransferData → STR_ItemData antes de llegar a UpdateContainerSlot |

### 2.2 — Tick Event (Update Loop)

UI_HUD tiene **Tick activo**. El Tick event llama tres funciones en secuencia por frame:

```
EventTick
  → UpdateInteractrionText (función interna)
  → UpdateDurabilityBar (función interna)
  → UpdateGrowthBar (función interna)
```

> **Nota:** Esto contrasta con `UI_ItemSlot` que según Rule 3.4 debe tener Tick
> desactivado. `UI_HUD` es el orchestrator central y su Tick es intencional.

### 2.3 — Grupos del EventGraph

El EventGraph está organizado en secciones con comentarios de agrupación:

| Sección / Grupo | BPI Events / Triggers | Descripción |
|---|---|---|
| Update Tick | EventTick → UpdateInteractrionText → UpdateDurabilityBar → UpdateGrowthBar | Actualiza texto de interacción y barras por frame |
| Main (Init/Open/Close HUD) | InitPlayerContainers_BPI, OpenHUD_BPI, CloseHUD_BPI | Rutas BPI → funciones internas Open/Close/Init |
| Character | UpdateCharacterInfo_BPI, UpdateCharacterSkill_BPI, UpdateCharacterSkills_BPI | Pasa datos al widget UI_Character (CharacterInformation) |
| Containers | OpenInventory_BPI, OpenContainer_BPI, UpdateContainerSlot_BPI, BlockHotbarSlot_BPI, UpdateContainerResource_BPI, UpdateWeight_BPI, ResizeContainer_BPI, UpdateSelectedHotbarSlot_BPI | Toda la gestión de inventario/hotbar. UpdateContainerSlot usa ItemTransferToData antes de llamar a UpdateContainerSlot interno |
| Crafting | OpenCrafting_BPI, UpdateBlueprintList_BPI, UpdateCraftingQueueBlueprint_BPI, UpdateCraftingQueue_BPI | Ver sección OpenCrafting Function |
| Dialogue | OpenDialogue_BPI, CloseDialogue_BPI, UpdateDialogue_BPI, QuickSelectDialogueReply_BPI | Sistema de diálogos y respuestas |
| Building | OpenBuildingMenu_BPI, CloseBuildingMenu_BPI, OpenMalletMenu_BPI, CloseMalletMenu_BPI | Menús de construcción y mazo |
| Journal | OpenJournal_BPI, UpdateJournalNotes_BPI, UpdateActiveQuestNote_BPI | UpdateJournalNotes llama UpdateActiveQuests y luego UpdateJournalNotes interno. UpdateActiveQuestNote llama UpdateActiveQuest y UpdateJournalNote |
| Trading | OpenTrade_BPI, UpdateTradeList_BPI, UpdateSelectedTradingItem_BPI | Sistema de comercio |
| Maps | OpenMap_BPI, SetMapEnabled_BPI, SetMinimapEnabled_BPI | Control de mapa y minimapa |
| Settings | SetShowCharacterState_BPI, SetShowMinimap_BPI, SetShowBuildingDurability_BPI | Set directo a variables Boolean de instancia |
| Chat | OpenChat_BPI, CloseChat_BPI, AddChatMessage_BPI, UpdateChatMessages_BPI | Sistema de chat en partida |
| Other | OnItemPicked_BPI, UpdateSelectedItem_BPI, OpenInteractionSwitcher_BPI, OpenCodelock_BPI, UpdateTargetActor_BPI, PlayDamageReaction_BPI, PlayBlockReaction_BPI, PlayHitReaction_BPI, AddEventMessage_BPI, AddStatusEffect_BPI, RemoveStatusEffect_BPI, UpdateStatusEffect_BPI, OpenSubMenu_BPI, CloseSubMenu_BPI | Eventos misceláneos: ítem recogido, codelock, reacciones de combate, status effects |

---

## 3 — Función: OpenCrafting

**Tooltip:** "Open crafting." | **Category:** Crafting

### 3.1 — Inputs

| Parámetro | Tipo | Uso |
|---|---|---|
| `OnlyCraftingQueue` | Boolean | Se guarda en variable de instancia. Controla modo cola |
| `DisableManualCrafting` | Boolean | Controla visibilidad de BtnCraft en SelectedBlueprintInfo |

### 3.2 — Flujo de ejecución

| # | Nodo / Acción | Descripción |
|---|---|---|
| 1 | FunctionEntry (OnlyCraftingQueue, DisableManualCrafting) | Recibe ambos parámetros bool del caller (OpenCrafting_BPI) |
| 2 | Set OnlyCraftingQueue = input | Guarda el parámetro en variable de instancia |
| 3 | Set CraftingIsOpen = True | Marca el estado de crafting como abierto |
| 4 | SlotAsCanvasSlot(CraftingList) → SetSize(540, 900) | Redimensiona el panel CraftingList en el canvas |
| 5 | Get SelectedBlueprintInfo → Get BtnCraft → SetVisibility | Visibilidad de BtnCraft: SelfHitTestInvisible si DisableManualCrafting=False, Hidden si True |
| 6 | Select node (Index = DisableManualCrafting) | Option 0 (False) = SelfHitTestInvisible, Option 1 (True) = Hidden |

### 3.3 — Detalle: BtnCraft Visibility Logic

```
DisableManualCrafting = False → Option 0 → SelfHitTestInvisible (botón visible e interactuable)
DisableManualCrafting = True  → Option 1 → Hidden (botón invisible)
```

> **Nota:** El Select node usa el bool como index: False = 0, True = 1.
> Patrón estándar de Unreal para Select condicional con Boolean.

### 3.4 — Widget access chain

```
UI_HUD
  → GET SelectedBlueprintInfo (variable ref a UI_SelectedBlueprintInfo)
    → GET BtnCraft (variable ref a UI_Button dentro de UI_SelectedBlueprintInfo)
      → SetVisibility(InVisibility: Select result)
```

> ⚠️ `SelectedBlueprintInfo` es una referencia directa desde `UI_HUD`. Si el widget
> no está inicializado cuando `OpenCrafting` es llamado, `BtnCraft` será null y
> `SetVisibility` fallará silenciosamente.

---

## 4 — Rol en la cadena de visibilidad del RuneAltar

`UI_HUD` es el contenedor que permite llegar a `CharacterInformation` (`UI_Character`)
desde fuera del widget:

```
Cast To BP_HUD_Game
  → Get HUD (variable de BP_HUD_Game, tipo User Widget)
  → Cast To UI_HUD
  → Get Character Information   ← vive aquí
  → Show Rune Slots / Hide Rune Slots
```

El Cast To UI_HUD es necesario porque la variable `HUD` en `BP_HUD_Game` está
declarada como `User Widget Object Reference` genérico, no como `UI_HUD` tipado.

---

## 5 — Notas de Arquitectura

| Nota | Detalle |
|---|---|
| Patrón BPI → Función interna | Todos los eventos del EventGraph siguen el mismo patrón: BPI event recibe el call y lo delega inmediatamente a una función interna homónima sin _BPI. No hay lógica intermedia en el EventGraph, salvo UpdateContainerSlot. |
| UpdateContainerSlot_BPI (excepción) | Inserta `ItemTransferToData` (BP_ItemsLibrary, pure function) entre el BPI event y UpdateContainerSlot. Convierte STR_ItemTransferData a STR_ItemData antes de llegar al slot. |
| Tick en UI_HUD | UI_HUD tiene Tick activo. En cada frame llama: UpdateInteractrionText → UpdateDurabilityBar → UpdateGrowthBar. No es el mismo patrón que UI_ItemSlot (que según Rule 3.4 debe tener Tick desactivado). |
| OpenCrafting — BtnCraft visibility | La visibilidad de BtnCraft (dentro de SelectedBlueprintInfo) se controla por DisableManualCrafting usando un Select node. Lógica UI adicional fuera del patrón simple de delegación. |
| CharacterInformation widget ref | UI_HUD guarda una referencia directa a UI_Character (CharacterInformation). UpdateCharacterSkill_BPI y UpdateCharacterSkills_BPI llaman directamente a funciones de ese widget usando la referencia. |
| Deuda técnica — base asset editado directamente | Según Rule 4.1 de 03_LIGHTPARADOX_PROJECT_RULES.md, los cambios deben hacerse en clases hijo (UI_HUD_LP). UI_HUD aún no tiene clase hija documentada. Pendiente resolver. |
| bSlotUpdateEnabled — ausente en exports | El gate bSlotUpdateEnabled definido en 03_LIGHTPARADOX_PROJECT_RULES.md no aparece en estos exports. Debe vivir en la clase hija UI_HUD_LP, no en el base. Correctamente pendiente. |

---

## 6 — Relación con la arquitectura del proyecto

Según `03_LIGHTPARADOX_PROJECT_RULES.md`:

| Clase base | Clase hija Light Paradox | Estado |
|---|---|---|
| `UI_HUD` (base ESRPGv5) | `UI_HUD_LP` | **Pendiente de crear** |

El gate `bSlotUpdateEnabled` y la función `SetSlotUpdateEnabled` deben implementarse
en `UI_HUD_LP`, no en `UI_HUD` directamente.

---

## 7 — Deuda Técnica y Pendientes

| Ítem | Estado / Acción requerida |
|---|---|
| Clase hija UI_HUD_LP | No documentada aún. Debe crearse según Rule 4.1 para agregar bSlotUpdateEnabled sin modificar el base asset. |
| bSlotUpdateEnabled gate | Ausente en estos exports. Debe implementarse en UI_HUD_LP.SetSlotUpdateEnabled y UI_HUD_LP.UpdateInventorySlot según 03_LIGHTPARADOX_PROJECT_RULES.md. |
| Funciones no exportadas | UpdateInteractrionText, UpdateDurabilityBar, UpdateGrowthBar, OpenInventory, CloseInventory, etc. no fueron exportadas en esta sesión. Requieren sesión futura. |
| Edición directa en base asset | UI_HUD fue modificado directamente. Pendiente migrar a UI_HUD_LP para cumplir Rule 4.1. |

---

*Archivo actualizado — sesión Light Paradox*
*Cambios: contenido del .docx integrado — EventGraph completo con grupos BPI, patrón de delegación, Tick event, función OpenCrafting con flujo detallado, notas de arquitectura expandidas*
*Project: Light Paradox · Base: EasySurvivalRPGv5 · UE 5.4.4*
