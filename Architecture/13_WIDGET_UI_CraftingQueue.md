# 13 — Widget: UI_CraftingQueue
### Widget: UI_CraftingQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 — Cooldown RuneAssign)

---

## Contexto

`UI_CraftingQueue` es el widget que contiene y gestiona los slots visuales de la
cola de crafteo. Vive dentro de `UI_HUD` como widget hijo.
Contiene un panel `CraftingQueueBox` donde se instancian los widgets `UI_QueueBlueprint`
dinámicamente al actualizar la queue.

---

## Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `CraftingQueueBox` | Panel Widget | Contenedor donde se agregan los UI_QueueBlueprint instanciados |
| `QueueBlueprintWidgets` | Array de UI_QueueBlueprint | Referencias a los widgets instanciados en CraftingQueueBox |

---

## Funciones confirmadas

### UpdateCraftingQueue
**Tooltip:** "Update crafting queue."

**Inputs:** `Queue Blueprints` (Array de STR_QueueBlueprint)

**Flujo:**
```
Entry (Queue Blueprints) →
  Clear Children (Target: CraftingQueueBox) →
  CLEAR QueueBlueprintWidgets →
  For Each Loop (Array: Queue Blueprints)
    Loop Body →
      Create UI Queue Blueprint Widget
        (Class: UI_QueueBlueprint,
         Owning Player: ...,
         Queue Blueprint: Array Element,
         Size: X=100, Y=100) →
      Add Child (Target: CraftingQueueBox, Content: Return Value) →
      ADD Return Value → QueueBlueprintWidgets
    Completed → [termina]
```

> **Patrón:** Rebuild completo — no actualiza slots individuales, destruye todos
> y los recrea en cada llamada. Se llama cuando la queue cambia de tamaño.

### UpdateCraftingQueueBlueprint
**Tooltip:** "Update crafting queue blueprint by index."

**Inputs:** `Index` (Integer), `Queue Blueprint` (STR_QueueBlueprint)

**Flujo:**
```
Entry (Index, Queue Blueprint) →
  GET QueueBlueprintWidgets[Index] →
  Update Blueprint (Target: widget obtenido, Queue Blueprint: Queue Blueprint)
```

> **Propósito:** Actualizar un slot individual sin reconstruir toda la queue.
> Llamado por `BP_CraftingComponent` en cada tick para actualizar `Time Remaining`.

---

## Relación con UI_HUD

`UI_CraftingQueue` es accedido desde `UI_HUD` vía:
- `UpdateCraftingQueue_BPI` → delega a `UpdateCraftingQueue`
- `UpdateCraftingQueueBlueprint_BPI` → delega a `UpdateCraftingQueueBlueprint`

Sigue el patrón estándar BPI → función interna documentado en `11_BLUEPRINT_UI_HUD.md`.

---

## Notas de arquitectura

- El sistema de rebuild completo en `UpdateCraftingQueue` significa que los widgets
  se destruyen y recrean cada vez que un ítem entra o sale de la queue.
- `UpdateCraftingQueueBlueprint` es el canal de actualización por tick —
  más eficiente que rebuild completo para actualizar solo el tiempo restante.
- Este widget no tiene lógica de timer propio — el timer vive en `BP_CraftingComponent`.

---

*Archivo creado — sesión Light Paradox (Lógica 1 — análisis de factibilidad Cooldown)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
