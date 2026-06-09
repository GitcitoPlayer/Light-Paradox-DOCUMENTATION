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
| `StartTime` | `Float` | Tiempo de inicio del cooldown. Usado por Bind_Time_Text para calcular tiempo restante. |

### Funciones confirmadas

| Función | Estado | Notas |
|---|---|---|
| `Bind_Amount_Text` | Eliminada | No se usa en runas |
| `Bind_Time_Text` | Implementada | Muestra countdown en segundos enteros |
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
  → Get Game Time in Seconds
  → SET StartTime ← Return Value
```

### Bind_Time_Text — flujo implementado

```
Entry →
  Get Game Time in Seconds → Return Value
  GET StartTime
  - (resta) → Elapsed = GameTime - StartTime
  GET AssignDuration
  - (resta) → Remaining = AssignDuration - Elapsed
  MAX (Remaining, 0.0) → tiempo restante sin negativos
  To Text (Float) → Maximum Fractional Digits = 0
  Return Node ← Return Value de To Text
```

> **Nota:** El binding está reconectado en el Designer al TextBlock `Time`.

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

### Estado de implementación

| Componente | Estado |
|---|---|
| Duplicar y limpiar UI_QueueBlueprint | ✅ Completo |
| Variables de instancia | ✅ Completo |
| InitRuneAssignQueue | ✅ Completo |
| Bind_Time_Text countdown | ✅ Completo |
| BtnCancel lógica propia | ⏳ Pendiente — decisión de diseño del cliente pendiente |
| Event Construct | ⏳ Pendiente |

---

## Decisión de diseño pendiente — BtnCancel

El cliente confirmó que el diseño del sistema de runas cambiará y la
cancelación del queue ya no será necesaria en su forma actual.
La lógica de BtnCancel se revisará en una sesión futura cuando el
nuevo diseño esté definido.

**Contexto relevante para la próxima sesión:**
El asset base (ESRPGv5) remueve el ítem del inventario al iniciar
el crafteo queue, y lo devuelve al cancelar. Para runas, la decisión
de diseño tomada es la Opción A — mismo comportamiento. La implementación
queda pendiente hasta que el cliente defina el nuevo diseño del sistema.

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| BtnCancel sin lógica | Decisión de diseño del cliente pendiente | ⏳ Pendiente — próxima sesión |
| Event Construct sin implementar | Pendiente evaluar si es necesario | Pendiente |
| Runa permanece en inventario durante cooldown | Debe removerse al iniciar — Opción A acordada | ⏳ Pendiente — próxima sesión |

---

*Archivo actualizado — sesión Light Paradox (Lógica 1 completada parcialmente)*
*Cambios: Bind_Time_Text implementado, StartTime agregado, BtnCancel pendiente por decisión de diseño*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
