# 14 — Widget: UI_QueueBlueprint + UI_RuneAssignQueue
### Widget base: UI_QueueBlueprint
### Widget nuevo: UI_RuneAssignQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 — Cooldown RuneAssign)

---

## UI_QueueBlueprint — Documentación base

### Contexto

`UI_QueueBlueprint` es el widget visual de cada ítem en la cola de crafteo.
Se instancia dinámicamente dentro de `UI_CraftingQueue` por cada ítem en queue.
Muestra: icono del ítem, tiempo restante, cantidad, y botón de cancelar.

### Hierarchy confirmado

```
[UI_QueueBlueprint]
  └── BlueprintSizeBox
        └── [Overlay]
              ├── Icon (Image)
              ├── [Size Box]
              │     └── BtnCancel (Button)
              │           └── [Overlay]
              │                 ├── [BtnCancelBackground] (Image)
              │                 └── BtnCancelIcon (Image)
              └── [Border]
                    └── [Horizontal Box]
                          ├── Time (Text) — ej: "10 sec"
                          └── Amount (Text) — ej: "x2"
```

### Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `Queue Blueprint` | STR_QueueBlueprint | Struct con datos del crafteo. Se setea via Update Blueprint. |
| `Size` | Vector2D | Tamaño del widget. Pasado al crear desde UI_CraftingQueue (100x100). |

### Funciones confirmadas

| Función | Notas |
|---|---|
| `Bind_Time_Text` | Binding del TextBlock Time |
| `Bind_Amount_Text` | Binding del TextBlock Amount |
| `UpdateBlueprint` | Recibe STR_QueueBlueprint y guarda en variable |

---

## UI_RuneAssignQueue — Implementación

### Contexto

Duplicado de `UI_QueueBlueprint`. No reutiliza el original porque está
acoplado a `STR_QueueBlueprint` y a `Cancel Crafting BPI`.
Vive dentro de `UI_CraftingQueue` en el panel `RuneQueueBox`.

### Decisión de arquitectura

`UI_RuneAssignQueue` vive en `UI_CraftingQueue`, no en `UI_Character`.
Razones:
- `UI_Character` tiene mucha información — agregar más lo vuelve incontrolable
- `UI_CraftingQueue` es el contenedor natural de acciones con tiempo
- Todos los cooldowns futuros vivirán en la misma caja
- `RuneQueueBox` es independiente de `CraftingQueueBox` — el rebuild
  del crafteo no afecta los widgets de runa

### Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `RuneIcon` | `Texture2D` (Object Reference) | Icono de la runa |
| `AssignDuration` | `Float` | Duración del cooldown |
| `TargetContainer` | `BP_ContainerComponent` (Object Reference) | Contenedor destino |
| `TargetSlot` | `Integer` | Slot destino |
| `SourceContainer` | `BP_ContainerComponent` (Object Reference) | Contenedor origen |
| `SourceSlot` | `Integer` | Slot origen |

### Funciones confirmadas

| Función | Estado | Notas |
|---|---|---|
| `Bind_Amount_Text` | Eliminada | No se usa en runas |
| `Bind_Time_Text` | Vaciada | Pendiente reimplementar para countdown |
| `InitRuneAssignQueue` | Implementada | Renombrada desde UpdateBlueprint |

### InitRuneAssignQueue — flujo implementado

```
Entry (RuneIcon, AssignDuration, TargetContainer, TargetSlot,
       SourceContainer, SourceSlot)
  → SET RuneIcon
  → SET AssignDuration
  → SET TargetContainer
  → SET TargetSlot
  → SET SourceContainer
  → SET SourceSlot
  → Make Brush from Texture (Texture: GET RuneIcon, 256x256)
  → Set Brush (Target: Icon widget, Brush: Return Value)
```

### Hierarchy confirmado

```
[UI_RuneAssignQueue]
  └── BlueprintSizeBox
        └── [Overlay]
              ├── Icon (Image)
              ├── [Size Box]
              │     └── BtnCancel (Button)
              └── [Border]
                    └── [Horizontal Box]
                          └── Time (Text)
```

> **Nota:** `Amount` fue eliminado del Hierarchy. Las runas no tienen cantidad.

### Bindings

- `Bind_Time_Text` removido del TextBlock `Time` en Designer
- Pendiente reimplementar para mostrar countdown

### Estado de implementación

| Componente | Estado |
|---|---|
| Duplicar y limpiar UI_QueueBlueprint | ✅ Completo |
| Variables de instancia | ✅ Completo |
| InitRuneAssignQueue | ✅ Completo |
| Bind_Time_Text countdown | ⏳ Pendiente |
| BtnCancel lógica propia | ⏳ Pendiente |
| Event Construct | ⏳ Pendiente |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Bind_Time_Text vacío | Necesita reimplementarse para mostrar countdown en segundos | Pendiente |
| BtnCancel sin lógica | Debe devolver la runa al inventario y cancelar el timer | Pendiente |

---

*Archivo actualizado — sesión Light Paradox (Lógica 1 — implementación parcial)*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
