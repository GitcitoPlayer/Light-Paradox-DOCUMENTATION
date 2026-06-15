# 14 — Widget: UI_QueueBlueprint + UI_RuneAssignQueue
### Widget base: UI_QueueBlueprint
### Widget nuevo: UI_RuneAssignQueue
### Base Asset: EasySurvivalRPGv5
### Fuente: Inspección directa — sesión Light Paradox (Lógica 1 rediseño)

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
                          ├── Time (Text)
                          └── Amount (Text)
```

### Variables confirmadas

| Variable | Tipo | Notas |
|---|---|---|
| `Queue Blueprint` | STR_QueueBlueprint | Struct con datos del crafteo |
| `Size` | Vector2D | Tamaño del widget |

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

Representa visualmente el cooldown de desbloqueo de slot de runa.
Muestra un ícono de candado y un countdown — no el ícono de la runa arrastrada.

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
| `RuneIcon` | `Texture2D` (Object Reference) | ⚠️ Sin uso tras rediseño — ícono es candado hardcodeado |
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
| `Bind_Time_Text` | Implementada ✅ | Muestra countdown en segundos enteros |
| `InitRuneAssignQueue` | Implementada ✅ | Renombrada desde UpdateBlueprint. Ícono hardcodeado a candado. |

### InitRuneAssignQueue — flujo implementado

```
Entry (RuneIcon, AssignDuration, TargetContainer, TargetSlot,
       SourceContainer, SourceSlot)
  → SET RuneIcon (sin uso — pendiente limpiar)
  → SET AssignDuration
  → SET TargetContainer
  → SET TargetSlot
  → SET SourceContainer
  → SET SourceSlot
  → Make Brush from Texture (Texture: textura candado hardcodeada via Use Selected Asset)
  → Set Brush (Target: Icon widget, Brush: Return Value)
  → Get Game Time in Seconds
  → SET StartTime ← Return Value
```

> **Nota:** El pin `Texture` de `Make Brush from Texture` tiene la textura de candado
> hardcodeada via "Use Selected Asset from Content Browser". No usa el pin `RuneIcon`.

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
              ├── Icon (Image)   ← muestra candado hardcodeado
              ├── [Size Box]
              │     └── BtnCancel (Button)   ← deshabilitado
              └── [Border]
                    └── [Horizontal Box]
                          └── Time (Text)   ← countdown
```

> **Nota:** `Amount` fue eliminado del Hierarchy. Las runas no tienen cantidad.
> `BtnCancel` existe pero está deshabilitado — el cliente confirmó que no se necesita cancelación.

### Estado de implementación

| Componente | Estado |
|---|---|
| Duplicar y limpiar UI_QueueBlueprint | ✅ Completo |
| Variables de instancia | ✅ Completo |
| InitRuneAssignQueue con candado hardcodeado | ✅ Completo |
| Bind_Time_Text countdown | ✅ Completo |
| BtnCancel | ⛔ Deshabilitado — decisión de diseño del cliente |
| Event Construct | ⏳ Pendiente evaluar si es necesario |

---

## Deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Pin RuneIcon sin uso | Variable y pin en InitRuneAssignQueue no se usan — pendiente limpiar | ⏳ Pendiente |
| BtnCancel sin lógica | Deshabilitado por decisión del cliente. Revisar si se elimina o se mantiene inactivo | ⏳ Pendiente |
| Event Construct sin implementar | Pendiente evaluar si es necesario | Pendiente |

---

*Archivo actualizado — sesión Light Paradox (Lógica 1 rediseño)*
*Cambios: Ícono cambiado a candado hardcodeado, BtnCancel documentado como deshabilitado, diseño del sistema actualizado*
*Project: Light Paradox · Base: EasySurvivalRPGv5*
